# Golf Project Components Related to SPARQ Takehome

This document explains how various components from the Golf card game project relate to and can inform the SPARQ AI Knowledge Assistant architecture.

---

## Overview

The Golf project is a full-stack web application featuring AI-powered chatbots that interact with players during a card game. While the domain is different (gaming vs. education), several architectural patterns and implementations are directly applicable to the SPARQ takehome project.

---

## 1. Embedding Models & Caching (`embedding_models.py`)

### What It Does in Golf Project

The `EmbeddingModel` class provides a reusable embedding service with:
- Multiple model support (BGE, E5, Ollama models)
- **JSON file-based caching** for embeddings
- Batch processing capabilities
- Device management (CPU/GPU)

### Relation to SPARQ Takehome

**Direct Application**: This is exactly what the SPARQ architecture needs for the **Embedding Generation** component in the Data Processing Pipeline.

```python
# From Golf project - embedding_models.py
class EmbeddingModel:
    def __init__(self, model_name='bge-large', cache_dir="embedding_cache"):
        self.cache_file = self.cache_dir / f"{model_name}_cache.json"
        self.embedding_cache = self._load_cache()  # Load from JSON

    def get_embedding(self, text: str) -> np.ndarray:
        # Check cache first
        if text in self.embedding_cache:
            return np.array(self.embedding_cache[text])

        # Generate new embedding
        embedding = self.model.encode(text)

        # Cache the result
        self.embedding_cache[text] = embedding.tolist()
        self._save_cache()  # Save to JSON file
        return embedding
```

### Key Insights for SPARQ

1. **JSON File-Based Caching**: The SPARQ architecture mentions "JSON file-based caching (like your RAG project)" - this is the implementation pattern.

2. **Cache Management**:
   - Load cache on initialization
   - Check cache before generating embeddings
   - Save cache after new embeddings
   - Reduces API costs and latency

3. **Model Flexibility**: Supports multiple embedding models, allowing SPARQ to:
   - Start with cost-effective models (e5-base)
   - Upgrade to better models (bge-large) as needed
   - Switch models without code changes

### Implementation Example for SPARQ

```python
# SPARQ could use this pattern for document chunking:
embedding_model = EmbeddingModel(model_name='bge-base', cache_dir='sparq_embeddings')

# When ingesting documents:
chunks = chunk_document(pdf_content)
for chunk in chunks:
    embedding = embedding_model.get_embedding(chunk['text'])
    # Store in vector database with metadata
    vector_db.upsert(embedding, metadata=chunk['metadata'])
```

---

## 2. Context Management & Prompt Building (`chatbot.py`)

### What It Does in Golf Project

The `GolfChatbot` class demonstrates sophisticated context management:
- Formats game state into LLM-readable prompts
- Manages conversation history per game session
- Builds dynamic prompts with multiple context layers
- Handles personality, emotional state, and difficulty settings

### Relation to SPARQ Takehome

**Direct Application**: This pattern maps to the **Prompt Engineering Service** and **Context Management** in SPARQ's AI Orchestration Layer.

```python
# From Golf project - chatbot.py
def format_game_state_for_prompt(self, game_state: Dict[str, Any]) -> str:
    """Format the current game state into a readable prompt for the LLM"""
    # Extract key information
    current_player = game_state.get('current_player', 0)
    players = game_state.get('players', [])
    # ... format into structured text

    game_state_text = f"""
        Current Game State:
        - Round: {round_num}
        - Players: {player_info}
        - Discard pile: {discard_str}
    """
    return game_state_text

def generate_response(self, conversation_history, game_state, ai_bot_id):
    # Build context layers
    context = system_prompt + "\n\n"
    context += "Game Rules:\n" + self.game_rules + "\n\n"
    context += self.format_game_state_for_prompt(game_state) + "\n\n"

    # Add conversation history
    if conversation_history:
        context += "Recent conversation:\n"
        for msg in conversation_history[-3:]:
            context += f"{msg['sender']}: {msg['content']}\n"

    # Add personality/role context
    context += self.personality_prompt_builder(ai_bot_id)
    context += self.emotional_state_prompt_builder(ai_bot_id)

    # Call LLM
    response = call_cerebras_llm(prompt=context, ...)
    return response
```

