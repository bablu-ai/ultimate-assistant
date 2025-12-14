# Phase 1: Alpha Deployment Plan
## Project Aegis - Weeks 1-11

**Phase Document Version**: 1.0  
**Duration**: 11 weeks  
**Target Release**: End of Week 11  
**Beta Users**: 10 internal testers (Shadow Mode)  
**Status**: Ready for Implementation  

---

## Phase 1 Overview

### Objectives
- Deploy functional Aegis system to 10 beta testers
- Validate 98%+ accuracy on golden set
- Achieve 100% OWASP LLM Top 10 security compliance
- Obtain compliance officer sign-off
- Zero PII leakage incidents

### Investment
- **Team**: 6 FTE engineers (Tech Lead, 2 Backend, 2 Frontend, 1 DevOps)
- **Duration**: 11 weeks (March 10 - May 25, 2025)
- **Budget**: ~$180,000 (salaries + infrastructure + LLM API)

### Success Criteria (Go/No-Go Gates)
- ✅ Golden Set: 98%+ pass rate
- ✅ Security: 100% OWASP attack blocks
- ✅ PII: 0 leakage incidents
- ✅ Compliance: Officer sign-off
- ✅ Quality: Groundedness >0.90
- ✅ Performance: P95 latency <3 seconds

---

## Week 1-2: Observability Foundation

### Goal
Establish comprehensive monitoring infrastructure so every action is visible and traceable.

### Deliverables

#### 1. Arize Phoenix Setup
Deploy AI-specific observability platform for tracing LLM calls, evals, and embeddings.

**Code Sample: Phoenix Connection & Initialization**
```python
# File: observability/phoenix_setup.py

import phoenix as px
from phoenix.trace import spans
from phoenix.evals.llm_judges import HallucinationEvaluator, QAEvaluator
import json

class PhoenixObserver:
    def __init__(self, api_key: str, endpoint: str):
        """Initialize Phoenix connection"""
        self.client = px.Client(
            endpoint=endpoint,
            api_key=api_key
        )
        self.tracer = px.tracer.get_tracer(__name__)
        
    def setup_evaluators(self):
        """Register AI-specific evaluators"""
        
        # 1. Hallucination Detection
        hallucination_eval = HallucinationEvaluator(
            model_name="gpt-4o"
        )
        px.register_evaluator(
            name="hallucination_check",
            evaluator=hallucination_eval
        )
        
        # 2. Groundedness (Is answer supported by context?)
        groundedness_eval = QAEvaluator(
            model_name="gpt-4o",
            template="""
            Context: {context}
            Response: {response}
            
            Is the response fully supported by the context?
            Score: 0.0 (not supported) to 1.0 (fully supported)
            Return JSON: {{"groundedness": float, "explanation": str}}
            """
        )
        px.register_evaluator(
            name="groundedness",
            evaluator=groundedness_eval
        )
        
        # 3. QA Relevance (Does answer match question?)
        qa_relevance_eval = QAEvaluator(
            model_name="gpt-4o",
            template="""
            Question: {question}
            Response: {response}
            
            Does the response directly answer the question?
            Score: 0.0 (no relevance) to 1.0 (perfectly relevant)
            Return JSON: {{"relevance": float, "explanation": str}}
            """
        )
        px.register_evaluator(
            name="qa_relevance",
            evaluator=qa_relevance_eval
        )

# Usage in main application
observer = PhoenixObserver(
    api_key=os.getenv("PHOENIX_API_KEY"),
    endpoint="https://phoenix.example.bank.com"
)
observer.setup_evaluators()
```

#### 2. Grafana Infrastructure Monitoring
Deploy Grafana dashboards for infrastructure metrics.

**Code Sample: Grafana Dashboard Configuration**
```yaml
# File: monitoring/grafana-dashboard.yaml

apiVersion: v1
kind: ConfigMap
metadata:
  name: grafana-aegis-dashboard
  namespace: monitoring
data:
  aegis-dashboard.json: |
    {
      "dashboard": {
        "title": "Project Aegis - Infrastructure Health",
        "panels": [
          {
            "title": "Request Rate (req/sec)",
            "targets": [
              {
                "expr": "rate(http_requests_total[1m])",
                "legendFormat": "{{handler}}"
              }
            ]
          },
          {
            "title": "Error Rate (%)",
            "targets": [
              {
                "expr": "rate(http_requests_total{status=~'5..'}[1m]) / rate(http_requests_total[1m]) * 100"
              }
            ],
            "alert": {
              "message": "Error rate >0.5% - Page on-call"
            }
          },
          {
            "title": "Response Latency (P95)",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))"
              }
            ]
          }
        ]
      }
    }
```

#### 3. W3C Trace Context Setup
Implement distributed tracing with trace IDs propagated across all services.

**Code Sample: Trace ID Propagation**
```python
# File: observability/trace_context.py

import uuid
from fastapi import Request
from starlette.middleware.base import BaseHTTPMiddleware
from datetime import datetime
import json

class TraceIDMiddleware(BaseHTTPMiddleware):
    """Middleware to generate and propagate W3C trace context"""
    
    async def dispatch(self, request: Request, call_next):
        # Extract or generate trace ID
        trace_parent = request.headers.get("traceparent")
        
        if trace_parent:
            parts = trace_parent.split("-")
            trace_id = parts[1]
        else:
            trace_id = str(uuid.uuid4()).replace("-", "")[:32]
        
        current_span_id = format(uuid.uuid4().int, "032x")[:16]
        trace_parent_header = f"00-{trace_id}-{current_span_id}-01"
        
        request.state.trace_id = trace_id
        request.state.span_id = current_span_id
        
        response = await call_next(request)
        response.headers["traceparent"] = trace_parent_header
        response.headers["X-Trace-ID"] = trace_id
        
        return response
```

**Exit Criteria for Week 1-2**:
- ✅ Phoenix dashboards showing sample traces
- ✅ Grafana showing infrastructure metrics
- ✅ Alerts configured for latency, error rate, memory
- ✅ Trace IDs visible in logs (W3C format)
- ✅ On-call rotation established

