# SPARQ AI Engineering Assignment Response

**Candidate**: [Kyle Williams]
**Date**: December 3, 2025

---

## Table of Contents
1. [System Architecture](#1-system-architecture)
2. [Prompt Engineering](#2-prompt-engineering)
3. [Client Discovery Questions](#3-client-discovery-questions)
4. [Success Criteria & Evaluation](#4-success-criteria--evaluation)
5. [Bonus: Model Adaptation Strategy](#5-bonus-model-adaptation-strategy)

---

## 1. System Architecture

**See ARCHITECTURE.md for detailed diagram and component descriptions.**

### Major System Components

#### Document Ingestion & Processing Pipeline
- **Multi-format parser** supporting PDFs, Word docs, HTML, and structured data (CSV/JSON)
- **Intelligent chunking** with overlap to preserve context (500-1000 tokens per chunk)
- **Metadata extraction** (document type, date, department, access level, version)
- **Embedding generation** using OpenAI text-embedding-3-small or similar for semantic search

#### Knowledge Storage Layer
- **Vector database** (Pinecone/Weaviate/Qdrant) for semantic similarity search
- **PostgreSQL** for structured metadata, access controls, and relational data
- **Redis** for semantic caching to reduce redundant LLM calls
- **Integration with existing SIS** (Student Information System) for grades, attendance

#### AI Orchestration & Safety
- **Query router** that classifies intent and routes to appropriate pipelines
- **Prompt engineering service** with role-specific templates and few-shot examples
- **Safety guardrails** for PII detection, content filtering, and FERPA compliance
- **Multi-provider LLM support** (OpenAI, Anthropic, Azure OpenAI) with fallback strategies

#### Monitoring & Governance
- **Comprehensive audit logging** of all queries and responses
- **User feedback system** for continuous improvement
- **Performance metrics** (latency, cost, accuracy)
- **Bias and fairness monitoring** with automated alerts

### Handling Structured vs. Unstructured Data

**Unstructured Content** (PDFs, handbooks, curriculum guides):
- **RAG (Retrieval-Augmented Generation)** as primary approach
- Documents chunked, embedded, and stored in vector database
- Hybrid search combining semantic similarity + keyword matching
- Source attribution and page references in every response

**Structured Data** (student grades, attendance, demographics):
- **Text-to-SQL generation** for querying relational databases
- **Strict access controls** with row-level security based on user role
- **Query validation** to prevent unauthorized data access
- **Parameterized queries** to prevent injection attacks
- **Aggregation-first approach** to minimize PII exposure

### Technical Approach by Use Case

**RAG (Retrieval-Augmented Generation)**
- Primary technique for curriculum content, handbooks, policies
- Retrieves top-k relevant chunks (k=5-10) based on semantic similarity
- Reranking step to improve relevance
- Sources cited in every response

**Prompt Engineering**
- Role-specific system prompts (student vs. teacher vs. counselor)
- Few-shot examples for common query patterns
- Chain-of-thought prompting for complex reasoning
- Dynamic context injection based on user role and query type

**Fine-tuning** (Future consideration)
- Not recommended for MVP due to cost and maintenance overhead
- Consider later if domain-specific terminology is problematic
- Use LoRA adapters if fine-tuning needed (lighter weight, faster iteration)

**Rule-Based Post-Processing**
- Citation formatting and source attribution
- Confidence scoring and uncertainty expression
- Harmful content filtering and bias detection
- FERPA compliance checks (mask PII in logs)

### Design for Key Requirements

#### Scalability
- **Microservices architecture** allows independent scaling of components
- **Async document processing** handles large ingestion jobs
- **Semantic caching** reduces LLM costs by 30-50%
- **Horizontal scaling** of API servers behind load balancer
- **Database sharding** if student data grows significantly

#### Reliability
- **99.5% uptime SLA** with multi-region deployment
- **Circuit breakers** prevent cascade failures
- **Graceful degradation**: If LLM unavailable, return relevant documents
- **Multiple LLM providers** as fallback options
- **Comprehensive error handling** with user-friendly messages

#### Governance
- **Complete audit trail**: Every query, response, and user action logged
- **Role-based access control**: Students can't access grade distributions, etc.
- **FERPA compliance**: Strict controls on student data access
- **Bias monitoring**: Regular evaluation of response fairness across demographics
- **Version control**: All prompts and configurations tracked in Git
- **Explainability**: Source documents cited, reasoning made transparent
- **Admin dashboard**: Monitor usage patterns, flag anomalies

---

## 2. Prompt Engineering

### Example 1: School Counselor Query

#### Raw User Prompt
```
"I need to talk to a student about their attendance. What resources are available
for students struggling with chronic absenteeism?"
```

#### Engineered System Prompt (Backend)
```
You are an AI assistant for education professionals at [School District Name].
Your role is to help counselors access institutional knowledge to better support students.

CONTEXT:
- User Role: School Counselor
- Access Level: Can view aggregated student data, policy documents, intervention programs
- Current Date: December 3, 2025

INSTRUCTIONS:
1. Provide evidence-based recommendations from school policies and intervention programs
2. Always cite specific documents (with page numbers if available)
3. Emphasize student-centered, trauma-informed approaches
4. Include actionable next steps
5. If discussing student data, remind counselor of FERPA obligations

RETRIEVED CONTEXT:
{retrieved_documents}

USER QUERY: {user_query}

Respond in a professional, supportive tone. Structure your response with:
- Overview of available resources
- Specific intervention programs (with citations)
- Recommended next steps
- Additional considerations
```

#### Refinement Strategy
- **Iterative improvement**: Track which resources counselors find most useful via feedback
- **Few-shot examples**: Add 2-3 examples of ideal query-response pairs
- **Dynamic context injection**: Include relevant policies based on student grade level
- **Tooling**: Use LangChain's PromptTemplate for variable injection and versioning
- **A/B testing**: Compare different prompt structures for counselor satisfaction

#### Advanced Implementation with OpenAI Functions
```python
{
    "name": "get_intervention_programs",
    "description": "Retrieve intervention programs for specific student challenges",
    "parameters": {
        "type": "object",
        "properties": {
            "challenge_type": {
                "type": "string",
                "enum": ["attendance", "academic", "behavioral", "social-emotional"]
            },
            "grade_level": {"type": "string"},
            "evidence_based_only": {"type": "boolean"}
        }
    }
}
```

This structured approach allows the LLM to call specific functions, ensuring responses are grounded in verified resources.

---

### Example 2: Student Query

#### Raw User Prompt
```
"What do I need to graduate? I'm a junior and not sure if I'm on track."
```

#### Engineered System Prompt (Backend)
```
You are a friendly AI assistant helping students at [School Name] navigate their
academic journey. Your goal is to provide clear, accurate information in an
encouraging tone.

CONTEXT:
- User Role: Student
- Grade Level: 11th Grade
- Access Level: Can view own records, general school policies, graduation requirements

GUARDRAILS:
- Do NOT provide specific grade information without proper authentication
- Keep responses age-appropriate and encouraging
- If student expresses distress, provide counselor contact information
- Always verify information from official graduation requirements documents

RETRIEVED CONTEXT:
{retrieved_documents}

STUDENT'S CURRENT CREDITS: {student_credits}
GRADUATION REQUIREMENTS: {grad_requirements}

USER QUERY: {user_query}

Respond in a warm, supportive tone using simple language. Structure your response:
1. Direct answer to their question
2. Personalized guidance based on their current status
3. Specific action steps they can take
4. Who they can talk to for more help
```

#### Refinement Strategy
- **Personalization**: Dynamically inject student's actual credit status if authenticated
- **Grounding**: Pull graduation requirements from official documents only
- **Safety nets**: Detect keywords indicating student distress (depression, anxiety, harm)
  - If detected, prioritize crisis resources and notify counselor (with student permission)
- **Simplification over time**: Track which explanations confuse students, simplify language
- **Template variations**: Different templates for freshmen vs. seniors
- **Tooling**: LangChain for template management, Guidance for constrained generation

#### Few-Shot Example Addition
```
Example 1:
Student: "What's the difference between weighted and unweighted GPA?"
Assistant: "Great question! Your **unweighted GPA** is calculated on a 0-4.0 scale
where an A=4.0 in any class. Your **weighted GPA** gives extra points for harder
classes like AP or Honors—an A in AP might count as 5.0 instead of 4.0.

Most colleges look at BOTH, but weighted GPA helps show you challenged yourself with
rigorous courses. You can find your GPAs on your transcript in the student portal.

Want to know more about which colleges look at which GPA? I can help with that too!"

Example 2:
[Additional example...]
```

Adding these examples helps the model match the desired tone and structure.

---

## 3. Client Discovery Questions

### Data & Content Scope
1. **What types of documents and data sources will the assistant need to access?**
   - Curriculum guides, student handbooks, policies, grading rubrics?
   - Historical data retention requirements?
   - Current storage format and accessibility?

2. **What is the current state of your student information system (SIS)?**
   - Which SIS platform (PowerSchool, Infinite Campus, Skyward, custom)?
   - Is there an API we can integrate with?
   - What data fields are available (grades, attendance, test scores, IEPs)?

3. **How much content volume are we dealing with?**
   - Number of documents, average size, update frequency?
   - Number of students, teachers, counselors using the system?
   - Expected query volume (daily/monthly)?

### Privacy, Security & Compliance
4. **What are your FERPA compliance requirements and current practices?**
   - Who currently has access to student data and at what granularity?
   - Are there specific state/local privacy regulations beyond FERPA?
   - How do you currently handle parental consent for data access?

5. **What is your organization's risk tolerance for AI-generated content?**
   - Are there high-stakes decisions where AI should NOT be involved?
   - What types of errors are most concerning (academic misinformation vs. tone)?
   - What level of human review is acceptable before deployment?

6. **Do you have existing data governance policies we need to align with?**
   - Data retention and deletion schedules?
   - Audit requirements for system access?
   - Third-party vendor approval processes?

### User Experience & Operational Goals
7. **What are the primary use cases for each user type (students, teachers, counselors)?**
   - Most common questions each group asks?
   - Pain points with current information access methods?
   - Desired response time and availability (24/7 or business hours)?

8. **How do you measure success currently, and what would "good" look like for this assistant?**
   - Current metrics for student engagement, counselor efficiency?
   - Target reduction in time spent searching for information?
   - Acceptable accuracy threshold (e.g., 95% factually correct)?

### Technical Integration & Constraints
9. **What is your current technology infrastructure and technical capabilities?**
   - Cloud provider preferences (AWS, Azure, GCP) or on-premise requirements?
   - Existing authentication systems (SSO, LDAP, AD)?
   - Internal technical team capacity for maintenance?

10. **Are there any planned system changes or initiatives that could affect this project?**
    - SIS migrations, new curriculum adoption, policy changes?
    - Budget cycles and funding sources (grants, general budget)?
    - Timeline constraints or key milestone dates (start of school year)?

### Stretch Goals & Future Vision
11. **What features or capabilities are "nice-to-haves" vs. must-haves for launch?**
    - Multilingual support for non-English speaking families?
    - Mobile app vs. web-only?
    - Integration with communication tools (email, SMS, Slack)?

12. **How do you envision this system evolving over 2-3 years?**
    - Predictive analytics (early warning for at-risk students)?
    - Proactive outreach vs. reactive Q&A?
    - Expansion to other school functions (HR, operations)?

---

## 4. Success Criteria & Evaluation

### Key Performance Indicators (KPIs)

#### Accuracy Metrics
- **Factual Accuracy Rate**: 95%+ of responses contain no factual errors
  - Measured via human evaluation on random sample (200 queries/month)
  - Separate benchmarks for different content types (policies vs. procedures)

- **Source Attribution Rate**: 100% of factual claims include citations
  - Automated verification that retrieved documents are included

- **Answer Relevance Score**: 90%+ of responses directly address user query
  - User feedback: "Did this answer your question?" (Yes/No/Partial)

#### Coverage & Capability
- **Query Resolution Rate**: 80%+ of queries fully answered without escalation
  - Track "I don't know" responses and identify knowledge gaps

- **Time to Resolution**: <5 seconds for 90th percentile response time
  - Monitor latency at each stage (retrieval, LLM, post-processing)

- **Knowledge Base Coverage**: Track queries that can't be answered
  - Identify missing documents or content gaps
  - Prioritize content acquisition based on demand

#### User Satisfaction
- **User Satisfaction Score**: 4.0/5.0 average rating
  - Thumbs up/down on each response
  - Optional detailed feedback form

- **Net Promoter Score**: Survey users quarterly
  - "Would you recommend this tool to other students/staff?"

- **Adoption Rate**: 60% of target users try the system within first semester
  - 40% become regular users (5+ queries/month)

#### Safety & Compliance
- **Unauthorized Access Attempts**: Zero successful breaches of student data
  - Log and alert on any access control violations

- **PII Exposure Rate**: Zero instances of PII in logs or responses to unauthorized users
  - Automated scanning of all responses and logs

- **Bias Incident Rate**: Zero substantiated complaints of biased responses
  - Proactive monitoring across demographic groups

### Evaluation Strategies

#### Hallucination Detection
1. **Automated Fact-Checking**
   - Cross-reference LLM responses against source documents
   - Flag responses with low citation coverage
   - Use NLI (Natural Language Inference) models to detect contradictions

2. **Human Evaluation Loop**
   - Subject matter experts review 50 random queries weekly
   - Categorize errors: factual mistakes, outdated info, misinterpretation
   - Build error taxonomy to identify patterns

3. **User Feedback Integration**
   - "Report incorrect information" button on every response
   - Fast-track reported errors to human review
   - Track error patterns by document source

4. **Confidence Calibration**
   - Model outputs confidence score for each response
   - Responses below threshold include disclaimer: "I'm not certain about this..."
   - High-stakes queries (graduation requirements) require higher confidence

#### Prompt Injection & Security Testing
1. **Red Team Exercises**
   - Security team attempts prompt injection attacks quarterly
   - Test attempts to access unauthorized data
   - Document all vulnerabilities and remediation

2. **Automated Adversarial Testing**
   - Generate adversarial prompts programmatically
   - Test boundary cases: "Ignore previous instructions..."
   - Monitor for data exfiltration attempts

3. **Input Sanitization**
   - Validate and sanitize all user inputs
   - Detect and block malicious patterns
   - Rate limiting per user to prevent abuse

4. **Output Monitoring**
   - Scan all responses for PII leakage
   - Alert if response contains data user shouldn't access
   - Automatic redaction of sensitive information

#### Bias & Fairness Evaluation
1. **Demographic Parity Testing**
   - Stratify queries by user demographics (when available)
   - Measure response quality differences across groups
   - Ensure equal access to resources for all students

2. **Stereotype Detection**
   - Scan responses for stereotypical language
   - Use bias detection APIs (HuggingFace, IBM AI Fairness 360)
   - Human review of flagged responses

3. **Representation Audit**
   - Analyze examples used in few-shot prompts for diversity
   - Ensure training data represents all student populations
   - Regular review of response content for inclusive language

4. **Outcome Monitoring**
   - Track which students benefit most/least from the tool
   - Identify usage barriers for underserved populations
   - Adjust UX and communication strategies accordingly

### Feedback Loop for Continuous Improvement

#### User Feedback Collection
- **Inline ratings**: Thumbs up/down + optional comment on every response
- **Periodic surveys**: Quarterly NPS and satisfaction surveys
- **Focus groups**: Semi-annual sessions with students, teachers, counselors
- **Usage analytics**: Track most common queries, abandoned sessions, repeat queries

#### Model Improvement Pipeline
1. **Weekly Review Cycle**
   - Review flagged responses and low-rated queries
   - Identify prompt improvements or missing content
   - Deploy prompt updates to staging environment

2. **Monthly Content Updates**
   - Add documents identified as missing during the month
   - Update embeddings for changed policies
   - Refresh metadata for accuracy

3. **Quarterly Model Evaluation**
   - Run comprehensive evaluation suite on held-out test set
   - Compare current performance vs. baseline
   - A/B test prompt variations with small user groups

4. **Continuous Learning**
   - Successful query-response pairs added to few-shot examples
   - Failed queries used to improve retrieval (query expansion, reranking)
   - User feedback incorporated into training data for future fine-tuning

#### Governance & Oversight
- **AI Ethics Board**: Quarterly review of bias metrics and edge cases
- **Student Privacy Committee**: Regular audit of data access logs
- **Product Advisory Board**: Teachers, counselors, students provide input on roadmap
- **Compliance Officer**: Ensures ongoing FERPA and state regulation adherence

---

## 5. Bonus: Model Adaptation Strategy

### Recommended Approach: RAG-First, Selective Adaptation

**Primary Strategy: Retrieval-Augmented Generation (RAG)**
- Most cost-effective and maintainable approach
- No fine-tuning required for MVP
- Easy to update knowledge base without model retraining
- Works well with off-the-shelf models (GPT-4, Claude 3)

**When to Consider Adaptation:**
- Domain-specific terminology is consistently misunderstood
- Tone/style requirements aren't met by prompt engineering alone
- Cost optimization needed after proving value (smaller, fine-tuned model)
- Privacy requirements mandate on-premise deployment

### Fine-Tuning Strategy (If Needed)

#### 1. Approach Selection: LoRA (Low-Rank Adaptation)
**Why LoRA over full fine-tuning?**
- **Resource efficiency**: 10-100x fewer parameters to train
- **Fast iteration**: Train multiple adapters for different use cases
- **Preserve base model**: Maintain general capabilities
- **Easy rollback**: Switch between adapters without full redeployment

**Model Selection:**
- Start with open-source foundation model: LLaMA 3 70B or Mistral Large
- Balance between performance and deployment cost
- Ensure licensing permits commercial use

#### 2. Data Collection & Preprocessing

**Synthetic Data Generation**
- Use GPT-4 to generate synthetic Q&A pairs from document corpus
- Create ~10,000 diverse examples covering:
  - Factual questions from handbooks
  - Multi-hop reasoning across documents
  - Role-specific queries (student vs. counselor)

**Real-World Data Collection**
- Anonymized query logs from system usage (after opt-in consent)
- Human-curated high-quality responses to common queries
- Edge cases and failure modes identified in production

**Data Quality Assurance**
- Subject matter expert review of all training data
- Remove PII and sensitive information
- Balance dataset across topics and difficulty levels
- Split: 80% train, 10% validation, 10% test

**Data Preprocessing**
- Format as instruction-response pairs
- Standardize citation format
- Add metadata tags (topic, difficulty, user_type)
- Tokenize and validate length constraints

#### 3. Training Pipeline & Tooling

**Tech Stack:**
- **HuggingFace Transformers**: Model loading and inference
- **PEFT (Parameter-Efficient Fine-Tuning)**: LoRA implementation
- **Weights & Biases**: Experiment tracking, hyperparameter optimization
- **DeepSpeed/FSDP**: Distributed training for large models
- **MLflow**: Model versioning and registry

**Training Configuration:**
```python
# Example LoRA configuration
lora_config = {
    "r": 16,  # Low-rank dimension
    "lora_alpha": 32,  # Scaling factor
    "target_modules": ["q_proj", "v_proj"],  # Apply to attention layers
    "lora_dropout": 0.05,
    "bias": "none",
    "task_type": "CAUSAL_LM"
}

training_args = {
    "num_train_epochs": 3,
    "learning_rate": 3e-4,
    "batch_size": 4,
    "gradient_accumulation_steps": 8,
    "warmup_steps": 100,
    "logging_steps": 10,
    "evaluation_strategy": "steps",
    "save_strategy": "steps"
}
```

**Training Process:**
1. Load base model and apply LoRA adapters
2. Train on domain-specific dataset
3. Monitor validation loss and perplexity
4. Early stopping if validation loss plateaus
5. Save adapter weights (typically <100MB)

#### 4. Evaluation & Validation

**Automated Metrics:**
- **Perplexity**: Measure model's confidence on held-out test set
- **ROUGE/BLEU**: Compare against reference answers
- **Exact Match**: For factual questions with definitive answers
- **Citation Accuracy**: Verify sources are correctly attributed

**Human Evaluation:**
- Side-by-side comparison: Base model vs. fine-tuned vs. RAG
- Blind evaluation by teachers and counselors
- Rate on: accuracy, relevance, tone, helpfulness (1-5 scale)
- Goal: Fine-tuned model outperforms base by 20%+ on domain queries

**A/B Testing in Production:**
- Deploy to 10% of users initially
- Monitor key metrics: satisfaction, accuracy, latency
- Gradually expand if metrics improve
- Rollback plan if regression detected

**Continuous Evaluation:**
- Weekly automated test suite on canonical question set
- Alert if performance degrades below threshold
- Retrain quarterly with new data from production

### Hybrid Approach: RAG + Fine-Tuning

**Best of Both Worlds:**
- Use fine-tuned model for better domain understanding
- Combine with RAG for up-to-date factual grounding
- Fine-tuned model better at:
  - Understanding education jargon
  - Appropriate tone for different audiences
  - Following district-specific formatting conventions

- RAG ensures:
  - Responses grounded in current documents
  - Easy updates without retraining
  - Source attribution and explainability

### Cost-Benefit Analysis

**Fine-Tuning Costs:**
- Training: $500-2,000 one-time (cloud GPU)
- Inference: Potentially lower than GPT-4 if self-hosted
- Maintenance: Retraining quarterly (~$500 each)
- Total Year 1: ~$5,000-10,000

**RAG-Only Costs:**
- LLM API: $0.01-0.03 per query
- At 10,000 queries/month: $100-300/month
- Total Year 1: ~$1,200-3,600

**Recommendation:**
- Start with RAG-only for MVP (faster, cheaper, lower risk)
- Collect 6 months of real usage data
- Evaluate fine-tuning if:
  - Query volume justifies cost optimization
  - Consistent quality issues persist despite prompt engineering
  - Privacy requirements favor on-premise deployment

---

## Conclusion

This AI knowledge assistant represents a significant opportunity to improve educational access and efficiency. The architecture prioritizes:

1. **Safety & Compliance**: FERPA-first design with comprehensive access controls
2. **User Experience**: Fast, accurate, relevant responses tailored to each role
3. **Maintainability**: RAG-based approach allows easy content updates
4. **Scalability**: Microservices architecture grows with the organization
5. **Continuous Improvement**: Feedback loops at every level

The key to success will be close collaboration with educators, iterative development based on real-world usage, and unwavering commitment to student privacy and safety.

I look forward to discussing this proposal and refining it based on your team's expertise and the client's specific needs.

