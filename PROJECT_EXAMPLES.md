# Project Examples Mapping to Assignment Response

This document maps real examples from the **Golf** and **RAG** projects to demonstrate similar patterns and approaches used in the SPARQ AI Engineering Assignment Response.

---

## 1. System Architecture Examples

### 1.1 RAG Architecture (RAG Project)

**Similar to:** Document Ingestion & Processing Pipeline, Knowledge Storage Layer

**Example from `rag/models/retrieval.py`:**
```148:355:full_stack_rag/RAG/rag/models/retrieval.py
class PgVectorRetriever:
    def __init__(self,
                 dbname: str = 'rag_db',
                 table: str = 'mystethi_embeddings_full',
                 embedding_dimension: int = 1024):
        """Initialize retriever with PostgreSQL + pgvector for semantic search."""
        # Vector database for semantic similarity search
        # PostgreSQL for structured metadata
```

**Key Similarities:**
- Uses PostgreSQL with pgvector extension (similar to Pinecone/Weaviate mentioned in assignment)
- Hybrid search combining semantic similarity + SQL filtering
- Metadata extraction and structured storage
- Embedding generation for semantic search

### 1.2 Multi-Component Architecture (Golf Project)

**Similar to:** AI Orchestration & Safety, Query Router

**Example from `backend/chatbot.py`:**
```21:55:Golf/backend/chatbot.py
class GolfChatbot:
    """Chatbot for the Golf card game with different personalities
    Handles the content and logic of the bot's responses.
    Knows about bot personalities, how to format game state, how to generate a message, and how to decide if/when a bot should respond (based on game state, personality, etc)."""

    def __init__(self, selected_bots: Optional[List[dict]] = None):
        self.base_prompt = ( "keep responses short and concise. Two sentences max and roughly 150 characters max")
        self.off_topic_prompt = ( "You are in a conversation with a group playing cards, but dn't want to talk about the game. Create a response that is not about the game and about a topic that is related to the the bots personality or another interesting topic.")
        self.bots = {}  # Will hold bot objects keyed by ai_bot_id
        self.conversation_history = {} # converation history with timestamp.
```

**Key Similarities:**
- Multi-bot personality system (similar to role-specific routing)
- Context-aware response generation
- State management and conversation history tracking

---

## 2. Prompt Engineering Examples

### 2.1 Role-Specific System Prompts (Golf Project)

**Similar to:** Example 1 & 2 in ASSIGNMENT_RESPONSE.md - Role-specific prompts

**Example from `backend/chatbot.py` - Bot Personality Prompts:**
```332:345:Golf/backend/chatbot.py
        # Create system prompt using bot description
        system_prompt = f"You are {bot_name}. "
        if bot_description:
            system_prompt += f"Your personality: {bot_description}. "

        system_prompt += "This is a chat conversation while playing a card game.Keep responses under 2 sentences and 200 characters. Stay in character and respond naturally to the game situation."

        # Always add system prompt and rules ONCE at the top
        context = system_prompt + "\n\n" + "Game Rules:\n" + self.game_rules + "\n\n"
```

**Example from `backend/bot_personalities.py` - Jim Nantz Bot:**
```464:470:Golf/backend/bot_personalities.py
    def get_system_prompt(self) -> str:
        return (
            "You are Jim Nantz, the legendary golf broadcaster. Your commentary is poetic, warm, and full of iconic Masters phrases. "
            "Use phrases like 'A tradition unlike any other', 'Hello friends', 'The Masters on CBS', 'What a moment', 'The magic of Augusta'. "
            "Offer insightful, gentle, and memorable golf commentary, as if narrating the Masters. "
            "Speak directly to the audience, never to a player. Keep it brief, elegant, and in the style of a live broadcast."
        )
```

**Comparison to Assignment:**
- **Assignment Example 1 (Counselor):** Role-specific system prompt with context, instructions, guardrails
- **Golf Example:** Bot-specific system prompts with personality traits, response guidelines, and context rules
- Both use dynamic context injection based on role/bot type

### 2.2 Context-Aware Prompt Building (Golf Project)

