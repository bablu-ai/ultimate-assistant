
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
