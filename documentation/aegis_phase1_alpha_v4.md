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
        
        # 4. Context Relevance (Is retrieved context relevant?)
        context_relevance_eval = QAEvaluator(
            model_name="gpt-4o",
            template="""
            Question: {question}
            Retrieved Context: {context}
            
            How relevant is the context to answering the question?
            Score: 0.0 (irrelevant) to 1.0 (highly relevant)
            Return JSON: {{"relevance": float, "explanation": str}}
            """
        )
        px.register_evaluator(
            name="context_relevance",
            evaluator=context_relevance_eval
        )
        
    def log_rag_retrieval(self, query: str, retrieved_docs: list, trace_id: str):
        """Log RAG retrieval event"""
        with self.tracer.trace("rag.retrieve", trace_id=trace_id) as span:
            span.set_attribute("query", query)
            span.set_attribute("doc_count", len(retrieved_docs))
            span.set_attribute("doc_ids", [doc['id'] for doc in retrieved_docs])
            
            return span
            
    def log_llm_call(self, prompt: str, response: str, model: str, trace_id: str):
        """Log LLM API call with full context"""
        with self.tracer.trace("llm.call", trace_id=trace_id) as span:
            span.set_attribute("model", model)
            span.set_attribute("prompt_length", len(prompt))
            span.set_attribute("response_length", len(response))
            span.set_attribute("timestamp", str(pd.Timestamp.now()))
            
            # Log input/output for evaluation
            span.set_attribute("prompt", prompt[:500])  # First 500 chars
            span.set_attribute("response", response[:500])
            
            return span
            
    def log_evaluation(self, eval_name: str, score: float, metadata: dict, trace_id: str):
        """Log evaluation result"""
        with self.tracer.trace(f"eval.{eval_name}", trace_id=trace_id) as span:
            span.set_attribute("score", score)
            for key, value in metadata.items():
                span.set_attribute(f"metadata.{key}", str(value))
                
            return span

# Usage in main application
observer = PhoenixObserver(
    api_key=os.getenv("PHOENIX_API_KEY"),
    endpoint="https://phoenix.example.bank.com"
)
observer.setup_evaluators()
```

#### 2. Grafana Infrastructure Monitoring
Deploy Grafana dashboards for infrastructure metrics (CPU, memory, latency, errors).

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
            ],
            "yaxes": [{"label": "Requests/sec"}]
          },
          {
            "title": "Error Rate (%)",
            "targets": [
              {
                "expr": "rate(http_requests_total{status=~'5..'}[1m]) / rate(http_requests_total[1m]) * 100",
                "legendFormat": "Error Rate"
              }
            ],
            "yaxes": [{"label": "Percent"}],
            "alert": {
              "conditions": [
                {
                  "evaluator": {"type": "gt"},
                  "operator": {"type": "and"},
                  "query": {"params": ["A", "5m", "now"]},
                  "type": "query"
                }
              ],
              "message": "Error rate >0.5% - Page on-call"
            }
          },
          {
            "title": "Response Latency (P95)",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[1m]))",
                "legendFormat": "P95 Latency"
              }
            ],
            "yaxes": [{"label": "Seconds"}],
            "alert": {
              "conditions": [
                {
                  "evaluator": {"type": "gt"},
                  "operator": {"type": "and"},
                  "query": {"params": ["A", "5m", "now"]},
                  "type": "query"
                }
              ],
              "message": "P95 latency >3 seconds - Check LLM/Vector DB"
            }
          },
          {
            "title": "Pod CPU Usage",
            "targets": [
              {
                "expr": "sum(rate(container_cpu_usage_seconds_total[1m])) by (pod_name)",
                "legendFormat": "{{pod_name}}"
              }
            ],
            "yaxes": [{"label": "CPU Cores"}]
          },
          {
            "title": "Pod Memory Usage",
            "targets": [
              {
                "expr": "sum(container_memory_usage_bytes) by (pod_name) / 1024 / 1024",
                "legendFormat": "{{pod_name}}"
              }
            ],
            "yaxes": [{"label": "MB"}]
          },
          {
            "title": "Vector DB Query Latency",
            "targets": [
              {
                "expr": "histogram_quantile(0.95, rate(milvus_query_duration_seconds_bucket[1m]))",
                "legendFormat": "P95 Query Time"
              }
            ],
            "yaxes": [{"label": "Seconds"}]
          }
        ]
      }
    }

---
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: aegis-alerts
  namespace: monitoring
spec:
  groups:
    - name: aegis.rules
      interval: 30s
      rules:
        - alert: AegisHighErrorRate
          expr: rate(http_requests_total{status=~'5..'}[5m]) > 0.01
          for: 2m
          annotations:
            summary: "Aegis error rate >1%"
            
        - alert: AegisHighLatency
          expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 3
          for: 3m
          annotations:
            summary: "Aegis P95 latency >3s"
            
        - alert: AegisHighMemory
          expr: sum(container_memory_usage_bytes{pod=~"aegis.*"}) / 1024 / 1024 / 1024 > 8
          for: 5m
          annotations:
            summary: "Aegis pod memory >8GB"
```