### Key Insights for SPARQ

1. **Layered Context Building**:
   - System prompt (role definition)
   - Domain knowledge (game rules → curriculum documents)
   - Current state (game state → student context)
   - Conversation history
   - Role-specific prompts (personality → student/teacher/counselor)

2. **Conversation History Management**:
   ```python
   # Golf project pattern
   self.conversation_history = {}  # {game_id: [messages]}

   # SPARQ equivalent
   conversation_history = {}  # {user_id: [messages]}
   ```

3. **Dynamic Prompt Construction**: The Golf project shows how to:
   - Combine multiple context sources
   - Limit history to last N messages (prevents token overflow)
   - Format structured data for LLM consumption

### Implementation Example for SPARQ

```python
# SPARQ Prompt Engineering Service could use this pattern:
class PromptEngineeringService:
    def build_prompt(self, query, user_role, student_context, retrieved_docs):
        context = self.get_role_system_prompt(user_role) + "\n\n"

        # Add student context (like game state)
        if student_context:
            context += self.format_student_context(student_context) + "\n\n"

        # Add retrieved documents (like game rules)
        if retrieved_docs:
            context += "Relevant Information:\n"
            for doc in retrieved_docs:
                context += f"- {doc['content']}\n"
            context += "\n"

        # Add conversation history
        if conversation_history:
            context += "Recent conversation:\n"
            for msg in conversation_history[-3:]:
                context += f"{msg['sender']}: {msg['content']}\n"

        context += f"\nUser Query: {query}\n"
        return context
```

---

## 3. Conversation History Management

### What It Does in Golf Project

The Golf project maintains conversation history per game session:

```python
# From chatbot.py
self.conversation_history = {}  # {game_id: [messages]}

def add_message_to_history(self, sender: str, content: str, game_id: str):
    message = {
        'sender': sender,
        'content': content,
        'timestamp': time.time()
    }
    if game_id not in self.conversation_history:
        self.conversation_history[game_id] = []
    self.conversation_history[game_id].append(message)
```

### Relation to SPARQ Takehome

**Direct Application**: SPARQ needs similar conversation history management for:
- Maintaining context across multiple queries
- Providing continuity in student/counselor interactions
- Audit logging (FERPA compliance)

### Key Differences

| Golf Project | SPARQ Project |
|-------------|---------------|
| Per-game conversation history | Per-user or per-session history |
| Game state context | Student data context |
| Bot personalities | Role-based access (student/teacher/counselor) |

### Implementation Pattern for SPARQ

```python
# SPARQ could use similar structure:
class ConversationManager:
    def __init__(self):
        self.conversation_history = {}  # {user_id: [messages]}
        self.max_history = 10  # Keep last 10 messages

    def add_message(self, user_id: str, role: str, content: str, metadata: dict = None):
        message = {
            'role': role,  # 'user', 'assistant', 'system'
            'content': content,
            'timestamp': time.time(),
            'metadata': metadata  # For audit logging
        }
        if user_id not in self.conversation_history:
            self.conversation_history[user_id] = []
        self.conversation_history[user_id].append(message)

        # Trim to max_history
        if len(self.conversation_history[user_id]) > self.max_history:
            self.conversation_history[user_id] = \
                self.conversation_history[user_id][-self.max_history:]

    def get_recent_history(self, user_id: str, n: int = 5):
        history = self.conversation_history.get(user_id, [])
        return history[-n:] if len(history) > n else history
```

---

## 4. LLM Orchestration & Response Generation

### What It Does in Golf Project

The Golf project calls LLMs with:
- Context-aware prompts
- Error handling and fallbacks
- Usage tracking
- Response caching considerations