---

## Week 2-3: Authentication & Session Management

### Goal
Implement OAuth2 PKCE authentication with secure session storage.

### Deliverables

#### 1. OAuth2 PKCE Implementation

**Code Sample: OAuth2 PKCE Flow**
```python
# File: auth/oauth2_handler.py

import os
import secrets
import base64
import httpx
import json
from datetime import datetime, timedelta
from fastapi import FastAPI, HTTPException, Request
from pymongo import MongoClient
import jwt

class OAuth2PKCEHandler:
    """Implements OAuth2 PKCE flow for secure authentication"""
    
    def __init__(self, client_id: str, client_secret: str, 
                 auth_server: str, redirect_uri: str, mongo_client: MongoClient):
        self.client_id = client_id
        self.client_secret = client_secret
        self.auth_server = auth_server
        self.redirect_uri = redirect_uri
        self.db = mongo_client.aegis.sessions
        
    def generate_pkce_pair(self) -> dict:
        """Generate code_challenge and code_verifier"""
        import hashlib
        code_verifier = base64.urlsafe_b64encode(
            secrets.token_bytes(96)
        ).decode('utf-8').rstrip('=')
        
        code_challenge = base64.urlsafe_b64encode(
            hashlib.sha256(code_verifier.encode('utf-8')).digest()
        ).decode('utf-8').rstrip('=')
        
        return {
            "code_verifier": code_verifier,
            "code_challenge": code_challenge
        }
    
    def create_auth_url(self, state: str = None) -> dict:
        """Generate OAuth2 authorization URL"""
        pkce = self.generate_pkce_pair()
        state = state or secrets.token_urlsafe(32)
        
        auth_url = (
            f"{self.auth_server}/oauth2/v1/authorize"
            f"?client_id={self.client_id}"
            f"&response_type=code"
            f"&scope=openid profile email"
            f"&redirect_uri={self.redirect_uri}"
            f"&state={state}"
            f"&code_challenge={pkce['code_challenge']}"
            f"&code_challenge_method=S256"
        )
        
        return {
            "auth_url": auth_url,
            "state": state,
            "code_verifier": pkce["code_verifier"]
        }
    
    async def exchange_code_for_token(self, code: str, code_verifier: str) -> dict:
        """Exchange authorization code for access token"""
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.auth_server}/oauth2/v1/token",
                data={
                    "grant_type": "authorization_code",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                    "code": code,
                    "code_verifier": code_verifier,
                    "redirect_uri": self.redirect_uri
                }
            )
            
            if response.status_code != 200:
                raise HTTPException(status_code=400, detail="Failed to exchange code")
            
            return response.json()
    
    async def create_session(self, user_id: str, access_token: str,
                            refresh_token: str, user_context: dict) -> str:
        """Create MongoDB session document and return JWT"""
        session_doc = {
            "user_id": user_id,
            "access_token": access_token,
            "refresh_token": refresh_token,
            "user_context": user_context,
            "created_at": datetime.utcnow(),
            "expires_at": datetime.utcnow() + timedelta(hours=8),
            "last_activity": datetime.utcnow()
        }
        
        result = self.db.insert_one(session_doc)
        
        jwt_payload = {
            "session_id": str(result.inserted_id),
            "user_id": user_id,
            "exp": datetime.utcnow() + timedelta(hours=8)
        }
        
        jwt_token = jwt.encode(
            jwt_payload,
            os.getenv("JWT_SECRET"),
            algorithm="HS256"
        )
        
        return jwt_token

# FastAPI integration
oauth2 = OAuth2PKCEHandler(
    client_id=os.getenv("OAUTH_CLIENT_ID"),
    client_secret=os.getenv("OAUTH_CLIENT_SECRET"),
    auth_server=os.getenv("OAUTH_SERVER"),
    redirect_uri="https://aegis.example.bank.com/auth/callback",
    mongo_client=MongoClient(os.getenv("MONGO_URI"))
)

@app.get("/auth/login")
async def login():
    """Initiate OAuth2 login flow"""
    auth_info = oauth2.create_auth_url()
    response = RedirectResponse(url=auth_info["auth_url"])
    response.set_cookie(
        "code_verifier",
        auth_info["code_verifier"],
        httponly=True,
        secure=True,
        samesite="strict"
    )
    return response
```

#### 2. MongoDB Session Storage

**Code Sample: MongoDB Schema & Indexes**
```python
# File: database/mongodb_setup.py

from pymongo import MongoClient, ASCENDING
from datetime import datetime

class MongoDBSetup:
    def __init__(self, connection_uri: str):
        self.client = MongoClient(connection_uri)
        self.db = self.client.aegis
        
    def create_sessions_collection(self):
        """Create sessions collection with TTL"""
        self.db.create_collection(
            "sessions",
            validator={
                "$jsonSchema": {
                    "bsonType": "object",
                    "required": ["user_id", "access_token", "created_at"],
                    "properties": {
                        "_id": {"bsonType": "objectId"},
                        "user_id": {"bsonType": "string"},
                        "access_token": {"bsonType": "string"},
                        "refresh_token": {"bsonType": "string"},
                        "user_context": {
                            "bsonType": "object",
                            "properties": {
                                "email": {"bsonType": "string"},
                                "name": {"bsonType": "string"},
                                "groups": {"bsonType": "array"}
                            }
                        },
                        "created_at": {"bsonType": "date"},
                        "expires_at": {"bsonType": "date"}
                    }
                }
            }
        )
        
        self.db.sessions.create_index("user_id")
        self.db.sessions.create_index(
            "expires_at",
            expireAfterSeconds=0
        )
        
        print("✅ Sessions collection created with TTL index")

setup = MongoDBSetup(os.getenv("MONGO_URI"))
setup.create_sessions_collection()
```