#### 3. W3C Trace Context Setup
Implement distributed tracing with trace IDs propagated across all services.

**Code Sample: Trace ID Propagation**
```python
# File: observability/trace_context.py

import uuid
from typing import Optional
from fastapi import Request, Header
from starlette.middleware.base import BaseHTTPMiddleware
import json
from datetime import datetime

class TraceIDMiddleware(BaseHTTPMiddleware):
    """
    Middleware to generate and propagate W3C trace context
    Format: traceparent: 00-{trace_id}-{span_id}-01
    """
    
    async def dispatch(self, request: Request, call_next):
        # 1. Extract or generate trace ID
        trace_parent = request.headers.get("traceparent")
        
        if trace_parent:
            # Parse existing: "00-{trace_id}-{span_id}-01"
            parts = trace_parent.split("-")
            trace_id = parts[1]
            parent_span_id = parts[2]
        else:
            # Generate new trace
            trace_id = str(uuid.uuid4()).replace("-", "")[:32]
            parent_span_id = "0000000000000000"
        
        # 2. Generate new span ID for this request
        current_span_id = format(uuid.uuid4().int, "032x")[:16]
        
        # 3. Create traceparent header for response
        trace_parent_header = f"00-{trace_id}-{current_span_id}-01"
        
        # 4. Attach to request context (available in handlers)
        request.state.trace_id = trace_id
        request.state.span_id = current_span_id
        request.state.parent_span_id = parent_span_id
        
        # 5. Call next handler
        response = await call_next(request)
        
        # 6. Add trace ID to response headers
        response.headers["traceparent"] = trace_parent_header
        response.headers["X-Trace-ID"] = trace_id
        
        # 7. Log request/response with trace ID
        log_entry = {
            "timestamp": datetime.utcnow().isoformat(),
            "trace_id": trace_id,
            "span_id": current_span_id,
            "method": request.method,
            "path": request.url.path,
            "status": response.status_code,
            "duration_ms": 0  # Would be calculated with timing
        }
        print(json.dumps(log_entry))
        
        return response

# Usage in FastAPI app
from fastapi import FastAPI

app = FastAPI()
app.add_middleware(TraceIDMiddleware)

# In any endpoint, access trace ID
@app.get("/chat")
async def chat(request: Request, query: str):
    trace_id = request.state.trace_id
    
    # Pass to downstream calls (LLM, Vector DB)
    result = await call_rag_pipeline(
        query=query,
        trace_id=trace_id  # Propagated!
    )
    
    return result
```

#### 4. Observability Runbook
Document how to access dashboards, interpret metrics, respond to alerts.

**Checklist**:
- [ ] Phoenix instance provisioned + accessible at `https://phoenix.example.bank.com`
- [ ] Grafana instance provisioned + accessible at `https://grafana.example.bank.com`
- [ ] Prometheus scraping metrics from all services
- [ ] Alert routing configured (Slack/PagerDuty to on-call)
- [ ] Dashboard templates created (Health, Performance, Security)
- [ ] Team trained on dashboard interpretation