```python
# From chatbot.py
response, usage = call_cerebras_llm(
    prompt=context,
    model="llama3.1-8b",
    structured=False,
    stream=False,
    temperature=0.8
)

# Log usage for monitoring
if usage:
    print(f"Token usage: {usage}")
    upload_llm_call_info(game_id, ai_bot_id, usage)
```

### Relation to SPARQ Takehome

**Direct Application**: This maps to the **LLM Orchestration** component in SPARQ, which needs:
- Multi-model support (OpenAI, Anthropic, Azure)
- Cost tracking
- Error handling with fallbacks
- Usage logging for monitoring

### Key Insights for SPARQ

1. **Usage Tracking**: The Golf project logs LLM calls - SPARQ needs this for:
   - Cost monitoring
   - Performance metrics
   - Audit compliance

2. **Error Handling**: SPARQ should implement similar error handling:
   ```python
   try:
       response = call_llm(prompt, model="gpt-4")
   except Exception as e:
       # Fallback to cheaper/faster model
       response = call_llm(prompt, model="gpt-3.5-turbo")
   ```

3. **Temperature Control**: Golf project adjusts temperature based on context - SPARQ could:
   - Use lower temperature (0.3) for factual queries
   - Use higher temperature (0.7) for creative/empathetic responses

---

## 5. Role-Based Context Building

### What It Does in Golf Project

The Golf project builds different prompts based on bot personality and difficulty:

```python
# From chatbot.py
def personality_prompt_builder(self, ai_bot_id: str) -> str:
    bot = self.bots[ai_bot_id]
    personality = bot.personality_config

    if personality == "friendly":
        prompt += "You are friendly and encouraging. "
    elif personality == "strategic":
        prompt += "You are balanced and strategic. "
    # ...
    return prompt

def emotional_state_prompt_builder(self, ai_bot_id: str) -> str:
    emotional_state = self.bots[ai_bot_id].emotional_state

    if emotional_state == "confident":
        prompt += "You are confident and assertive. "
    elif emotional_state == "frustrated":
        prompt += "You are frustrated. "
    # ...
    return prompt
```

### Relation to SPARQ Takehome

**Direct Application**: SPARQ needs similar role-based prompt building for:
- **Student prompts**: Friendly, encouraging, age-appropriate
- **Teacher prompts**: Professional, data-focused, curriculum-aware
- **Counselor prompts**: Empathetic, intervention-focused, FERPA-compliant

### Implementation Example for SPARQ

```python
# SPARQ Prompt Engineering Service
class RoleBasedPromptBuilder:
    def build_student_prompt(self, student_context):
        prompt = "You are a helpful academic assistant for students. "
        prompt += "Use friendly, encouraging language appropriate for high school. "
        prompt += "Provide clear, actionable advice. "
        return prompt

    def build_teacher_prompt(self, teacher_context):
        prompt = "You are a professional educational assistant for teachers. "
        prompt += "Provide data-driven insights and curriculum guidance. "
        prompt += "Focus on aggregated, anonymized student data. "
        return prompt

    def build_counselor_prompt(self, counselor_context):
        prompt = "You are a supportive counseling assistant. "
        prompt += "Maintain strict FERPA compliance. "
        prompt += "Provide empathetic, intervention-focused guidance. "
        return prompt
```

---

## 6. Proactive Comment System (Optional Pattern)

### What It Does in Golf Project

The Golf project has a proactive comment system that generates bot responses based on game events, not just user messages:

```python
# From chatbot.py - ChatHandler class
def proactive_comment_timer(self, game_id='global', time_interval=300):
    """Waits for inactivity, then triggers proactive comment"""
    while True:
        if time.time() - last_message_time >= time_interval:
            if self.should_generate_proactive_comment(game_state):
                response = self.chatbot.generate_response(
                    conversation_history, game_state, ai_bot_id
                )
```

### Relation to SPARQ Takehome

**Optional Application**: SPARQ could use this pattern for:
- Proactive alerts to counselors about at-risk students
- Reminders to students about upcoming deadlines
- Notifications to teachers about curriculum updates