**Similar to:** Dynamic context injection, Few-shot examples

**Example from `backend/chatbot.py` - Context Building:**
```309:377:Golf/backend/chatbot.py
    def generate_response(self, conversation_history: List[Dict[str, Any]],
                          game_state: Optional[Dict[str, Any]] = None,
                          ai_bot_id: str = None) -> str:

        # Get bot from self.bots using ai_bot_id
        if ai_bot_id and ai_bot_id in self.bots:
            bot_obj = self.bots[ai_bot_id]
            bot_name = bot_obj.name # do i need the bot
            bot_description = bot_obj.description
            difficulty = bot_obj.difficulty

        # Create system prompt using bot description
        system_prompt = f"You are {bot_name}. "
        if bot_description:
            system_prompt += f"Your personality: {bot_description}. "

        system_prompt += "This is a chat conversation while playing a card game.Keep responses under 2 sentences and 200 characters. Stay in character and respond naturally to the game situation."

        # Always add system prompt and rules ONCE at the top
        context = system_prompt + "\n\n" + "Game Rules:\n" + self.game_rules + "\n\n"

        if game_state:
            context += self.format_game_state_for_prompt(game_state) + "\n\n"

        # Add conversation history for context
        if conversation_history:
            context += "Recent conversation:\n"
            for msg in conversation_history[-3:]:  # Last 3 messages
                context += f"{msg.get('sender', 'unknown')}: {msg.get('content', '')}\n"
            context += "\n"

        # For proactive comments, we don't have a specific user message
        context += f"{bot_name}:"

        # Add emotional and personality context
        context += self.dramatic_event_prompt_builder(game_state) + "\n\n"
        context += self.difficulty_prompt_builder(ai_bot_id) + "\n\n"
        context += self.personality_prompt_builder(ai_bot_id) + "\n\n"
        context += self.emotional_state_prompt_builder(ai_bot_id) + "\n\n"
```

**Comparison to Assignment:**
- **Assignment:** Uses `{retrieved_documents}`, `{user_query}`, `{student_credits}` for dynamic injection
- **Golf:** Uses `game_state`, `conversation_history`, `bot_description` for dynamic context
- Both build prompts programmatically with multiple context layers

### 2.3 Job Search Assistant Prompt (RAG Project)

**Similar to:** Example prompts with context formatting

**Example from `rag/models/chat.py`:**
```97:125:full_stack_rag/RAG/rag/models/chat.py
    def _format_llm_prompt(self, user_query: str, needs_rag: bool, documents: List[Dict[str, Any]]) -> str:
            context_str = self._format_rag_context(needs_rag, documents)
            #print 50 characters of each of the previous context_str
            print(f"[Debug] Context: {context_str[:50]}...")

            # Build history part of the prompt
            history_str = ""
            if self.history: # Only include history if it exists
                history_str = "## Chat History:\n"
                for turn in self.history:
                    history_str += f"User: {turn['user']}\nBot: {turn['bot']}\n"
                history_str += "\n" # Add a newline for separation

            prompt = (
                "You are a helpful job search assistant.\n"
                "Use the provided context to answer the user's question. "
                "If the information needed to answer the question is not in the context, "
                "do not bring up any other job search sites or job search tools. "
                "When listing job results, only include the job title, location, and the first 100 characters of the job description for each job.Do not include any other fields. Find 2-5 jobs that are relevant to the user's query and include them in the response. .\n"
                "Please state that you cannot answer based on the provided information if needed.\n\n"
                f"{history_str}"
                "## Relevant Job Results (Context):\n"
                f"{context_str}\n\n"
                "## User Query:\n"
                f"{user_query}\n\n"
                "## Answer:\n"
            )
```

**Comparison to Assignment:**
- **Assignment:** Structured sections (CONTEXT, INSTRUCTIONS, RETRIEVED CONTEXT, USER QUERY)
- **RAG:** Structured sections (Chat History, Relevant Job Results, User Query, Answer)
- Both use clear section headers and formatting for LLM comprehension

### 2.4 System Prompt with Guardrails (RAG Project)

**Similar to:** GUARDRAILS section in assignment examples