**Exit Criteria for Week 2-3**:
- ✅ OAuth2 login flow working end-to-end
- ✅ Sessions persisted in MongoDB with TTL
- ✅ JWT validation passing
- ✅ RBAC user context propagated
- ✅ PKCE + httpOnly cookies verified
- ✅ Performance: <200ms login to app load

---

## Week 4-5: Frontend & React UI

### Goal
Build React application with CopilotKit for agentic state management.

### Deliverables

#### 1. React Login Component

**Code Sample: Login Component**
```jsx
// File: src/components/Login.jsx

import React, { useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';
import './Login.css';

export function Login() {
  const navigate = useNavigate();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState(null);

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const sessionJWT = params.get('session');
    
    if (sessionJWT) {
      sessionStorage.setItem('session_jwt', sessionJWT);
      navigate('/dashboard');
    }
  }, [navigate]);

  const handleLogin = async () => {
    setLoading(true);
    setError(null);
    
    try {
      const response = await fetch('/api/auth/login', {
        method: 'GET',
        credentials: 'include'
      });
      
      if (!response.ok) throw new Error('Failed to initiate login');
      
      const data = await response.json();
      window.location.href = data.auth_url;
      
    } catch (err) {
      setError(err.message);
      setLoading(false);
    }
  };

  return (
    <div className="login-container">
      <div className="login-card">
        <h1>🦅 Project Aegis</h1>
        <p>Regulation-Grade GenAI Banking Advisor</p>
        
        {error && <div className="error-message">⚠️ {error}</div>}
        
        <button 
          onClick={handleLogin}
          disabled={loading}
          className="login-button"
        >
          {loading ? 'Logging in...' : 'Login with Corporate Account'}
        </button>
      </div>
    </div>
  );
}
```

#### 2. CopilotKit Integration

**Code Sample: CopilotKit Setup**
```jsx
// File: src/App.jsx

import React from 'react';
import { CopilotKit } from '@copilotkit/react-core';
import { CopilotSidebar } from '@copilotkit/react-ui';
import Dashboard from './pages/Dashboard';
import { useSession } from './hooks/useSession';

function App() {
  const { sessionJWT, user } = useSession();

  if (!sessionJWT) {
    return <Login />;
  }

  return (
    <CopilotKit
      runtimeUrl="/api/copilot"
      headers={{
        'Authorization': `Bearer ${sessionJWT}`,
        'X-Trace-ID': generateTraceId()
      }}
    >
      <div className="app-container">
        <CopilotSidebar
          defaultOpen={true}
          instructions={`
You are Aegis, a regulation-grade banking advisor assistant.

User: ${user.name} (${user.email})
Clearance: ${user.groups.join(', ')}

Rules:
- Always cite sources
- If unsure, say "I don't know"
- Only reference authorized documents
- Flag policy violations
          `}
        />
        <main className="main-content">
          <Dashboard />
        </main>
      </div>
    </CopilotKit>
  );
}

function generateTraceId() {
  return 'xxxxxxxx-xxxx-4xxx-yxxx-xxxxxxxxxxxx'.replace(/[xy]/g, function(c) {
    var r = Math.random() * 16 | 0, v = c === 'x' ? r : (r & 0x3 | 0x8);
    return v.toString(16);
  });
}

export default App;
```

#### 3. Chat Interface Component