**Note**: This is more advanced and may not be in the initial SPARQ MVP, but the pattern is valuable.

---

## 7. Data Upload & Logging (`data_upset.py`)

### What It Does in Golf Project

The Golf project logs interactions to Supabase:

```python
# From data_upset.py (referenced in chatbot.py)
upload_chatbot_message(
    game_id=game_id,
    ai_bot_id=ai_bot_id,
    bot_name=bot_name,
    message=response,
    sender="bot",
    media=None,
    metadata=None
)

upload_llm_call_info(game_id, ai_bot_id, usage)
```

### Relation to SPARQ Takehome

**Direct Application**: SPARQ needs similar logging for:
- **Audit Logs**: Every query/response for FERPA compliance
- **Performance Metrics**: Token usage, latency, costs
- **User Feedback**: Thumbs up/down, reports

### Implementation Pattern for SPARQ

```python
# SPARQ Audit Logging Service
class AuditLogger:
    def log_query(self, user_id: str, role: str, query: str, metadata: dict):
        log_entry = {
            'user_id': user_id,
            'role': role,
            'query': query,
            'timestamp': time.time(),
            'metadata': metadata  # IP address, session ID, etc.
        }
        # Store in PostgreSQL or audit log service
        self.db.insert('audit_logs', log_entry)

    def log_response(self, user_id: str, response: str, llm_usage: dict):
        log_entry = {
            'user_id': user_id,
            'response': response,
            'token_usage': llm_usage,
            'timestamp': time.time()
        }
        self.db.insert('audit_logs', log_entry)
```

---

## Summary: Key Takeaways

### Components Directly Applicable

1. **Embedding Models with JSON Caching** → SPARQ's Embedding Generation
2. **Context Management & Prompt Building** → SPARQ's Prompt Engineering Service
3. **Conversation History Management** → SPARQ's Context Management
4. **Role-Based Prompt Building** → SPARQ's Role-Based Access Control
5. **LLM Orchestration Patterns** → SPARQ's LLM Orchestration
6. **Audit Logging Patterns** → SPARQ's Monitoring & Governance

### Implementation Patterns to Adopt

1. **Layered Context Building**: System prompt → Domain knowledge → Current state → History → Role context
2. **JSON File-Based Caching**: For embeddings and semantic cache
3. **Per-Session History Management**: Similar to per-game history in Golf
4. **Error Handling with Fallbacks**: Multi-model support with graceful degradation
5. **Usage Tracking**: Log all LLM calls for cost and performance monitoring

### Differences to Consider

| Golf Project | SPARQ Project |
|-------------|---------------|
| Game state context | Student data context |
| Bot personalities | User roles (student/teacher/counselor) |
| Game session IDs | User/session IDs |
| Entertainment focus | Educational/compliance focus |
| No PII concerns | Strict FERPA compliance required |

---

## Code References

### Golf Project Files

- `backend/embedding_models.py` - Embedding generation with caching
- `backend/chatbot.py` - Context management and prompt building
- `backend/web_app.py` - API endpoints and session management
- `backend/data_upset.py` - Logging and data upload (referenced)

### SPARQ Architecture Files

- `ARCHITECTURE.md` - System architecture diagram
- `ARCHITECTURE_EXPLAINED.md` - Detailed component explanations
- `CODEBASE_ARCHITECTURE.md` - Implementation details

---

## Next Steps

1. **Adapt EmbeddingModel class** for SPARQ's document ingestion pipeline
2. **Extract prompt building patterns** from Golf's chatbot.py for SPARQ's Prompt Engineering Service
3. **Implement conversation history management** similar to Golf's per-game history
4. **Add FERPA-specific logging** on top of Golf's audit logging patterns
5. **Build role-based prompt templates** inspired by Golf's personality system

---

**Note**: The Golf project demonstrates production-ready patterns for LLM integration, context management, and caching that can be directly adapted for the SPARQ takehome project with appropriate modifications for the educational domain and compliance requirements.