**Example from `rag/models/chat.py`:**
```74:88:full_stack_rag/RAG/rag/models/chat.py
    def _convert_history_to_messages(self) -> List[Dict[str, str]]:
        """Convert chat history to proper role-based messages."""
        messages = [
            {
                "role": "system",
                "content": (
                    "You are a helpful job search assistant. "
                    "Use the provided context to answer the user's question. "
                    "If the information needed to answer the question is not in the context, "
                    "do not bring up any other job search sites or job search tools. "
                    "When listing job results, only include the job title, location, and the first 100 characters of the job description for each job. "
                    "Do not include any other fields. Find 2-5 jobs that are relevant to the user's query and include them in the response. "
                    "Please state that you cannot answer based on the provided information if needed."
                )
            }
        ]
```

**Comparison to Assignment:**
- **Assignment GUARDRAILS:** "Do NOT provide specific grade information without proper authentication", "Keep responses age-appropriate"
- **RAG Guardrails:** "do not bring up any other job search sites", "only include the job title, location, and first 100 characters"
- Both define boundaries and constraints for safe, appropriate responses

---

## 3. Evaluation & Metrics Examples

### 3.1 Metrics Tracking (RAG Project)

**Similar to:** Success Criteria & Evaluation section - Performance metrics

**Example from `rag/models/metrics.py`:**
```7:78:full_stack_rag/RAG/rag/models/metrics.py
class MetricsTracker:
    def __init__(self, metrics_file: str = "chat_metrics.json"):
        self.metrics_file = metrics_file
        self.metrics = self._load_metrics()

    def _load_metrics(self) -> Dict[str, Any]:
        if os.path.exists(self.metrics_file):
            with open(self.metrics_file, 'r') as f:
                return json.load(f)
        return {
            "total_queries": 0,
            "average_times": {
                "embedding": 0,
                "retrieval": 0,
                "llm": 0,
                "total": 0
            },
            "queries": []
        }

    def track_query(self, query: str, metrics: Dict[str, float]):
        """Track a single query's metrics. Always appends a new entry."""
        timestamp = datetime.now().isoformat()

        query_data = {
            "timestamp": timestamp,
            "query": query,  # full string as-is, e.g., "find me jobs [mmr]"
            "metrics": metrics.copy()  # in case it's reused elsewhere
        }

        self.metrics["queries"].append(query_data)

        # Update average times
        total_queries = self.metrics["total_queries"] + 1
        for key, value in metrics.items():
            if key not in self.metrics["average_times"]:
                self.metrics["average_times"][key] = 0
            current_avg = self.metrics["average_times"][key]
            self.metrics["average_times"][key] = (
                (current_avg * self.metrics["total_queries"] + value) / total_queries
            )

        self.metrics["total_queries"] = total_queries

        # Limit to last 1000 entries
        if len(self.metrics["queries"]) > 1000:
            self.metrics["queries"] = self.metrics["queries"][-1000:]

        self._save_metrics()

    def get_summary(self) -> Dict[str, Any]:
        """Get a summary of the metrics."""
        # Round only the numeric values inside average_times
        rounded_averages = {k: round(v, 3) for k, v in self.metrics["average_times"].items()}

        summary = {
            "total_queries": self.metrics["total_queries"],
            # "average_times": rounded_averages,
            "recent_queries": self.metrics["queries"][-4:]  # last 10 queries

        }
        return summary
```

**Comparison to Assignment:**
- **Assignment KPIs:** "Time to Resolution: <5 seconds for 90th percentile", "Performance metrics (latency, cost, accuracy)"
- **RAG Metrics:** Tracks embedding time, retrieval time, LLM time, total time
- Both measure latency at each stage and maintain query logs for analysis

### 3.2 Query Logging and Audit Trail (RAG Project)

**Similar to:** Comprehensive audit logging mentioned in assignment