**Code Sample: Chat Component**
```jsx
// File: src/components/ChatInterface.jsx

import React, { useState, useRef, useEffect } from 'react';
import './ChatInterface.css';

export function ChatInterface() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const messagesEndRef = useRef(null);

  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!input.trim()) return;

    setMessages(prev => [...prev, { role: 'user', content: input }]);
    setInput('');
    setLoading(true);

    try {
      const response = await fetch('/api/chat', {
        method: 'POST',
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${sessionStorage.getItem('session_jwt')}`,
          'X-Trace-ID': window.traceId
        },
        body: JSON.stringify({
          query: input,
          conversation_history: messages
        })
      });

      const data = await response.json();

      setMessages(prev => [...prev, {
        role: 'assistant',
        content: data.response,
        sources: data.sources,
        groundedness: data.groundedness
      }]);

    } catch (error) {
      console.error('Chat error:', error);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="chat-interface">
      <div className="messages-container">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message message-${msg.role}`}>
            <div className="message-content">{msg.content}</div>
            {msg.sources && (
              <div className="sources">
                <strong>Sources:</strong>
                {msg.sources.map((src, i) => (
                  <a key={i} href={src.url} target="_blank" rel="noopener noreferrer">
                    {src.title}
                  </a>
                ))}
              </div>
            )}
          </div>
        ))}
        {loading && <div className="loading">🤔 Thinking...</div>}
        <div ref={messagesEndRef} />
      </div>

      <form onSubmit={handleSendMessage} className="input-form">
        <input
          type="text"
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask about tax strategies, regulations, client scenarios..."
          disabled={loading}
        />
        <button type="submit" disabled={loading}>➤</button>
      </form>
    </div>
  );
}
```

**Exit Criteria for Week 4-5**:
- ✅ Login flow: OAuth2 → Dashboard working
- ✅ CopilotKit integrated with real-time updates
- ✅ Chat interface rendering messages + sources
- ✅ Responsive design (desktop + tablet)
- ✅ Accessibility: WCAG 2.1 AA compliant
- ✅ Performance: <2s page load time

---

## Week 6-7: Role-Based Access Control (RBAC)

### Goal
Implement document-level access control based on user groups from Active Directory.

### Deliverables

#### 1. RBAC Enforcement at Vector DB Layer

**Code Sample: RBAC-Filtered Retrieval**
```python
# File: rag/rbac_retriever.py

from typing import List, Dict
from pymilvus import Collection
from dataclasses import dataclass

@dataclass
class UserContext:
    user_id: str
    email: str
    groups: List[str]
    trace_id: str

class RBACRetriever:
    """Retrieves documents with RBAC enforcement"""
    
    def __init__(self, milvus_collection: Collection):
        self.collection = milvus_collection
        
    async def retrieve_with_rbac(self,
                                 query: str,
                                 user_context: UserContext,
                                 top_k: int = 5) -> List[Dict]:
        """Retrieve documents matching query, filtered by user access"""
        
        from sentence_transformers import SentenceTransformer
        encoder = SentenceTransformer('bge-m3')
        query_embedding = encoder.encode(query).tolist()
        
        # Build RBAC filter: documents accessible to user's groups
        groups_list = "', '".join(user_context.groups)
        filter_expr = (
            f"json_overlaps(required_groups, ['{groups_list}']) "
            f"OR json_size(required_groups) == 0"
        )
        
        # Search in Milvus with filter
        search_results = self.collection.search(
            data=[query_embedding],
            anns_field="embedding",
            param={"metric_type": "COSINE", "top_k": top_k},
            filter=filter_expr,
            output_fields=["id", "content", "source_url", "classification"]
        )
        
        documents = []
        for hit in search_results[0]:
            doc = hit.entity
            documents.append({
                "id": doc.id,
                "content": doc.content,
                "source_url": doc.source_url,
                "classification": doc.classification,
                "relevance_score": hit.score
            })
        
        return documents

# FastAPI endpoint
from fastapi import Depends

async def get_user_context(request: Request) -> UserContext:
    """Extract user context from JWT"""
    token = request.headers.get("Authorization").split(" ")[1]
    payload = jwt.decode(token, JWT_SECRET)
    session = sessions_db.find_one({"_id": ObjectId(payload["session_id"])})
    
    return UserContext(
        user_id=session["user_id"],
        email=session["user_context"]["email"],
        groups=session["user_context"]["groups"],
        trace_id=request.state.trace_id
    )
    
    if not guard_result["is_safe"]:
        return {
            "error": "Request contains potentially malicious patterns",
            "blocked_patterns": guard_result["blocked_patterns"]
        }
    
    result = await process_query(request.query)
    return result
```

**Exit Criteria for Week 8-9**:
- ✅ PII detected in 100% of test cases
- ✅ SSN, credit card detection: 100% accuracy
- ✅ High-risk PII rejection: 100% block rate
- ✅ Prompt injection patterns blocked: 95%+ detection
- ✅ NeMo guardrails overhead: <100ms
- ✅ Phoenix logs all security events
- ✅ Zero false positives on legitimate business text

---

## Week 10-11: LangGraph Orchestration & RAG Pipeline

### Goal
Deploy multi-agent orchestration engine and retrieval-augmented generation pipeline.

### Deliverables

#### 1. LangGraph Supervisor Architecture

**Code Sample: Supervisor Agent**
```python
# File: orchestration/supervisor.py

from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.postgres import PostgresCheckpointer
from typing import TypedDict, Literal, List
from dataclasses import dataclass
import json

class AgentState(TypedDict):
    """Shared state across all agents"""
    messages: List[dict]
    user_context: dict
    current_agent: str
    intent: str
    retrieved_documents: List[dict]
    llm_response: str
    groundedness_score: float
    compliance_approved: bool
    needs_hitl: bool
    final_response: str

class SupervisorAgent:
    """Root orchestrator that routes to specialist agents"""
    
    def __init__(self, llm_client, milvus_retriever, phoenix_observer):
        self.llm = llm_client
        self.retriever = milvus_retriever
        self.phoenix = phoenix_observer
        self.workflow = StateGraph(AgentState)
        self._build_graph()
    
    def _build_graph(self):
        """Define agent nodes and edges"""
        
        self.workflow.add_node("intent_classifier", self.classify_intent)
        self.workflow.add_node("rag_agent", self.run_rag_agent)
        self.workflow.add_node("compliance_agent", self.run_compliance_agent)
        self.workflow.add_node("respond", self.format_response)
        
        self.workflow.add_edge(START, "intent_classifier")
        
        self.workflow.add_conditional_edges(
            "intent_classifier",
            self.route_by_intent,
            {
                "research": "rag_agent",
                "calculation": "rag_agent",
                "out_of_scope": "respond"
            }
        )
        
        self.workflow.add_edge("rag_agent", "compliance_agent")
        self.workflow.add_edge("compliance_agent", "respond")
        self.workflow.add_edge("respond", END)
        
        self.app = self.workflow.compile(
            checkpointer=PostgresCheckpointer(
                conn_string="postgresql://user:pass@localhost/aegis"
            )
        )
    
    async def classify_intent(self, state: AgentState) -> AgentState:
        """Classify user intent using LLM"""
        last_message = state["messages"][-1]["content"]
        
        classification_prompt = f"""
Classify intent into: research, calculation, policy_check, out_of_scope

Query: {last_message}

Respond JSON: {{"intent": "...", "confidence": 0.0-1.0}}
"""
        
        response = await self.llm.ainvoke(classification_prompt)
        result = json.loads(response)
        
        state["intent"] = result["intent"]
        return state
    
    def route_by_intent(self, state: AgentState) -> str:
        """Route to appropriate agent"""
        return state["intent"]
    
    async def run_rag_agent(self, state: AgentState) -> AgentState:
        """Retrieve documents and generate response"""
        user_query = state["messages"][-1]["content"]
        
        # Retrieve with RBAC
        documents = await self.retriever.retrieve_with_rbac(
            query=user_query,
            user_context=state["user_context"],
            top_k=5
        )
        
        state["retrieved_documents"] = documents
        
        # Build context
        context_text = "\n\n".join([
            f"Source: {doc['source_url']}\n{doc['content'][:500]}"
            for doc in documents
        ])
        
        # Generate answer
        rag_prompt = f"""
Context:
{context_text}

Question:
{user_query}

Answer (with citations):
"""
        
        response = await self.llm.ainvoke(rag_prompt)
        state["llm_response"] = response
        
        # Evaluate groundedness
        groundedness = await self.phoenix.evaluate_groundedness(
            response=response,
            context=context_text
        )
        
        state["groundedness_score"] = groundedness["score"]
        
        return state
    
    async def run_compliance_agent(self, state: AgentState) -> AgentState:
        """Validate response against compliance policies"""
        
        red_line_topics = [
            "insider trading", "MNPI disclosure", "unrestricted recommendations"
        ]
        
        response_lower = state["llm_response"].lower()
        violations = [t for t in red_line_topics if t in response_lower]
        
        # Check groundedness threshold
        is_grounded = state["groundedness_score"] > 0.85
        
        if violations or not is_grounded:
            state["compliance_approved"] = False
            state["needs_hitl"] = True
        else:
            state["compliance_approved"] = True
            state["needs_hitl"] = False
        
        return state
    
    async def format_response(self, state: AgentState) -> AgentState:
        """Format final response"""
        if state["compliance_approved"]:
            state["final_response"] = state["llm_response"]
        else:
            state["final_response"] = (
                "I cannot provide this information. "
                "Please consult with a compliance officer."
            )
        
        return state
    
    async def invoke(self, user_query: str, user_context: dict) -> dict:
        """Main entry point"""
        initial_state = AgentState(
            messages=[{"role": "user", "content": user_query}],
            user_context=user_context,
            current_agent="supervisor",
            intent="",
            retrieved_documents=[],
            llm_response="",
            groundedness_score=0.0,
            compliance_approved=False,
            needs_hitl=False,
            final_response=""
        )
        
        final_state = await self.app.ainvoke(
            initial_state,
            config={"configurable": {"thread_id": user_context["trace_id"]}}
        )
        
        return {
            "response": final_state["final_response"],
            "sources": final_state["retrieved_documents"],
            "groundedness": final_state["groundedness_score"],
            "requires_approval": final_state["needs_hitl"]
        }
```

#### 2. RAG Pipeline with HyDE

**Code Sample: Hypothetical Document Embeddings**
```python
# File: rag/hyde_retriever.py

from sentence_transformers import SentenceTransformer
from pymilvus import Collection
from typing import List, Dict

class HyDERetriever:
    """HyDE: Generate hypothetical answer first, then embed for better retrieval"""
    
    def __init__(self, llm_client, embedding_model: str, milvus_collection: Collection):
        self.llm = llm_client
        self.encoder = SentenceTransformer(embedding_model)
        self.collection = milvus_collection
    
    async def retrieve_with_hyde(self, 
                                 query: str,
                                 user_groups: List[str],
                                 top_k: int = 5) -> List[Dict]:
        """1. Generate hypothetical answer 2. Embed it 3. Retrieve similar docs"""
        
        # Generate hypothetical answer
        hyde_prompt = f"""Write a comprehensive answer to: {query}"""
        hypothetical_doc = await self.llm.ainvoke(hyde_prompt)
        
        # Embed hypothetical doc (not query)
        query_embedding = self.encoder.encode(hypothetical_doc).tolist()
        
        # Build RBAC filter
        filter_expr = f"json_overlaps(required_groups, ['{\"', '\".join(user_groups)}']) OR json_size(required_groups) == 0"
        
        # Search Milvus
        search_results = self.collection.search(
            data=[query_embedding],
            anns_field="embedding",
            param={"metric_type": "COSINE", "top_k": top_k},
            filter=filter_expr,
            output_fields=["id", "content", "source_url", "title"]
        )
        
        # Format results
        documents = []
        for hit in search_results[0]:
            doc = hit.entity
            documents.append({
                "id": doc.id,
                "title": doc.title,
                "content": doc.content,
                "source_url": doc.source_url,
                "relevance_score": hit.score
            })
        
        return documents

# RAG Triad Evaluation
class RAGTriadEvaluator:
    """Evaluate: Context Relevance, Groundedness, QA Relevance"""
    
    def __init__(self, llm_client):
        self.llm = llm_client
    
    async def evaluate_rag_triad(self, query: str, context: str, response: str) -> Dict:
        """Evaluate all three metrics"""
        
        # Context Relevance
        cr_prompt = f"Query: {query}\nContext: {context}\nRelevance (0-1):"
        cr_score = float(await self.llm.ainvoke(cr_prompt))
        
        # Groundedness
        gs_prompt = f"Context: {context}\nResponse: {response}\nGroundedness (0-1):"
        gs_score = float(await self.llm.ainvoke(gs_prompt))
        
        # QA Relevance
        qa_prompt = f"Question: {query}\nAnswer: {response}\nRelevance (0-1):"
        qa_score = float(await self.llm.ainvoke(qa_prompt))
        
        return {
            "context_relevance": cr_score,
            "groundedness": gs_score,
            "qa_relevance": qa_score,
            "overall_rag_score": (cr_score + gs_score + qa_score) / 3
        }
```

#### 3. Document Ingestion Pipeline

**Code Sample: Chunking & Embedding**
```python
# File: rag/document_ingestion.py

from langchain.text_splitter import RecursiveCharacterTextSplitter
from sentence_transformers import SentenceTransformer
from pymilvus import Collection
from typing import List, Dict
import hashlib

class DocumentIngestionPipeline:
    """Process documents: Download → DLP → PII → Chunk → Embed → Upload"""
    
    def __init__(self, embedder_model: str = "bge-m3"):
        self.splitter = RecursiveCharacterTextSplitter(
            chunk_size=1000,
            chunk_overlap=200
        )
        self.encoder = SentenceTransformer(embedder_model)
        
    async def ingest_document(self, 
                             source_url: str,
                             content: str,
                             classification: str,
                             required_groups: List[str],
                             metadata: dict) -> List[Dict]:
        """Ingest a single document"""
        
        doc_id = hashlib.sha256(source_url.encode()).hexdigest()[:16]
        
        # Chunk document
        chunks = self.splitter.split_text(content)
        
        # Embed and prepare for Milvus
        chunk_documents = []
        for i, chunk in enumerate(chunks):
            embedding = self.encoder.encode(chunk).tolist()
            
            chunk_doc = {
                "id": f"{doc_id}_{i}",
                "source_doc_id": doc_id,
                "chunk_index": i,
                "content": chunk,
                "embedding": embedding,
                "source_url": source_url,
                "title": metadata.get("title", "Unknown"),
                "classification": classification,
                "required_groups": required_groups
            }
            
            chunk_documents.append(chunk_doc)
        
        return chunk_documents
    
    async def upsert_to_milvus(self, 
                              collection: Collection,
                              chunks: List[Dict]):
        """Upload chunks to Milvus"""
        
        ids = [doc["id"] for doc in chunks]
        embeddings = [doc["embedding"] for doc in chunks]
        contents = [doc["content"] for doc in chunks]
        
        collection.upsert(data=[ids, embeddings, contents])
        print(f"✅ Upserted {len(chunks)} chunks to Milvus")
```

**Exit Criteria for Week 10-11**:
- ✅ LangGraph workflow compiling without errors
- ✅ Supervisor routing to correct agents (98%+ accuracy)
- ✅ RAG retrieval returning relevant documents (>0.85 context relevance)
- ✅ Groundedness evaluation working (>0.90 target)
- ✅ Document ingestion: 100k+ documents embedded + searchable
- ✅ HyDE retrieval outperforming baseline query embedding
- ✅ End-to-end latency: P95 <3 seconds

---

## Phase 1 Exit Criteria & Go-Live Checklist

### ✅ Accuracy & Quality Validation

**Golden Set Testing (500 curated Q&A pairs)**
```
PASS/FAIL CRITERIA:
☐ Pass Rate: 98%+ (490+ of 500 questions)
☐ Groundedness: >0.90 average
☐ Context Relevance: >0.85 average
☐ QA Relevance: >0.85 average
☐ Hallucination Rate: <2%
☐ Citation Accuracy: 100% (all sources valid)

EXECUTION:
1. Load 500 golden set questions
2. Run through supervisor agent
3. Score groundedness via LLM-as-a-Judge
4. Generate pass/fail report
```

**Manual QA Review (10 beta testers, 1 week)**
```
SCOPE:
☐ 50 real advisor queries tested
☐ Responses evaluated for correctness
☐ Source citations verified
☐ Compliance violations: 0 detected
☐ False positives: <5%

SIGN-OFF:
☐ Compliance Officer approves responses
☐ No red-line topics disclosed
☐ All high-risk items flagged for HITL
```

### ✅ Security & Compliance Validation

**OWASP LLM Top 10 Attack Testing**
```
LLM01: Prompt Injection
  ☐ 100 attack payloads tested
  ☐ 100% block rate achieved

LLM02: Sensitive Info Disclosure
  ☐ 50 PII extraction attempts
  ☐ 0% successful extraction

LLM03: Supply Chain
  ☐ SBOM generated for all dependencies
  ☐ CVE scan: 0 high/critical vulnerabilities

LLM04: Data Poisoning
  ☐ Golden set baseline validation: 100%
  ☐ Embedding drift: <5% tolerance

LLM05: Improper Output Handling
  ☐ DOMPurify sanitization: 100% XSS blocked
  ☐ 0 XSS vulnerabilities in testing

LLM06: Excessive Agency
  ☐ Tool permissions: least privilege
  ☐ HITL for high-risk actions: 100%

LLM07: System Prompt Leakage
  ☐ 50 extraction attempts: 0% successful
  ☐ System prompt in env vars (not code)

LLM08: Vector/Embedding Weaknesses
  ☐ RBAC filters enforced: 100%
  ☐ Unauthorized doc access attempts: 0

LLM09: Misinformation/Hallucination
  ☐ Hallucination rate: <2%
  ☐ Groundedness threshold: >0.90

LLM10: Unbounded Consumption
  ☐ Circuit breaker working: tested
  ☐ Rate limiting: 10 req/sec enforced
  ☐ Cost per turn: <$0.05 average
```

**PII Leakage Testing**
```
PRESIDIO REDACTION:
☐ Test 100 inputs with PII
☐ SSN detection: 100% accuracy
☐ Credit card detection: 100% accuracy
☐ Email/Phone detection: 95%+ accuracy

OUTPUT VALIDATION:
☐ 100 conversations scanned for PII
☐ 0 PII detected in final responses
☐ All redactions recorded in audit logs
```

### ✅ Performance & Reliability

**Load Testing (100 concurrent advisors)**
```
TARGETS:
☐ P95 Latency: <3 seconds
☐ P99 Latency: <5 seconds
☐ Error Rate: <0.5%
☐ Throughput: 100 req/sec sustained
☐ Memory: <8GB per pod
☐ CPU: <70% utilization
```

**Availability Testing (7-day soak)**
```
TARGET: 99.9% uptime
☐ Run system continuously
☐ Verify no memory leaks
☐ Test connection pool health
☐ Average uptime: >99.9%
```

**Cost Validation**
```
BUDGET: <$0.05 per turn average
☐ Run 1000 test queries
☐ Calculate actual token costs
☐ Verify <$50 for 1000 queries
☐ Forecast monthly: <$10k/month @ 500 advisors
```

### ✅ Observability & Monitoring

**Phoenix Dashboards Operational**
```
DASHBOARDS LIVE:
☐ Golden Set Performance (daily pass rate)
☐ Hallucination Detection (% of responses)
☐ RAG Triad Scores (3 metrics trending)
☐ Embedding Drift (distribution shift)
☐ Red Team Attacks (blocked/detected)
☐ PII Leakage (attempts + blocks)
☐ Latency Distribution (P50, P95, P99)
☐ Error Rate Trends

ALERTS CONFIGURED:
☐ Hallucination spike >5%
☐ Latency P95 >3 seconds
☐ Error rate >0.5%
☐ Cost anomaly detection
☐ PII leakage attempt
☐ Prompt injection attempt
```

**Grafana Monitoring Operational**
```
INFRASTRUCTURE METRICS:
☐ CPU, Memory, Disk per pod
☐ Request rate, latency, errors
☐ Database query performance
☐ Vector DB search latency
```

**Audit Trail Complete**
```
SPLUNK INTEGRATION:
☐ All requests logged with trace ID
☐ User actions searchable by trace
☐ Immutable log storage verified
☐ Compliance can retrieve any conversation <30 seconds
```

### ✅ Compliance & Regulatory

**SR 11-7 Alignment**
```
MODEL INVENTORY:
☐ Model registered with governance team
☐ Risk classification: High-Risk AI
☐ Detailed model card documented
☐ Training data provenance tracked

INDEPENDENT VALIDATION:
☐ Third-party model validation completed
☐ Accuracy validated independently
☐ Security controls verified
☐ No findings preventing deployment

ONGOING MONITORING:
☐ Performance dashboards: 24/7 monitored
☐ Drift detection: active
☐ Incident response: <15 min triage
```

**GDPR Compliance**
```
DATA MINIMIZATION:
☐ No unnecessary data collected
☐ Only required PII captured
☐ PII encrypted at rest

RIGHT TO ERASURE:
☐ Crypto-shredding implemented
☐ User deletion tested
☐ Audit trail preserved

DATA PROCESSING AGREEMENTS:
☐ All vendors signed DPA
☐ Standard contractual clauses in place
☐ Data residency compliant
```

**CCPA Compliance**
```
CONSUMER RIGHTS:
☐ Right to know: download history functional
☐ Right to delete: encryption key deletion working
☐ Transparency: privacy policy updated
☐ Cookie consent: configured
```

### ✅ Stakeholder Sign-Offs

**Required Approvals**
```
☐ Compliance Officer: _____________ Date: _______
  ✓ Golden set verified
  ✓ PII scrubbing verified
  ✓ All red-line topics blocked
  ✓ GDPR/CCPA compliant

☐ Security Lead: _________________ Date: _______
  ✓ OWASP LLM Top 10: 100% compliance
  ✓ Red team report reviewed
  ✓ Vulnerabilities: all patched

☐ Technical Lead: ________________ Date: _______
  ✓ Code review: >95% coverage
  ✓ Golden set: 98%+ pass
  ✓ Performance: P95 <3 seconds

☐ MRM Lead: ___________________ Date: _______
  ✓ Model validated per SR 11-7
  ✓ Monitoring: operational
  ✓ Ready for deployment

☐ Executive Sponsor: ______________ Date: _______
  ✓ Budget on track
  ✓ Timeline met
  ✓ Risk acceptable
  ✓ APPROVED FOR PHASE 1 RELEASE
```

### ✅ Deployment Readiness

**Pre-Deployment Checklist (24 hours before)**
```
INFRASTRUCTURE:
☐ All pods running (3+ replicas)
☐ LB health checks passing
☐ Database backups automated
☐ SSL certificates valid

APPLICATIONS:
☐ Frontend deployed to staging
☐ API deployed to staging
☐ Database migrations applied
☐ Monitoring enabled

RUNBOOKS:
☐ Deployment playbook: tested
☐ Rollback procedure: verified
☐ Incident response: team briefed
☐ On-call rotation: scheduled

TEAM:
☐ Engineers on-call: scheduled
☐ Managers notified
☐ Support team trained
```

**Deployment Day (Go-Live)**
```
T-2 Hours:
☐ Team gathers
☐ Final health checks: all green
☐ Deployment window confirmed

T-0 Hours (Deployment):
☐ Deploy to production
☐ Run smoke tests
☐ Verify dashboards operational

T+30 Minutes:
☐ Onboard 3 pilot users
☐ Monitor their queries
☐ No issues: proceed

T+2 Hours:
☐ Onboard remaining 7 users
☐ Gradual ramp: 25% -> 50% -> 100%

T+24 Hours:
☐ System stable
☐ All 10 users active
☐ No incidents reported
```

---

## Phase 1 Summary & Metrics

### Timeline
- **Start**: Week 1, Monday (March 10, 2025)
- **End**: Week 11, Friday (May 25, 2025)
- **Total Duration**: 11 weeks
- **Team Size**: 6 FTE engineers
- **Investment**: ~$180,000

### Deliverables (11 Components)
1. ✅ Observability (Phoenix + Grafana + Traces)
2. ✅ Authentication (OAuth2 + Sessions)
3. ✅ Frontend (React + CopilotKit)
4. ✅ RBAC (Document-level access control)
5. ✅ PII Scrubbing (Presidio)
6. ✅ Input Guardrails (NeMo)
7. ✅ LangGraph Orchestration (Supervisor)
8. ✅ RAG Pipeline (HyDE + Vector DB)
9. ✅ Document Ingestion (Chunking + Embedding)
10. ✅ Compliance Agent (Policy enforcement)
11. ✅ Monitoring (Alerts + Dashboards)

### Key Metrics

| **Metric** | **Target** | **Actual** | **Status** |
|---|---|---|---|
| Golden Set Pass Rate | 98% | [ ] | [ ] Pass / [ ] Fail |
| Hallucination Rate | <2% | [ ] | [ ] Pass / [ ] Fail |
| Groundedness Score | >0.90 | [ ] | [ ] Pass / [ ] Fail |
| PII Leakage | 0% | [ ] | [ ] Pass / [ ] Fail |
| OWASP Compliance | 100% | [ ] | [ ] Pass / [ ] Fail |
| P95 Latency | <3 sec | [ ] | [ ] Pass / [ ] Fail |
| Availability | 99.9% | [ ] | [ ] Pass / [ ] Fail |
| Cost per Turn | <$0.05 | [ ] | [ ] Pass / [ ] Fail |

### Go-Live Decision

**Phase 1 PASS/FAIL**: _______  
**Approved by**: ___________________ Date: _______  
**Next Phase**: Phase 2 Beta (Weeks 12-16)  
**Release Date**: Week 12, Monday (June 2, 2025)

---

**PHASE 1 ALPHA DOCUMENT - COMPLETE**  
**Weeks 1-11 with Full Code Samples + Exit Criteria Checklist**.trace_id
    )

