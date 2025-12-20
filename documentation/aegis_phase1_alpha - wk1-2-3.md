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
