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