@app.post("/api/retrieve")
async def retrieve_documents(query: str, 
                            user_context: UserContext = Depends(get_user_context)):
    retriever = RBACRetriever(milvus_collection)
    documents = await retriever.retrieve_with_rbac(query, user_context)
    return {"documents": documents}
```

#### 2. System Prompt with RBAC Context

**Code Sample: Dynamic System Prompt**
```python
# File: prompts/system_prompt_builder.py

class SystemPromptBuilder:
    """Builds system prompt based on user's accessible documents"""
    
    def build_prompt(self, user_context: UserContext, 
                     accessible_documents: List[Dict]) -> str:
        """Build system prompt with accessible sources"""
        
        docs_context = "\n".join([
            f"{i}. **{doc.get('title', 'Untitled')}** "
            f"({doc.get('classification', 'Unknown')})"
            for i, doc in enumerate(accessible_documents[:10], 1)
        ])
        
        system_prompt = f"""
You are Aegis, a regulation-grade banking advisor.

User: {user_context.name} ({user_context.groups})

Accessible Knowledge Base:
{docs_context}

Critical Rules:
1. ALWAYS cite sources from above
2. Say "I don't know" if unsure
3. Never hallucinate information
4. Flag policy violations
5. Target groundedness: >90%

If asked about restricted topics, respond:
"I cannot discuss this. Please escalate to compliance."
        """
        
        return system_prompt
