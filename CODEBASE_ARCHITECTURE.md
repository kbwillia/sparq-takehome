# Codebase Architecture - AI Knowledge Assistant

This document outlines the complete codebase structure for the AI Knowledge Assistant, showing how authentication, authorization, and RAG work together in a production system.

## What is Redis? (And Why We're Not Using It)

**Redis** is an in-memory data store that's commonly used for:
- Caching frequently accessed data
- Session storage
- Rate limiting counters
- Message queues

**However**, in your RAG and Golf projects, you used a **simpler, file-based caching approach**:
- **JSON file cache** (`embedding_cache.json`) stored on disk
- **In-memory dictionaries** (`self.embedding_cache = {}`) for fast lookups
- **Automatic persistence** - cache is saved to disk after each update

This approach is:
- ✅ **Simpler** - No need for a separate Redis server
- ✅ **Sufficient** - Works great for embedding caching
- ✅ **Persistent** - Cache survives server restarts
- ✅ **Cost-effective** - No additional infrastructure needed

We'll use this same approach in the architecture below.

---

## Project Structure

```
ai-knowledge-assistant/
├── frontend/                    # React frontend (JavaScript)
│   ├── src/
│   │   ├── components/
│   │   │   ├── ChatInterface.jsx      # Shared chat UI (all roles)
│   │   │   ├── Dashboard.jsx          # Role-based dashboard
│   │   │   └── RoleBasedNav.jsx       # Different nav per role
│   │   ├── hooks/
│   │   │   ├── useAuth.js             # Auth context
│   │   │   └── useQuery.js            # Query submission
│   │   ├── pages/
│   │   │   ├── login.jsx              # SSO login
│   │   │   ├── chat.jsx               # Main chat (all roles)
│   │   │   └── dashboard.jsx          # Role-specific dashboard
│   │   └── utils/
│   │       └── roleConfig.js          # Role-based UI config
│   └── package.json
│
├── backend/
│   ├── api_gateway/             # FastAPI entry point
│   │   ├── middleware/
│   │   │   ├── auth.py          # JWT validation
│   │   │   ├── rbac.py          # Permission checks
│   │   │   └── rate_limit.py
│   │   └── routes/
│   │       └── query.py
│   │
│   ├── auth_service/            # Authentication
│   │   ├── sso/
│   │   │   ├── saml.py
│   │   │   └── oauth.py
│   │   ├── jwt/
│   │   │   └── token_service.py
│   │   └── roles/
│   │       └── permissions.py
│   │
│   ├── query_service/           # Main query processing
│   │   ├── router/
│   │   │   └── query_router.py
│   │   ├── rag/
│   │   │   ├── retrieval.py
│   │   │   ├── prompt_builder.py
│   │   │   └── llm_orchestrator.py
│   │   └── structured/
│   │       └── sql_generator.py
│   │
│   ├── data_service/            # Data access
│   │   ├── vector/
│   │   │   └── vector_db.py
│   │   ├── relational/
│   │   │   └── database.py
│   │   └── cache/
│   │       └── json_cache.py    # JSON file-based cache (like your RAG project)
│   │
│   ├── main.py                  # FastAPI app entry point
│   └── requirements.txt
│
└── infrastructure/
    └── docker-compose.yml
```

---

## Key Architecture Principles

### 1. Single Frontend, Role-Based UI
- **One React app** serves all users (students, teachers, counselors)
- **UI adapts** based on user role (different features, same interface)
- **Permissions enforced on backend** - frontend just shows/hides features

### 2. Authentication Happens BEFORE RAG
```
Request Flow:
1. User sends query → Frontend
2. Frontend adds JWT token → API Gateway
3. API Gateway validates token → Auth Middleware
4. Check permissions → RBAC Middleware
5. Route query → Query Router
6. Process query → RAG or SQL Service
7. Return response → Frontend
```

**Authentication is "higher level"** - it happens at the API gateway, before any query processing. This ensures:
- Unauthorized users never reach expensive RAG operations
- Security is centralized
- Performance is optimized (fail fast)

### 3. RAG is Optional
- **Simple data queries** (e.g., "What's my GPA?") → Use SQL, not RAG
- **Complex knowledge questions** (e.g., "What are graduation requirements?") → Use RAG
- **Query Router** decides which path to take

---

## Code Examples

### 1. Frontend: Single App with Role-Based UI