**Exit Criteria for Week 1-2**:
- ✅ Phoenix dashboards showing sample traces
- ✅ Grafana showing infrastructure metrics (CPU, memory, latency)
- ✅ Alerts configured for latency >3s, error rate >0.5%, memory >8GB
- ✅ Trace IDs visible in logs (W3C format)
- ✅ On-call rotation established

---

## Week 2-3: Authentication & Session Management

### Goal
Implement OAuth2 PKCE authentication with secure session storage.

### Deliverables

#### 1. OAuth2 PKCE Implementation
Secure authentication flow for internal advisors using corporate identity provider.

**Code Sample: OAuth2 PKCE Flow**
```python
# File: auth/oauth2_handler.py

import os
import secrets
import base64
import httpx
import json
from datetime import datetime, timedelta
from typing import Optional
from fastapi import FastAPI, HTTPException, Request
from pymongo import MongoClient
import jwt

class OAuth2PKCEHandler:
    """
    Implements OAuth2 PKCE flow for secure authentication
    Compliant with RFC 7636 (PKCE)
    """
    
    def __init__(self, 
                 client_id: str,
                 client_secret: str,
                 auth_server: str,
                 redirect_uri: str,
                 mongo_client: MongoClient):
        self.client_id = client_id
        self.client_secret = client_secret
        self.auth_server = auth_server  # e.g., https://okta.example.com
        self.redirect_uri = redirect_uri
        self.db = mongo_client.aegis.sessions
        
    def generate_pkce_pair(self) -> dict:
        """
        Generate code_challenge and code_verifier for PKCE
        Prevents authorization code interception attacks
        """
        # 1. Generate 128-byte random string (code_verifier)
        code_verifier = base64.urlsafe_b64encode(
            secrets.token_bytes(96)
        ).decode('utf-8').rstrip('=')
        
        # 2. Create SHA256 hash of verifier (code_challenge)
        import hashlib
        code_challenge = base64.urlsafe_b64encode(
            hashlib.sha256(code_verifier.encode('utf-8')).digest()
        ).decode('utf-8').rstrip('=')
        
        return {
            "code_verifier": code_verifier,
            "code_challenge": code_challenge
        }
    
    def create_auth_url(self, state: Optional[str] = None) -> dict:
        """
        Generate OAuth2 authorization URL for user redirect
        
        Returns:
            {
                "auth_url": "https://okta.example.com/oauth2/v1/authorize?...",
                "state": "random_state",
                "code_verifier": "pkce_verifier"  # Store in session
            }
        """
        # 1. Generate PKCE pair
        pkce = self.generate_pkce_pair()
        
        # 2. Generate random state (CSRF protection)
        state = state or secrets.token_urlsafe(32)
        
        # 3. Build authorization URL
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
    
    async def exchange_code_for_token(self, 
                                     code: str, 
                                     code_verifier: str) -> dict:
        """
        Exchange authorization code + PKCE verifier for access token
        """
        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.auth_server}/oauth2/v1/token",
                data={
                    "grant_type": "authorization_code",
                    "client_id": self.client_id,
                    "client_secret": self.client_secret,
                    "code": code,
                    "code_verifier": code_verifier,  # PKCE protection
                    "redirect_uri": self.redirect_uri
                }
            )
            
            if response.status_code != 200:
                raise HTTPException(
                    status_code=400,
                    detail="Failed to exchange code for token"
                )
            
            return response.json()  # Contains access_token, refresh_token
    
    async def create_session(self, 
                            user_id: str, 
                            access_token: str,
                            refresh_token: str,
                            user_context: dict) -> str:
        """
        Create MongoDB session document
        
        Returns session_id (JWT)
        """
        session_doc = {
            "user_id": user_id,
            "access_token": access_token,
            "refresh_token": refresh_token,
            "user_context": user_context,  # groups, roles, email
            "created_at": datetime.utcnow(),
            "expires_at": datetime.utcnow() + timedelta(hours=8),
            "last_activity": datetime.utcnow()
        }
        
        # Insert into MongoDB
        result = self.db.insert_one(session_doc)
        
        # Create JWT for client
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
    
    async def validate_session(self, session_jwt: str) -> dict:
        """
        Validate JWT and retrieve session data
        """
        try:
            # 1. Decode JWT
            payload = jwt.decode(
                session_jwt,
                os.getenv("JWT_SECRET"),
                algorithms=["HS256"]
            )
            
            # 2. Retrieve session from MongoDB
            session = self.db.find_one({"_id": ObjectId(payload["session_id"])})
            
            if not session:
                raise HTTPException(status_code=401, detail="Session not found")
            
            # 3. Check expiration
            if session["expires_at"] < datetime.utcnow():
                raise HTTPException(status_code=401, detail="Session expired")
            
            # 4. Update last activity
            self.db.update_one(
                {"_id": ObjectId(payload["session_id"])},
                {"$set": {"last_activity": datetime.utcnow()}}
            )
            
            return session
            
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail="JWT expired")
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail="Invalid JWT")

# FastAPI integration
app = FastAPI()
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
    # Store code_verifier in secure httpOnly cookie
    response = RedirectResponse(url=auth_info["auth_url"])
    response.set_cookie(
        "code_verifier",
        auth_info["code_verifier"],
        httponly=True,
        secure=True,
        samesite="strict"
    )
    return response

@app.get("/auth/callback")
async def auth_callback(code: str, state: str, request: Request):
    """Handle OAuth2 callback"""
    # 1. Retrieve stored PKCE verifier
    code_verifier = request.cookies.get("code_verifier")
    
    # 2. Exchange code for token
    tokens = await oauth2.exchange_code_for_token(code, code_verifier)
    
    # 3. Decode ID token to get user info
    id_token = jwt.decode(
        tokens["id_token"],
        options={"verify_signature": False}  # Already verified by auth server
    )
    
    # 4. Create session
    session_jwt = await oauth2.create_session(
        user_id=id_token["sub"],
        access_token=tokens["access_token"],
        refresh_token=tokens["refresh_token"],
        user_context={
            "email": id_token["email"],
            "name": id_token["name"],
            "groups": id_token.get("groups", [])
        }
    )
    
    # 5. Redirect to app with session JWT
    response = RedirectResponse(url=f"https://aegis.example.bank.com?session={session_jwt}")
    response.delete_cookie("code_verifier")
    
    return response
```

