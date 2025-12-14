Week 6-7: Role-Based Access Control (RBAC)
Goal
Implement document-level access control based on user groups from Active Directory.

Deliverables
1. RBAC Enforcement at Vector DB Layer
Code Sample: RBAC-Filtered Retrieval

# File: rag/rbac_retriever.py

from typing import List, Dict
from pymilvus import Collection
from dataclasses import dataclass

@dataclass
class UserContext:
    user_id: str
    email: str
    groups: List[str]  # From AD: ["Senior Advisors", "PrivateWealth", "Compliance"]
    trace_id: str

class RBACRetriever:
    """
    Retrieves documents from Milvus Vector DB
    while enforcing RBAC based on user's AD groups
    """
    
    def __init__(self, milvus_collection: Collection):
        self.collection = milvus_collection
        
    async def retrieve_with_rbac(self,
                                 query: str,
                                 user_context: UserContext,
                                 top_k: int = 5) -> List[Dict]:
        """
        Retrieve documents matching query, filtered by user's access level
        
        Document metadata schema:
        {
            "id": "doc-123",
            "content": "...",
            "classification": "Confidential",  # Public, Internal, Confidential, Restricted
            "required_groups": ["PrivateWealth", "Senior"],  # Empty = public
            "source_url": "https://...",
            "last_updated": "2024-12-01"
        }
        """
        
        # 1. Embed query using same model as docs
        from sentence_transformers import SentenceTransformer
        encoder = SentenceTransformer('bge-m3')
        query_embedding = encoder.encode(query).tolist()
        
        # 2. Build RBAC filter expression
        # Milvus filter: "required_groups EMPTY OR required_groups IN user_groups"
        filter_expr = self._build_filter_expression(user_context.groups)
        
        # 3. Search in Milvus with filter
        search_results = self.collection.search(
            data=[query_embedding],
            anns_field="embedding",
            param={
                "metric_type": "COSINE",
                "top_k": top_k
            },
            filter=filter_expr,
            output_fields=["id", "content", "source_url", "classification", "required_groups"]
        )
        
        # 4. Format results
        documents = []
        for hit in search_results[0]:
            doc = hit.entity
            documents.append({
                "id": doc.id,
                "content": doc.content,
                "source_url": doc.source_url,
                "classification": doc.classification,
                "relevance_score": hit.score,
                "accessible": True
            })
        
        # 5. Log to Phoenix for audit
        await self._log_retrieval(
            query=query,
            user_context=user_context,
            returned_count=len(documents),
            returned_docs=[d['id'] for d in documents]
        )
        
        return documents
    
    def _build_filter_expression(self, user_groups: List[str]) -> str:
        """
        Build Milvus filter expression for RBAC
        
        Logic:
        - Public docs: required_groups is empty
        - Internal docs: user has at least one matching group
        - Restricted docs: user listed explicitly
        """
        groups_list = "', '".join(user_groups)
        
        # "json_contains(required_groups, user_groups) OR required_groups is empty"
        filter_expr = (
            f"json_overlaps(required_groups, ['{groups_list}']) "
            f"OR json_size(required_groups) == 0"
        )
        
        return filter_expr
    
    async def _log_retrieval(self, query: str, user_context: UserContext,
                            returned_count: int, returned_docs: List[str]):
        """Log retrieval event to Phoenix for audit trail"""
        import requests
        
        log_payload = {
            "trace_id": user_context.trace_id,
            "event_type": "document_retrieval",
            "user_id": user_context.user_id,
            "user_groups": user_context.groups,
            "query": query,
            "returned_count": returned_count,
            "returned_doc_ids": returned_docs,
            "timestamp": datetime.utcnow().isoformat()
        }
        
        # Send to Phoenix
        requests.post(
            "https://phoenix.example.bank.com/api/events",
            json=log_payload
        )

# Usage in FastAPI endpoint
from fastapi import Depends

async def get_user_context(request: Request) -> UserContext:
    """Dependency to extract user context from JWT"""
    token = request.headers.get("Authorization").split(" ")[1]
    payload = jwt.decode(token, JWT_SECRET)
    session = sessions_db.find_one({"_id": ObjectId(payload["session_id"])})
    
    return UserContext(
        user_id=session["user_id"],
        email=session["user_context"]["email"],
        groups=session["user_context"]["groups"],
        trace_id=request.state.trace_id
    )