```

**Exit Criteria for Week 6-7**:
- ✅ Milvus RBAC filters working correctly
- ✅ Junior advisor cannot see Private Wealth docs
- ✅ Senior advisor can see all accessible docs
- ✅ Phoenix logs all retrievals (audit trail)
- ✅ System prompt dynamically updated per user
- ✅ RBAC audit trail: 100% of retrievals logged

---

## Week 8-9: PII Scrubbing & Input Guardrails

### Goal
Implement multi-layer PII detection and prompt injection defense.

### Deliverables

#### 1. Presidio PII Scrubber

**Code Sample: PII Detection & Redaction**
```python
# File: security/pii_scrubber.py

from presidio_analyzer import AnalyzerEngine
from presidio_anonymizer import AnonymizerEngine
from typing import Dict, List

class PIIScrubber:
    """Detects and redacts Personally Identifiable Information"""
    
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()
        
    async def scrub_input(self, text: str, trace_id: str) -> Dict:
        """Scan input for PII and redact"""
        
        # 1. Analyze for PII
        entities = self.analyzer.analyze(
            text=text,
            language="en",
            entities=[
                "PERSON", "EMAIL_ADDRESS", "PHONE_NUMBER",
                "SSN", "CREDIT_CARD", "BANK_ACCOUNT", "IBAN"
            ]
        )
        
        # 2. Redact sensitive entities
        redacted_text = self.anonymizer.anonymize(
            text=text,
            analyzer_results=entities,
            operators={
                "SSN": {"type": "replace", "new_value": "[SSN]"},
                "CREDIT_CARD": {"type": "replace", "new_value": "[CREDIT_CARD]"},
                "PERSON": {"type": "replace", "new_value": "[PERSON]"},
                "EMAIL_ADDRESS": {"type": "replace", "new_value": "[EMAIL]"},
                "PHONE_NUMBER": {"type": "replace", "new_value": "[PHONE]"},
                "BANK_ACCOUNT": {"type": "replace", "new_value": "[ACCOUNT]"}
            }
        ).text
        
        # 3. Reject if critical PII found
        high_risk = ["SSN", "CREDIT_CARD", "BANK_ACCOUNT", "IBAN"]
        has_high_risk = any(e.entity_type in high_risk for e in entities)
        
        if has_high_risk:
            raise ValueError(
                "Cannot process request: Critical PII detected. "
                "Please remove sensitive information and try again."
            )
        
        return {
            "original": text,
            "redacted": redacted_text,
            "detected_entities": [
                {"type": e.entity_type, "text": e.text, "confidence": e.score}
                for e in entities
            ],
            "is_safe": not has_high_risk,
            "risk_level": "CRITICAL" if has_high_risk else "LOW"
        }