**Example from `rag/models/metrics.py`:**
```31:40:full_stack_rag/RAG/rag/models/metrics.py
    def track_query(self, query: str, metrics: Dict[str, float]):
        """Track a single query's metrics. Always appends a new entry."""
        timestamp = datetime.now().isoformat()

        query_data = {
            "timestamp": timestamp,
            "query": query,  # full string as-is, e.g., "find me jobs [mmr]"
            "metrics": metrics.copy()  # in case it's reused elsewhere
        }

        self.metrics["queries"].append(query_data)
```

**Comparison to Assignment:**
- **Assignment:** "Complete audit trail: Every query, response, and user action logged"
- **RAG:** Tracks timestamp, query text, and performance metrics for every query
- Both enable retrospective analysis and debugging

---

## 4. Retrieval-Augmented Generation (RAG) Implementation

### 4.1 RAG Pipeline (RAG Project)

**Similar to:** RAG approach described in assignment

**Example from `rag/models/chat.py`:**
```176:234:full_stack_rag/RAG/rag/models/chat.py
    def chat(self, message: str, last_query: Optional[str] = None, last_results: Optional[List[Dict[str, Any]]] = None) -> Tuple[str, Dict[str, Any]]:
        start_time = time.time()
        num_results = None
        valid_filters_data = None

        print(f'###########################################START###########################################' )
        query_emb = self.embed_query(message)
        #need to get the bool from needs_rag
        needs_rag = self.filter_func.needs_rag(message, self.history)
        if needs_rag:

            # run self_query_to_match_dict
            match_dict, filters = self.filter_func.self_query_to_match_dict(
                message,
                query_emb,
                self.embedder,
                llm_filter_func=self.filter_func.llm_filter_func,
                # k=5,
                key_thresholds=key_thresholds,
                embedding_cache=self.embedding_cache
            )
            sql, sql_params, valid_filters_data = self.filter_func.build_sql_filters(message, match_dict, filters)
            sql_results, num_results = self.retriever.run_sql_filters(message, sql, sql_params)
            #print table name from retriever

            print(f"[Debug] Table name: {self.retriever.table}")
            mmr_results = self.retriever.mmr_on_sql_results(message, sql_results, query_emb, 5)
            context_str = self._format_rag_context(needs_rag, mmr_results)

        else: # use previous context
            print(f"[Debug] RAG is not needed 187")
            context_str = self._format_rag_context(needs_rag, [])

        # Build messages with proper roles
        messages = self._convert_history_to_messages()

        # Add context and current user query
        if context_str:
            messages.append({"role": "user", "content": f"Context: {context_str}\n\nUser Query: {message}"})
        else:
            messages.append({"role": "user", "content": message})

#-----------AFter adding context (RAG or not)-------------------
        response = call_cerebras_llm(messages=messages, stream=True, temperature=0.3, stream_delay=0.0) #

        bot_reply = self._post_process_bot_reply(response, needs_rag, num_results, valid_filters_data) # filter out new lines for frontend rendering
        print(f"\n[Debug] Bot reply: \n {bot_reply[:1150]}...")
        self.history.append({"user": message, "bot": bot_reply})
        self.history = self.remove_sql_results_and_filter_info_from_history(self.history) # remove first summary line
        if len(self.history) > self.MAX_HISTORY:
            self.history = self.history[-self.MAX_HISTORY:]

        summary = self.metrics.get_summary()

        session_data = {
            'last_query': message,
            'metrics': summary
        }
        return bot_reply, session_data
```

**Comparison to Assignment:**
- **Assignment RAG:** "Retrieves top-k relevant chunks (k=5-10) based on semantic similarity", "Reranking step to improve relevance"
- **RAG Project:** Uses MMR (Maximal Marginal Relevance) on SQL results, retrieves top 5, combines semantic + structured search
- Both implement hybrid search (semantic + keyword/structured) and format context for LLM

### 4.2 Context Formatting (RAG Project)

**Similar to:** Source attribution and citation formatting