@app.post("/api/retrieve")
async def retrieve_documents(query: str, 
                            user_context: UserContext = Depends(get_user_context)):
    retriever = RBACRetriever(milvus_collection)
    documents = await retriever.retrieve_with_rbac(
        query=query,
        user_context=user_context
    )
    return {"documents": documents}
2. System Prompt with RBAC Context
Dynamically include only accessible sources in LLM system prompt.

Code Sample: Dynamic System Prompt

# File: prompts/system_prompt_builder.py

class SystemPromptBuilder:
    """
    Builds system prompt dynamically based on user's accessible documents
    """
    
    def __init__(self, max_docs_in_context: int = 10):
        self.max_docs_in_context = max_docs_in_context
    
    def build_prompt(self, user_context: UserContext, 
                     accessible_documents: List[Dict]) -> str:
        """
        Build system prompt that:
        1. Defines Aegis role
        2. Lists accessible source documents
        3. Sets safety guardrails
        4. Specifies output format
        """
        
        docs_context = self._format_documents(accessible_documents)
        
        system_prompt = f"""
You are Aegis, a regulation-grade AI banking advisor.

## Your Role
Provide accurate, compliant financial advice to advisors by retrieving 
information from the institutional knowledge base.

## User Context
- Name: {user_context.name}
- Email: {user_context.email}
- Clearance Level: {', '.join(user_context.groups)}

## Accessible Knowledge Base
You ONLY have access to these documents:

{docs_context}

## Critical Safety Rules

1. **Always Cite Sources**: Every statement must reference a document above.
   Format: [Source: Document Name] 
   
2. **Know Your Limits**: If information is not in your knowledge base, say:
   "I don't have information about this in my knowledge base. 
    Please consult [specialist/manager/resource]."
   
3. **Never Hallucinate**: Do not generate information beyond the documents.
   Example BAD: "The 2025 tax code changes to..." (if not in documents)
   Example GOOD: "According to IRS Publication 559, the gift tax exclusion..."
   
4. **Groundedness Check**: Your responses must be grounded in the documents.
   Target: >90% of claims should directly cite source material.
   
5. **Flag Violations**: If asked about restricted topics, respond:
   "I cannot discuss this topic. Please escalate to [compliance/manager]."
   
6. **Output Format**: Use clear markdown formatting:
   - Use headers for organization
   - Use bullet points for lists
   - Always end with a "Sources" section

## Response Template

[Your answer here]

### Sources
- [Source 1: Document Name](url)
- [Source 2: Document Name](url)

## Compliance Mode
You operate under these regulations:
- SR 11-7: Model Risk Management compliance required
- GDPR: Do not expose personal data
- CCPA: Respect privacy rights
- OWASP LLM Top 10: Security controls active

---
"""
        
        return system_prompt
    
    def _format_documents(self, documents: List[Dict]) -> str:
        """Format accessible documents for inclusion in prompt"""
        if not documents:
            return "[No documents accessible to this user]"
        
        formatted = []
        for i, doc in enumerate(documents[:self.max_docs_in_context], 1):
            formatted.append(
                f"{i}. **{doc.get('title', 'Untitled')}** "
                f"(Classification: {doc.get('classification', 'Unknown')})\n"
                f"   - URL: {doc.get('source_url', 'N/A')}\n"
                f"   - Last Updated: {doc.get('last_updated', 'Unknown')}"
            )
        
        return "\n".join(formatted)

# Usage in LangGraph
builder = SystemPromptBuilder()
system_prompt = builder.build_prompt(user_context, retrieved_documents)

# Pass to LLM
response = await llm.ainvoke(
    prompt=user_query,
    system=system_prompt
)
Exit Criteria for Week 6-7:

✅ Milvus RBAC filter expressions working
✅ Junior advisor cannot see "Private Wealth" documents
✅ Senior advisor can see all accessible docs
✅ Compliance officer can audit all retrievals (Phoenix logs)
✅ System prompt dynamically updated per user
✅ RBAC audit trail: 100% of retrievals logged