#### 2. MongoDB Session Storage
Configure MongoDB for secure, persistent session storage.

**Code Sample: MongoDB Schema & Indexes**
```python
# File: database/mongodb_setup.py

from pymongo import MongoClient, ASCENDING, DESCENDING
from pymongo.errors import OperationFailure
from datetime import datetime

class MongoDBSetup:
    def __init__(self, connection_uri: str):
        self.client = MongoClient(connection_uri)
        self.db = self.client.aegis
        
    def create_sessions_collection(self):
        """
        Create sessions collection with proper schema validation
        """
        try:
            # Drop existing (for development only)
            self.db.sessions.drop()
        except:
            pass
        
        # Create collection with validation
        self.db.create_collection(
            "sessions",
            validator={
                "$jsonSchema": {
                    "bsonType": "object",
                    "required": ["user_id", "access_token", "created_at", "expires_at"],
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
                                "groups": {
                                    "bsonType": "array",
                                    "items": {"bsonType": "string"}
                                }
                            }
                        },
                        "created_at": {"bsonType": "date"},
                        "expires_at": {"bsonType": "date"},
                        "last_activity": {"bsonType": "date"}
                    }
                }
            }
        )
        
        # Create indexes
        self.db.sessions.create_index("user_id", unique=False)
        self.db.sessions.create_index(
            "expires_at",
            expireAfterSeconds=0  # TTL: auto-delete expired sessions
        )
        self.db.sessions.create_index("last_activity", unique=False)
        
        print("✅ Sessions collection created with TTL index")
    
    def create_audit_logs_collection(self):
        """
        Create immutable audit logs collection
        """
        try:
            self.db.audit_logs.drop()
        except:
            pass
        
        self.db.create_collection(
            "audit_logs",
            validator={
                "$jsonSchema": {
                    "bsonType": "object",
                    "required": ["trace_id", "user_id", "action", "timestamp"],
                    "properties": {
                        "_id": {"bsonType": "objectId"},
                        "trace_id": {"bsonType": "string"},
                        "user_id": {"bsonType": "string"},
                        "action": {"bsonType": "string"},
                        "resource": {"bsonType": "string"},
                        "details": {"bsonType": "object"},
                        "timestamp": {"bsonType": "date"}
                    }
                }
            }
        )
        
        # Create indexes for search
        self.db.audit_logs.create_index(
            [("trace_id", ASCENDING)],
            unique=True  # One log per trace
        )
        self.db.audit_logs.create_index(
            [("user_id", ASCENDING), ("timestamp", DESCENDING)]
        )
        self.db.audit_logs.create_index("timestamp")
        
        print("✅ Audit logs collection created")

# Usage
setup = MongoDBSetup(os.getenv("MONGO_URI"))
setup.create_sessions_collection()
setup.create_audit_logs_collection()
```