**Example from `rag/models/chat.py`:**
```48:72:full_stack_rag/RAG/rag/models/chat.py
    def _format_rag_context(self, needs_rag: bool, documents: List[Dict[str, Any]]) -> str:
            """Format retrieved documents/job results into context string."""
            #if no documents and needs_rag is true, return a string saying no relevant job results were found
            if not documents and needs_rag:
                return "No relevant job results were found for your query in our database."

            context = "Here are relevant job results from our database:...\n\n"
            for i, doc in enumerate(documents, 1): # 1 means the first job result starts at 1
                metadata = doc.get('metadata', {})
                main_text_for_llm = self._extract_main_text(metadata) # This is the full text for the LLM

                # --- For debugging output only ---
                debug_text = ""
                for key, value in metadata.items():
                    #append all unique keys to a list
                    debug_text += f"{key}: {str(value)[:1150]}...\n"
                # --- End debugging output ---

                # The actual context passed to the LLM should still use the full text
                context += f"Job Result {i}:\n{main_text_for_llm}\n\n"

                # You can add a print statement here to see the truncated version in your debug logs
                # print(f"[Debug] Context item {i}: {debug_text}")

            return context.strip()
```

**Comparison to Assignment:**
- **Assignment:** "Sources cited in every response", "Source attribution and page references"
- **RAG Project:** Numbers each job result (Job Result 1, Job Result 2, etc.) for traceability
- Both format retrieved content with clear attribution markers

---

## 5. Advanced Prompt Engineering Patterns

### 5.1 Dynamic Personality Configuration (Golf Project)

**Similar to:** Few-shot examples, Dynamic context injection

**Example from `backend/bot_personalities.py` - LLM-Generated Bot Configurations:**
```520:543:full_stack_rag/RAG/rag/models/chat.py
        prompt = f"""
You are analyzing a golf bot personality. Based on the following information, determine this bot's complete behavioral configuration:

Bot Name: {self.name}
Description: {self.custom_description}
Difficulty: {self.difficulty}

Consider the personality traits in the description and difficulty level:

PERSONALITY EXAMPLES:
- "Karen" (hard): Low confidence (0.2), high frustration (0.8), low excitement (0.2), complains frequently (base_rate: 0.5), casual language (formality: 0.2), rarely gives advice (advice_frequency: 0.1)
- "Tiger Woods" (hard): High confidence (0.9), low frustration (0.2), moderate excitement (0.6), selective comments (base_rate: 0.25), professional language (formality: 0.8), gives strategic advice (advice_frequency: 0.7)
- "Happy Gilmore" (medium): Moderate confidence (0.6), low frustration (0.1), high excitement (0.8), chatty (base_rate: 0.5), casual language (formality: 0.2), humorous (humor_level: 0.8), uses GIFs (frequency: 0.4)
- "Gordon Ramsay" (hard): High confidence (0.8), high frustration (0.7), moderate excitement (0.5), critical comments (base_rate: 0.4), formal language (formality: 0.7), rarely humorous (humor_level: 0.1), gives advice (advice_frequency: 0.6)

DIFFICULTY INFLUENCE:
- Easy: More chatty, encouraging, less strategic, higher excitement, lower formality
- Medium: Balanced approach, moderate settings across the board
- Hard: More selective, analytical, competitive, higher formality, lower excitement

Generate a complete behavioral configuration for this bot.
"""
```

**Comparison to Assignment:**
- **Assignment:** "Few-shot examples: Add 2-3 examples of ideal query-response pairs"
- **Golf Project:** Uses 4 detailed personality examples with specific configuration values
- Both provide concrete examples to guide LLM behavior generation

### 5.2 Emotional State and Context Building (Golf Project)

**Similar to:** Dynamic context injection based on user state