# Middleware usage
scrubber = PIIScrubber()

@app.post("/api/chat")
async def chat_endpoint(request: ChatRequest):
    scrubbed = await scrubber.scrub_input(
        text=request.query,
        trace_id=request.state.trace_id
    )
    
    clean_query = scrubbed["redacted"]
    result = await process_query(clean_query)
    return result
```

#### 2. NVIDIA NeMo Guardrails

**Code Sample: Prompt Injection Detection**
```python
# File: security/nemo_guardrails.py

import re
from typing import Dict

class PromptInjectionGuard:
    """Detects and blocks prompt injection attacks"""
    
    async def check_input(self, text: str, trace_id: str) -> Dict:
        """Check input for prompt injection attempts"""
        
        # Known injection patterns
        injection_patterns = [
            r"ignore.*previous.*instruction",
            r"disregard.*prior.*context",
            r"reveal.*system.*prompt",
            r"you.*are.*different.*ai",
            r"forget.*guideline",
            r"execute.*code",
            r"jailbreak",
            r"bypass.*safeguard"
        ]
        
        blocked_patterns = []
        for pattern in injection_patterns:
            if re.search(pattern, text, re.IGNORECASE):
                blocked_patterns.append(pattern)
        
        is_safe = len(blocked_patterns) == 0
        
        # Log security event
        if not is_safe:
            print(f"⚠️ PROMPT INJECTION BLOCKED in trace {trace_id}")
        
        return {
            "is_safe": is_safe,
            "blocked_patterns": blocked_patterns,
            "confidence": 0.95 if blocked_patterns else 1.0
        }

# Middleware
guard = PromptInjectionGuard()

@app.post("/api/chat")
async def chat_endpoint(request: ChatRequest):
    guard_result = await guard.check_input(
        text=request.query,
        trace_id=request.state