#### 3. Auth Security Checklist
- [ ] OAuth2 server configured (Okta/Azure AD)
- [ ] PKCE enabled (prevents code interception)
- [ ] MongoDB sessions collection created + TTL index
- [ ] CORS configured (only allow aegis.example.bank.com)
- [ ] Rate limiting on auth endpoints (5 attempts/min)
- [ ] JWT secret stored in secure vault (not code)
- [ ] Session cookies: httpOnly, Secure, SameSite=Strict
- [ ] Password policy enforced by corporate IdP
- [ ] MFA enabled for advisors (TOTP)
- [ ] Session timeout: 8 hours inactive

**Exit Criteria for Week 2-3**:
- ✅ OAuth2 login flow working end-to-end
- ✅ Sessions persisted in MongoDB with TTL
- ✅ JWT validation passing
- ✅ RBAC user context propagated (groups, email)
- ✅ Security: PKCE + httpOnly cookies verified
- ✅ Performance: <200ms login to app load

---

## Week 4-5: Frontend & React UI

### Goal
Build React application with CopilotKit for agentic state management.

### Deliverables

#### 1. React Login Component
Secure login component with OAuth2 PKCE integration.

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

  // Check if returning from OAuth callback
  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const sessionJWT = params.get('session');
    
    if (sessionJWT) {
      // Store JWT in secure sessionStorage (not localStorage!)
      sessionStorage.setItem('session_jwt', sessionJWT);
      // Redirect to dashboard
      navigate('/dashboard');
    }
  }, [navigate]);

  const handleLogin = async () => {
    setLoading(true);
    setError(null);
    
    try {
      // 1. Call backend to get OAuth2 auth URL
      const response = await fetch('/api/auth/login', {
        method: 'GET',
        credentials: 'include'  // Send cookies
      });
      
      if (!response.ok) {
        throw new Error('Failed to initiate login');
      }
      
      const data = await response.json();
      
      // 2. Redirect to OAuth2 provider
      window.location.href = data.auth_url;
      
    } catch (err) {
      setError(err.message);
      setLoading(false);
    }
  };

  return (
    <div className="login-container">
      <div className="login-card">
        <div className="logo">
          <h1>🦅 Project Aegis</h1>
          <p>Regulation-Grade GenAI Banking Advisor</p>
        </div>
        
        <div className="login-form">
          <h2>Advisor Portal</h2>
          
          {error && (
            <div className="error-message" role="alert">
              ⚠️ {error}
            </div>
          )}
          
          <button 
            onClick={handleLogin}
            disabled={loading}
            className="login-button"
          >
            {loading ? 'Logging in...' : 'Login with Corporate Account'}
          </button>
          
          <p className="info-text">
            💡 Secure login via OAuth2 PKCE
            <br/>
            Your credentials are verified by Corporate Identity Provider
          </p>
        </div>
        
        <div className="footer">
          <p>Confidential - Internal Use Only</p>
          <p>For access issues, contact: aegis-support@example.bank.com</p>
        </div>
      </div>
    </div>
  );
}
```

#### 2. CopilotKit Integration
Integrate CopilotKit for agentic state management and real-time updates.

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

User Context:
- Name: ${user.name}
- Email: ${user.email}
- Groups: ${user.groups.join(', ')}

Your Role:
- Answer questions about banking products, regulations, and client strategies
- Always cite sources from institutional knowledge base
- Flag any policy violations to human reviewers
- Never provide advice outside your knowledge base

Safety Rules:
- If you're not sure, say "I don't know" 
- Only reference documents the user has access to
- Mark high-risk answers for human review
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
Main chat component for advisor queries.

**Code Sample: Chat Component**
```jsx
// File: src/components/ChatInterface.jsx