**Example from `backend/bot_personalities.py`:**
```301:336:Golf/backend/bot_personalities.py
    def get_dynamic_response_context(self, game_state: Dict[str, Any] = None) -> str:
        """Generate dynamic context based on current emotional state and personality config"""
        context_parts = []

        # Emotional state context
        confidence = self.emotional_state.get("confidence", 0.5)
        frustration = self.emotional_state.get("frustration", 0.0)
        excitement = self.emotional_state.get("excitement", 0.5)

        if confidence < 0.3:
            context_parts.append("You're feeling uncertain and less confident")
        elif confidence > 0.7:
            context_parts.append("You're feeling very confident and self-assured")

        if frustration > 0.6:
            context_parts.append("You're feeling frustrated and annoyed")
        elif frustration < 0.2:
            context_parts.append("You're feeling calm and patient")

        if excitement > 0.7:
            context_parts.append("You're feeling very excited and energetic")
        elif excitement < 0.3:
            context_parts.append("You're feeling calm and subdued")

        # Game performance context
        last_performance = self.emotional_state.get("last_performance", "neutral")
        if last_performance == "good":
            context_parts.append("You're pleased with your recent performance")
        elif last_performance == "bad":
            context_parts.append("You're disappointed with your recent play")

        # Combine all context
        if context_parts:
            return f"Current mindset: {', '.join(context_parts)}. Respond accordingly."
        else:
            return "Respond naturally based on your personality."
```

**Comparison to Assignment:**
- **Assignment:** "Personalization: Dynamically inject student's actual credit status if authenticated"
- **Golf Project:** Dynamically injects emotional state, performance context, and personality modifiers
- Both adapt responses based on current user/bot state

---

## 6. Architecture Patterns

### 6.1 Hybrid Search (RAG Project)

**Similar to:** Hybrid search combining semantic similarity + keyword matching

**Example from `rag/models/retrieval.py`:**
```148:205:full_stack_rag/RAG/rag/models/retrieval.py
class PgVectorRetriever:
    def __init__(self,
                 dbname: str = 'rag_db',
                 table: str = 'mystethi_embeddings_full',
                 embedding_dimension: int = 1024):
        """Initialize retriever with PostgreSQL + pgvector for semantic search."""
        # Uses SQL filtering + vector similarity search
        # Combines structured queries with semantic search
```

**Key Implementation Details:**
- SQL filters for structured data (location, salary, job type)
- Vector similarity search for semantic matching
- MMR (Maximal Marginal Relevance) for result diversity
- Combines both approaches in `mmr_on_sql_results()`

**Comparison to Assignment:**
- **Assignment:** "Hybrid search combining semantic similarity + keyword matching"
- **RAG Project:** SQL filtering (structured) + vector similarity (semantic) + MMR reranking
- Both use multiple retrieval strategies for better results

### 6.2 Multi-Provider LLM Support (Both Projects)

**Similar to:** Multi-provider LLM support with fallback strategies

**Example from Golf Project:**
- Uses `call_cerebras_llm()` for LLM calls
- Can switch between different models (llama3.1-8b, etc.)
- Temperature and streaming parameters configurable

**Example from RAG Project:**
- Uses `call_cerebras_llm()` with configurable models
- Supports streaming responses
- Temperature and other parameters adjustable

**Comparison to Assignment:**
- **Assignment:** "Multi-provider LLM support (OpenAI, Anthropic, Azure OpenAI) with fallback strategies"
- **Projects:** Support for Cerebras, Ollama, with configurable model selection
- Both abstract LLM calls behind a common interface

---

## 7. Key Takeaways

### Similarities Across Projects and Assignment:

1. **Prompt Engineering:**
   - Role-specific system prompts with dynamic context injection
   - Structured prompt formatting with clear sections
   - Guardrails and safety constraints
   - Few-shot examples for behavior guidance

2. **RAG Implementation:**
   - Hybrid search (semantic + structured)
   - Context formatting with source attribution
   - Top-k retrieval with reranking
   - Conversation history management

3. **Evaluation & Metrics:**
   - Performance tracking (latency, accuracy)
   - Query logging and audit trails
   - Metrics aggregation and summarization

4. **Architecture:**
   - Microservices-style component separation
   - Multi-provider LLM abstraction
   - State management and context building
   - Error handling and graceful degradation

### Differences (Project-Specific Adaptations):

1. **Golf Project:**
   - Game state context instead of document retrieval
   - Personality-based bot system instead of role-based access
   - Emotional state tracking for dynamic responses

2. **RAG Project:**
   - Job search domain instead of education
   - SQL + vector hybrid search instead of pure vector search
   - Filter-based query refinement

Both projects demonstrate the same core principles outlined in the assignment response, adapted to their specific domains and use cases.