```javascript
// frontend/src/hooks/useAuth.js
import { createContext, useContext, useState, useEffect } from 'react';
import { useNavigate } from 'react-router-dom';

const AuthContext = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [isLoading, setIsLoading] = useState(true);
  const navigate = useNavigate();

  useEffect(() => {
    const token = localStorage.getItem('auth_token');
    if (token) {
      validateToken(token);
    } else {
      setIsLoading(false);
    }
  }, []);

  async function validateToken(token) {
    try {
      const response = await fetch('/api/auth/validate', {
        headers: { 'Authorization': `Bearer ${token}` }
      });

      if (response.ok) {
        const userData = await response.json();
        setUser(userData);
      } else {
        localStorage.removeItem('auth_token');
      }
    } catch (error) {
      console.error('Token validation failed:', error);
    } finally {
      setIsLoading(false);
    }
  }

  async function login() {
    window.location.href = '/api/auth/sso/login';
  }

  function logout() {
    localStorage.removeItem('auth_token');
    setUser(null);
    navigate('/login');
  }

  return (
    <AuthContext.Provider value={{ user, login, logout, isLoading }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  const context = useContext(AuthContext);
  if (!context) throw new Error('useAuth must be used within AuthProvider');
  return context;
}
```

```javascript
// frontend/src/components/ChatInterface.jsx
import { useAuth } from '../hooks/useAuth';
import { useState } from 'react';

export function ChatInterface() {
  const { user } = useAuth();
  const [messages, setMessages] = useState([]);
  const [input, setInput] = useState('');

  async function handleSubmit(e) {
    e.preventDefault();

    const userMessage = { role: 'user', content: input };
    setMessages(prev => [...prev, userMessage]);
    setInput('');

    // Send to backend - auth token automatically included
    const response = await fetch('/api/v1/query', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${localStorage.getItem('auth_token')}`
      },
      body: JSON.stringify({
        query: input,
        user_role: user?.role,
        user_id: user?.id
      })
    });

    const data = await response.json();
    setMessages(prev => [...prev, { role: 'assistant', content: data.response }]);
  }

  // UI is the same for all roles - permissions enforced on backend
  return (
    <div className="chat-container">
      <div className="messages">
        {messages.map((msg, i) => (
          <div key={i} className={`message ${msg.role}`}>
            {msg.content}
          </div>
        ))}
      </div>
      <form onSubmit={handleSubmit}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          placeholder="Ask me anything..."
        />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

### 2. Backend: Authentication Middleware (Before RAG)

```python
# backend/api_gateway/middleware/auth.py
from functools import wraps
from flask import request, jsonify
import jwt
import os

JWT_SECRET = os.getenv('JWT_SECRET')

def authenticate_token(f):
    """Decorator to verify JWT token on protected routes"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        # Get token from Authorization header
        auth_header = request.headers.get('Authorization')

        if not auth_header:
            return jsonify({'error': 'No token provided'}), 401

        try:
            # Extract token (format: "Bearer <token>")
            token = auth_header.split(' ')[1]

            # Verify and decode JWT
            decoded = jwt.decode(token, JWT_SECRET, algorithms=['HS256'])

            # Attach user info to request object
            request.user = {
                'id': decoded['user_id'],
                'role': decoded['role'],
                'permissions': decoded['permissions']
            }

        except jwt.ExpiredSignatureError:
            return jsonify({'error': 'Token expired'}), 403
        except jwt.InvalidTokenError:
            return jsonify({'error': 'Invalid token'}), 403

        return f(*args, **kwargs)

    return decorated_function
```

```python
# backend/api_gateway/middleware/rbac.py
from functools import wraps
from flask import request, jsonify
import re

# Permission matrix - what each role can do
ROLE_PERMISSIONS = {
    'student': ['read:own_data', 'read:public_content'],
    'teacher': [
        'read:own_data',
        'read:public_content',
        'read:aggregated_class_data',
        'read:curriculum'
    ],
    'counselor': [
        'read:own_data',
        'read:public_content',
        'read:individual_student_data',
        'read:interventions',
        'read:caseload'
    ]
}

def check_permission(required_permission):
    """Decorator to check if user has required permission"""
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not hasattr(request, 'user'):
                return jsonify({'error': 'Not authenticated'}), 401

            user_role = request.user['role']
            user_permissions = ROLE_PERMISSIONS.get(user_role, [])

            if required_permission not in user_permissions:
                return jsonify({
                    'error': 'Insufficient permissions',
                    'required': required_permission,
                    'user_permissions': user_permissions
                }), 403

            return f(*args, **kwargs)
        return decorated_function
    return decorator

def check_query_permissions(f):
    """Analyze query to determine what permissions it needs"""
    @wraps(f)
    def decorated_function(*args, **kwargs):
        query = request.json.get('query', '').lower()

        # Check if query asks for other students' data
        other_student_patterns = [
            r'show.*all students',
            r'list.*students',
            r'everyone.*grade',
            r'all.*gpa'
        ]

        needs_other_student_data = any(
            re.search(pattern, query) for pattern in other_student_patterns
        )

        if needs_other_student_data and request.user['role'] == 'student':
            return jsonify({
                'error': 'Students can only access their own data'
            }), 403

        # Check if query needs own data
        own_data_patterns = ['my gpa', 'my grade', 'my attendance', 'my schedule']
        needs_own_data = any(pattern in query for pattern in own_data_patterns)

        if needs_own_data and request.user['role'] == 'student':
            request.needs_own_data_only = True

        return f(*args, **kwargs)

    return decorated_function
```

### 3. Query Route (FastAPI Example)

```python
# backend/api_gateway/routes/query.py
from fastapi import APIRouter, Depends, HTTPException, Request
from pydantic import BaseModel
from ..middleware.auth import authenticate_token
from ..middleware.rbac import check_query_permissions
from ...query_service.router.query_router import QueryService

router = APIRouter()
query_service = QueryService()

class QueryRequest(BaseModel):
    query: str
    user_role: str = None
    user_id: str = None

class QueryResponse(BaseModel):
    response: str
    sources: list = []
    confidence: float = 0.0

@router.post("/query", response_model=QueryResponse)
async def process_query(
    request: QueryRequest,
    current_user: dict = Depends(authenticate_token)
):
    """
    Process a user query. Authentication happens FIRST (before RAG).
    """
    try:
        # Check query permissions
        await check_query_permissions(request.query, current_user)

        # Process query with user context
        result = await query_service.process_query(
            query=request.query,
            user_id=current_user['id'],
            user_role=current_user['role'],
            permissions=current_user['permissions']
        )

        return QueryResponse(
            response=result['text'],
            sources=result['sources'],
            confidence=result['confidence']
        )

    except PermissionError as e:
        raise HTTPException(status_code=403, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail="Query processing failed")
```

### 4. Query Service (Decides RAG vs SQL)

```python
# backend/query_service/router/query_router.py
from ..rag.retrieval import RAGService
from ..structured.sql_generator import SQLService

class QueryService:
    def __init__(self):
        self.rag_service = RAGService()
        self.sql_service = SQLService()

    async def process_query(self, query, user_id, user_role, permissions):
        """
        Main entry point - decides whether to use RAG or SQL
        """
        # Classify query type
        query_type = self._classify_query(query)

        if query_type == 'structured_data':
            # Needs student data - use SQL, not RAG
            return await self.sql_service.query(
                query=query,
                user_id=user_id,
                user_role=user_role,
                permissions=permissions
            )
        else:
            # General knowledge - use RAG
            return await self.rag_service.query(
                query=query,
                user_role=user_role,
                permissions=permissions
            )

    def _classify_query(self, query):
        """
        Simple classification - could use ML model for better accuracy
        """
        query_lower = query.lower()

        # Keywords that indicate structured data lookup
        structured_keywords = [
            'my gpa', 'my grade', 'my attendance',
            'my schedule', 'my credits', 'my classes'
        ]

        needs_data = any(keyword in query_lower for keyword in structured_keywords)

        return 'structured_data' if needs_data else 'rag'
```

### 5. RAG Service (Only if Needed)

```python
# backend/query_service/rag/retrieval.py
from ...data_service.vector.vector_db import VectorDB
from ...data_service.cache.json_cache import JSONCache
from .llm_orchestrator import LLMOrchestrator
from .prompt_builder import PromptBuilder

class RAGService:
    def __init__(self):
        self.vector_db = VectorDB()
        self.llm = LLMOrchestrator()
        self.prompt_builder = PromptBuilder()
        # Use JSON file-based cache (like your RAG project)
        self.cache = JSONCache(cache_dir="embedding_cache")

    async def query(self, query, user_role, permissions):
        """
        RAG pipeline: Embed → Search → Prompt → LLM
        """
        # 1. Embed query (with caching)
        query_embedding = await self._embed_query(query)

        # 2. Vector search (filtered by permissions)
        allowed_access_levels = self._get_allowed_access_levels(permissions)

        relevant_chunks = await self.vector_db.search(
            embedding=query_embedding,
            top_k=5,
            filter={'access_level': {'$in': allowed_access_levels}}
        )

        # 3. Build prompt with role-specific template
        prompt = self.prompt_builder.build(
            query=query,
            user_role=user_role,
            context=relevant_chunks
        )

        # 4. Call LLM
        response = await self.llm.generate(prompt)

        return {
            'text': response['text'],
            'sources': [chunk['metadata'] for chunk in relevant_chunks],
            'confidence': response['confidence']
        }

    async def _embed_query(self, query):
        """Generate embedding for query (with caching)"""
        # Check cache first (like your RAG project)
        cached_embedding = self.cache.get(query)
        if cached_embedding:
            return cached_embedding

        # Generate new embedding
        import openai
        response = openai.embeddings.create(
            model="text-embedding-ada-002",
            input=query
        )
        embedding = response.data[0].embedding

        # Cache the result
        self.cache.set(query, embedding)

        return embedding

    def _get_allowed_access_levels(self, permissions):
        """Determine which document access levels user can see"""
        levels = ['public']
        if 'read:curriculum' in permissions:
            levels.append('teacher')
        if 'read:interventions' in permissions:
            levels.append('counselor')
        return levels
```

### 6. JSON File-Based Cache (Like Your RAG Project)

```python
# backend/data_service/cache/json_cache.py
import json
import os
from pathlib import Path
from typing import Dict, List, Optional
import hashlib

class JSONCache:
    """
    JSON file-based cache for embeddings and query responses.
    Similar to your RAG project's embedding cache.
    """
    def __init__(self, cache_dir: str = "embedding_cache", cache_file: str = "cache.json"):
        """
        Initialize cache with directory and file.

        Args:
            cache_dir: Directory to store cache files
            cache_file: Name of cache file
        """
        self.cache_dir = Path(cache_dir)
        self.cache_dir.mkdir(exist_ok=True)
        self.cache_file = self.cache_dir / cache_file
        self.cache = self._load_cache()

    def _load_cache(self) -> Dict:
        """Load cache from disk."""
        if self.cache_file.exists():
            try:
                with open(self.cache_file, 'r') as f:
                    cache = json.load(f)
                    print(f"[Cache] Loaded {len(cache)} entries from {self.cache_file}")
                    return cache
            except (json.JSONDecodeError, IOError) as e:
                print(f"[Cache] Error loading cache: {e}. Creating new cache.")
                return {}
        print(f"[Cache] No cache found. Creating new cache at {self.cache_file}")
        return {}

    def _save_cache(self):
        """Save cache to disk."""
        try:
            with open(self.cache_file, 'w') as f:
                json.dump(self.cache, f, indent=2)
        except IOError as e:
            print(f"[Cache] Error saving cache: {e}")

    def _get_key(self, text: str) -> str:
        """Generate cache key from text (hash for long texts)."""
        if len(text) > 100:
            # Use hash for long texts to keep keys manageable
            return hashlib.md5(text.encode()).hexdigest()
        return text

    def get(self, key: str) -> Optional[List[float]]:
        """
        Get cached value.

        Args:
            key: Cache key (query text or hash)

        Returns:
            Cached value or None
        """
        cache_key = self._get_key(key)
        if cache_key in self.cache:
            print(f"[Cache] Cache hit for: {key[:50]}...")
            return self.cache[cache_key]
        return None

    def set(self, key: str, value: List[float]):
        """
        Set cache value.

        Args:
            key: Cache key
            value: Value to cache (embedding vector)
        """
        cache_key = self._get_key(key)
        self.cache[cache_key] = value
        self._save_cache()
        print(f"[Cache] Cached: {key[:50]}...")

    def clear(self):
        """Clear all cache entries."""
        self.cache = {}
        self._save_cache()
        print("[Cache] Cache cleared")

    def size(self) -> int:
        """Get number of cache entries."""
        return len(self.cache)

    def get_stats(self) -> Dict:
        """Get cache statistics."""
        return {
            'size': len(self.cache),
            'cache_file': str(self.cache_file),
            'cache_dir': str(self.cache_dir)
        }
```

### 7. SQL Service (For Structured Data)

```python
# backend/query_service/structured/sql_generator.py
from ...data_service.relational.database import Database

class SQLService:
    def __init__(self):
        self.db = Database()

    async def query(self, query, user_id, user_role, permissions):
        """
        Handle queries that need structured data (grades, attendance, etc.)
        """
        # Simple text-to-SQL (in production, use more sophisticated approach)
        sql_query = self._generate_sql(query, user_id, user_role)

        # Execute with Row-Level Security (RLS)
        result = await self.db.execute(sql_query, user_id=user_id, role=user_role)

        # Format response
        return {
            'text': self._format_response(query, result),
            'sources': ['Student Information System'],
            'confidence': 1.0  # Structured data is always accurate
        }

    def _generate_sql(self, query, user_id, user_role):
        """Convert natural language to SQL"""
        query_lower = query.lower()

        if 'gpa' in query_lower or 'grade point average' in query_lower:
            return f"""
                SELECT unweighted_gpa, weighted_gpa
                FROM student_grades
                WHERE student_id = '{user_id}'
            """
        elif 'attendance' in query_lower:
            return f"""
                SELECT COUNT(*) as absent_days
                FROM attendance
                WHERE student_id = '{user_id}'
                AND status = 'absent'
            """
        else:
            # Default: return basic student info
            return f"SELECT * FROM students WHERE id = '{user_id}'"

    def _format_response(self, query, result):
        """Format SQL result into natural language"""
        if 'gpa' in query.lower():
            gpa_data = result[0]
            return f"Your GPA is {gpa_data['unweighted_gpa']} (unweighted) and {gpa_data['weighted_gpa']} (weighted)."
        elif 'attendance' in query.lower():
            return f"You have {result[0]['absent_days']} absent days this semester."
        else:
            return f"Here's your information: {result}"
```

### 8. Main Application Entry Point

```python
# backend/main.py
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from api_gateway.routes import query

app = FastAPI(title="AI Knowledge Assistant API")

# CORS middleware for frontend
app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],  # React dev server
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Include routes
app.include_router(query.router, prefix="/api/v1", tags=["query"])

@app.get("/")
def root():
    return {"message": "AI Knowledge Assistant API"}

@app.get("/health")
def health_check():
    return {"status": "healthy"}

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

## Complete Request Flow Example

```
1. User opens app → Single React app loads
   └─> Checks localStorage for token
   └─> If no token → Redirect to /login

2. User clicks "Login with School Account"
   └─> Redirects to SSO provider (Google Workspace)
   └─> User authenticates
   └─> SSO callback with code
   └─> Backend exchanges code for user info
   └─> Backend generates JWT with role/permissions
   └─> Token stored in localStorage

3. User types query: "What's my GPA?"
   └─> Frontend sends POST /api/v1/query
   └─> Headers: { Authorization: "Bearer <token>" }

4. API Gateway receives request
   └─> auth.py middleware: Validates JWT
       ├─> Extracts: { user_id, role: "student", permissions: [...] }
       └─> Attaches to request.user

   └─> rbac.py middleware: Checks permissions
       ├─> Query needs: "read:own_data"
       ├─> User has: ["read:own_data", "read:public_content"] ✅
       └─> Continues

5. Query Router classifies query
   └─> "my GPA" → structured_data (not RAG)
   └─> Routes to SQL service (not RAG pipeline)

6. SQL Service
   └─> Generates SQL: SELECT gpa FROM grades WHERE student_id = ?
   └─> Executes with RLS (Row-Level Security)
   └─> Returns: { gpa: 3.45 }

7. Response formatted and returned
   └─> Frontend displays: "Your GPA is 3.45"
```

---

## Key Differences from TypeScript Version

1. **Python decorators**: `@decorator` syntax instead of middleware functions
2. **FastAPI**: Automatic API docs, request validation with Pydantic
3. **JSON file cache**: Instead of Redis, using file-based cache like your RAG project
4. **No compilation**: Python is interpreted, not compiled

---

## Why This Architecture?

### Single Frontend
- ✅ **Maintainability**: One codebase instead of three
- ✅ **Consistency**: Same UI patterns across all roles
- ✅ **Easier updates**: Deploy once, all users get updates

### Authentication Before RAG
- ✅ **Security**: Unauthorized users never reach expensive operations
- ✅ **Performance**: Fail fast on invalid tokens
- ✅ **Centralized**: All auth logic in one place

### JSON File Cache (Not Redis)
- ✅ **Simplicity**: No additional infrastructure
- ✅ **Sufficient**: Works great for embedding caching
- ✅ **Persistent**: Cache survives server restarts
- ✅ **Cost-effective**: No Redis server costs

### RAG is Optional
- ✅ **Efficiency**: Simple queries use fast SQL
- ✅ **Cost**: Only use expensive LLM calls when needed
- ✅ **Accuracy**: Structured data is always accurate

---

## Summary

This architecture provides:
- **Single React frontend** with role-based UI
- **Authentication at API gateway** (before RAG)
- **JSON file-based caching** (like your RAG project)
- **Smart routing** (RAG vs SQL based on query type)
- **FERPA-compliant** access controls
- **Scalable** microservices design

The key insight: **Authentication is "higher level" than RAG** - it happens at the API gateway before any query processing, ensuring security and performance.