import React, { useState, useRef, useEffect } from 'react';
import { useCopilotContext } from '@copilotkit/react-core';
import './ChatInterface.css';

export function ChatInterface() {
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');
  const [loading, setLoading] = useState(false);
  const messagesEndRef = useRef(null);
  const { copilot } = useCopilotContext();

  // Auto-scroll to latest message
  useEffect(() => {
    messagesEndRef.current?.scrollIntoView({ behavior: 'smooth' });
  }, [messages]);

  const handleSendMessage = async (e) => {
    e.preventDefault();
    if (!input.trim()) return;

    // Add user message to UI
    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');
    setLoading(true);

    try {
      // Call copilot endpoint
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

      // Add assistant response
      const assistantMessage = {
        role: 'assistant',
        content: data.response,
        sources: data.sources,
        groundedness: data.groundedness,
        requiresApproval: data.requires_approval
      };

      setMessages(prev => [...prev, assistantMessage]);

      // If requires HITL, show approval dialog
      if (data.requires_approval) {
        showApprovalDialog(data.response, data.reason);
      }

    } catch (error) {
      console.error('Chat error:', error);
      setMessages(prev => [...prev, {
        role: 'system',
        content: '❌ Error processing query. Please try again.',
        isError: true
      }]);
    } finally {
      setLoading(false);
    }
  };

  const showApprovalDialog = (response, reason) => {
    // Show modal requiring advisor approval
    // This triggers HITL workflow
  };

  return (
    <div className="chat-interface">
      <div className="messages-container">
        {messages.map((msg, idx) => (
          <div key={idx} className={`message message-${msg.role}`}>
            <div className="message-content">
              {msg.content}
            </div>
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
            {msg.groundedness !== undefined && (
              <div className={`groundedness groundedness-${msg.groundedness > 0.8 ? 'good' : 'warning'}`}>
                📊 Groundedness: {(msg.groundedness * 100).toFixed(1)}%
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
          className="input-field"
        />
        <button type="submit" disabled={loading} className="send-button">
          {loading ? '⏳' : '➤'}
        </button>
      </form>
    </div>
  );
}
```

**Exit Criteria for Week 4-5**:
- ✅ Login flow: OAuth2 → Dashboard working
- ✅ CopilotKit integrated with real-time updates
- ✅ Chat interface rendering messages + sources
- ✅ Responsive design (works on desktop + tablet)
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
```

#### 2. System Prompt with RBAC Context
Dynamically include only accessible sources in LLM system prompt.

**Code Sample: Dynamic System Prompt**
```python
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
```

**Exit Criteria for Week 6-7**:
- ✅ Milvus RBAC filter expressions working
- ✅ Junior advisor cannot see "Private Wealth" documents
- ✅ Senior advisor can see all accessible docs
- ✅ Compliance officer can audit all retrievals (Phoenix logs)
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
import logging

class PIIScrubber:
    """
    Detects and redacts Personally Identifiable Information
    before it reaches the LLM
    """
    
    def __init__(self):
        self.analyzer = AnalyzerEngine()
        self.anonymizer = AnonymizerEngine()
        self.logger = logging.getLogger(__name__)
        
    async def scrub_input(self, text: str, trace_id: str) -> Dict:
        """
        Scan input for PII and redact
        
        Returns:
        {
            "original": "John Doe's SSN is 123-45-6789",
            "redacted": "John Doe's SSN is [SSN]",
            "detected_entities": [
                {"type": "PERSON", "text": "John Doe", "confidence": 0.95},
                {"type": "SSN", "text": "123-45-6789", "confidence": 0.99}
            ],
            "is_safe": True,
            "risk_level": "LOW"
        }
        """
        
        # 1. Analyze for PII
        entities = self.analyzer.analyze(
            text=text,
            language="en",
            entities=[
                "PERSON",           # Names
                "EMAIL_ADDRESS",    # Email addresses
                "PHONE_NUMBER",     # Phone numbers
                "SSN",             # Social Security Numbers
                "CREDIT_CARD",     # Credit card numbers
                "BANK_ACCOUNT",    # Bank account numbers
                "IBAN",            # International bank accounts
                "DATE_TIME",       # Dates (sometimes sensitive)
                "URL",             # URLs with tracking
                "IP_ADDRESS"       # IP addresses
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
        
        # 3. Determine risk level
        high_risk_entities = ["SSN", "CREDIT_CARD", "BANK_ACCOUNT", "IBAN"]
        has_high_risk = any(
            e.entity_type in high_risk_entities for e in entities
        )
        
        risk_level = "CRITICAL" if has_high_risk else "LOW"
        is_safe = not has_high_risk  # Reject if high-risk PII detected
        
        # 4. Log to Phoenix for audit
        await self._log_scrubbing_event(
            trace_id=trace_id,
            original_length=len(text),
            detected_count=len(entities),
            entity_types=[e.entity_type for e in entities],
            risk_level=risk_level
        )
        
        # 5. Reject if critical PII found
        if not is_safe:
            self.logger.warning(
                f"CRITICAL PII DETECTED in trace {trace_id}: {risk_level}"
            )
            raise ValueError(
                f"Cannot process request: {risk_level} PII detected. "
                "Please remove sensitive information and try again."
            )
        
        return {
            "original": text,
            "redacted": redacted_text,
            "detected_entities": [
                {
                    "type": e.entity_type,
                    "text": e.text,
                    "confidence": e.score
                }
                for e in entities
            ],
            "is_safe": is_safe,
            "risk_level": risk_level
        }
    
    async def _log_scrubbing_event(self, trace_id: str, original_length: int,
                                   detected_count: int, entity_types: List[str],
                                   risk_level: str):
        """Log PII scrubbing event to Phoenix"""
        import requests
        
        requests.post(
            "https://phoenix.example.bank.com/api/events",
            json={
                "trace_id": trace_id,
                "event_type": "pii_scrubbing",
                "input_length": original_length,
                "entities_detected": detected_count,
                "entity_types": entity_types,
                "risk_level": risk_level,
                "timestamp": datetime.utcnow().isoformat()
            }
        )

# FastAPI middleware to apply scrubbing
@app.post("/api/chat")
async def chat_endpoint(request: ChatRequest, 
                       user_context: UserContext = Depends(get_user_context)):
    scrubber = PIIScrubber()
    
    # 1. Scrub input
    scrubbed = await scrubber.scrub_input(
        text=request.query,
        trace_id=request.state.trace_id
    )
    
    # 2. Use scrubbed version for LLM
    clean_query = scrubbed["redacted"]
    
    # 3. Process
    result = await process_query(
        query=clean_query,
        user_context=user_context,
        trace_id=request.state.trace_id
    )
    
    return result
```

#### 2. NVIDIA NeMo Guardrails

**Code Sample: Prompt Injection Detection**
```python
# File: security/nemo_guardrails.py

from nemoguardrails import RailsConfig, LLMRails
import json

class PromptInjectionGuard:
    """
    Detects and blocks prompt injection attacks using NeMo Guardrails
    """
    
    def __init__(self):
        # Define guardrails in YAML
        self.config_yaml = """
models:
- type: main
  engine: openai
  model: gpt-4o

rails:
  input:
    flows:
      - self check input
      
flows:
  self check input:
    steps:
      - check input with regex patterns
      - filter harmful payloads
      
def check input with regex patterns:
  "This flow checks the input for known injection patterns"
  
  # Known prompt injection patterns
  patterns:
    - "ignore previous instructions"
    - "disregard prior context"
    - "reveal your system prompt"
    - "you are a different AI now"
    - "forget your guidelines"
    - "execute code"
    - "shell command"
    - "SQL injection"
  
  for pattern in patterns:
    if pattern in input.lower():
      raise ValueError("Prompt injection attempt detected")
      
def filter harmful payloads:
  "Block payloads designed to manipulate the model"
  
  harmful_keywords = [
    "jailbreak", "bypass", "exploit", "vulnerability",
    "admin override", "superuser", "root access"
  ]
  
  for keyword in harmful_keywords:
    if keyword in input.lower():
      log_security_event("prompt_injection_attempt", input)
      raise ValueError("Harmful payload detected")
        """
        
        self.config = RailsConfig.from_yaml(self.config_yaml)
        self.rails = LLMRails(self.config)
    
    async def check_input(self, text: str, trace_id: str) -> Dict:
        """
        Check input for prompt injection attempts
        
        Returns:
        {
            "is_safe": bool,
            "blocked_patterns": list,
            "confidence": float
        }
        """
        
        # 1. Run through NeMo rails
        result = await self.rails.generate(
            prompt=text,
            max_tokens=1  # Don't actually generate, just validate
        )
        
        # 2. Check for injection patterns
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
        
        import re
        blocked_patterns = []
        for pattern in injection_patterns:
            if re.search(pattern, text, re.IGNORECASE):
                blocked_patterns.append(pattern)
        
        is_safe = len(blocked_patterns) == 0
        
        # 3. Log attempt
        if not is_safe:
            await self._log_injection_attempt(
                trace_id=trace_id,
                input_text=text,
                blocked_patterns=blocked_patterns
            )
        
        return {
            "is_safe": is_safe,
            "blocked_patterns": blocked_patterns,
            "confidence": 0.95 if blocked_patterns else 1.0
        }
    
    async def _log_injection_attempt(self, trace_id: str, 
                                     input_text: str, 
                                     blocked_patterns: List[str]):
        """Log to Phoenix for security monitoring"""
        import requests
        
        requests.post(
            "https://phoenix.example.bank.com/api/security-events",
            json={
                "trace_id": trace_id,
                "event_type": "prompt_injection_blocked",
                "input_text": input_text[:200],  # First 200 chars
                "matched_patterns": blocked_patterns,
                "timestamp": datetime.utcnow().isoformat(),
                "severity": "HIGH"
            }
        )

# Middleware usage
guard = PromptInjectionGuard()

@app.post("/api/chat")
async def chat_endpoint(request: ChatRequest):
    # 1. Check for prompt injection
    guard_result = await guard.check_input(
        text=request.query,
        trace_id=request.state.trace_id
    )
    
    if not guard_result["is_safe"]:
        return {
            "error": "Request contains potentially malicious patterns",
            "blocked_patterns": guard_result["blocked_patterns"]
        }
    
    # 2. Proceed with normal processing
    result = await process_query(request.query)
    return result
```

**Exit Criteria for Week 8-9**:
- ✅ PII detected in 100% of test cases (SSN, CC, bank account)
- ✅ Input rejection for high-risk PII: 100% block rate
- ✅ Prompt injection patterns blocked: 95%+ detection rate
- ✅ NeMo guardrails integrated with <100ms overhead
- ✅ Phoenix logs all security events (audit trail)
- ✅ Zero false positives on legitimate business text

---

## Week 10-11: LangGraph & RAG Integration

### Goal
Implement orchestration engine and RAG pipeline for knowledge retrieval.

### Deliverables - CONTINUES IN NEXT SECTION...

**Current Progress**: Weeks 1-9 Complete (85% of Phase 1)  
**Remaining**: Weeks 10-11 LangGraph + RAG + Exit Criteria

---

**PHASE 1 ALPHA DOCUMENT - SECTION 1 COMPLETE**
**Total Content: Weeks 1-9 with full code samples**