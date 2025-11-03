# Week 14: AI and Advanced Integrations - Visual Guides

This document contains all visual diagrams for Week 14 content.

## Table of Contents

1. [AI Integration Architecture](#ai-integration-architecture)
2. [Chatbot Conversation Flow](#chatbot-conversation-flow)
3. [RAG (Retrieval Augmented Generation)](#rag-retrieval-augmented-generation)
4. [Content Generation Pipeline](#content-generation-pipeline)
5. [Document Processing Workflow](#document-processing-workflow)
6. [Sentiment Analysis System](#sentiment-analysis-system)

---

## AI Integration Architecture

### n8n + AI Services Architecture

```mermaid
graph TB
    subgraph "User Interfaces"
        WEB[Web App]
        SLACK[Slack]
        EMAIL[Email]
        API[REST API]
    end

    subgraph "n8n Orchestration Layer"
        ROUTER[Request Router]
        CONTEXT[Context Manager<br/>Store Conversation History]
        WORKFLOW[AI Workflow Engine]
    end

    subgraph "AI Services"
        OPENAI[OpenAI<br/>GPT-4, GPT-3.5]
        CLAUDE[Anthropic Claude]
        LANGCHAIN[LangChain<br/>Advanced Chains]
        EMBEDDINGS[Text Embeddings<br/>Vector Search]
    end

    subgraph "Data Layer"
        VECTOR[(Vector Database<br/>Pinecone/Weaviate)]
        KNOWLEDGE[(Knowledge Base<br/>Documents/FAQs)]
        CACHE[(Response Cache<br/>Redis)]
    end

    WEB --> ROUTER
    SLACK --> ROUTER
    EMAIL --> ROUTER
    API --> ROUTER

    ROUTER --> CONTEXT
    CONTEXT --> WORKFLOW

    WORKFLOW --> OPENAI
    WORKFLOW --> CLAUDE
    WORKFLOW --> LANGCHAIN
    WORKFLOW --> EMBEDDINGS

    EMBEDDINGS --> VECTOR
    WORKFLOW --> KNOWLEDGE
    WORKFLOW --> CACHE

    VECTOR --> WORKFLOW
    KNOWLEDGE --> WORKFLOW
    CACHE --> WORKFLOW

    style ROUTER fill:#EA4B71,stroke:#333,color:#fff
    style WORKFLOW fill:#FF9800,stroke:#333,color:#fff
    style OPENAI fill:#4CAF50,stroke:#333,color:#fff
    style VECTOR fill:#2196F3,stroke:#333,color:#fff
```

### AI Model Selection Flow

```mermaid
flowchart TD
    REQUEST[User Request] --> ANALYZE[Analyze Request]

    ANALYZE --> TYPE{Request Type?}

    TYPE -->|Simple Q&A| GPT35[GPT-3.5-turbo<br/>Fast & Cost-Effective]
    TYPE -->|Complex Reasoning| GPT4[GPT-4<br/>Advanced Capabilities]
    TYPE -->|Long Context| CLAUDE[Claude 3<br/>Large Context Window]
    TYPE -->|Code Generation| CODEX[Codex/GPT-4<br/>Code Optimized]

    GPT35 --> CACHE{In Cache?}
    GPT4 --> PROCESS[Process with AI]
    CLAUDE --> PROCESS
    CODEX --> PROCESS

    CACHE -->|Yes| RETURN_CACHE[Return Cached Response]
    CACHE -->|No| PROCESS

    PROCESS --> VALIDATE[Validate Response]

    VALIDATE -->|Invalid| RETRY{Retry Count<br/>< 3?}
    RETRY -->|Yes| PROCESS
    RETRY -->|No| FALLBACK[Use Fallback Response]

    VALIDATE -->|Valid| STORE[Store in Cache]
    STORE --> RETURN[Return to User]

    RETURN_CACHE --> RETURN
    FALLBACK --> RETURN

    style GPT35 fill:#4CAF50,stroke:#333,color:#fff
    style GPT4 fill:#FF9800,stroke:#333,color:#fff
    style CLAUDE fill:#2196F3,stroke:#333,color:#fff
    style RETURN fill:#4CAF50,stroke:#333,color:#fff
```

---

## Chatbot Conversation Flow

### Multi-Turn Conversation Management

```mermaid
sequenceDiagram
    participant User
    participant n8n
    participant Context as Context Store
    participant AI as OpenAI API
    participant KB as Knowledge Base

    User->>n8n: "What are your business hours?"
    activate n8n

    n8n->>Context: Get Conversation History<br/>(session_id: abc123)
    Context-->>n8n: [] (No history)

    n8n->>KB: Search for "business hours"
    KB-->>n8n: Found: Mon-Fri 9am-5pm EST

    n8n->>AI: Generate Response<br/>Context: business hours info
    AI-->>n8n: "We're open Mon-Fri, 9am-5pm EST"

    n8n->>Context: Store Interaction<br/>{user: "hours?", bot: "Mon-Fri..."}
    n8n-->>User: "We're open Mon-Fri, 9am-5pm EST"
    deactivate n8n

    Note over User,n8n: User asks follow-up

    User->>n8n: "Are you open on weekends?"
    activate n8n

    n8n->>Context: Get Conversation History
    Context-->>n8n: [Previous interaction about hours]

    n8n->>AI: Generate Response<br/>Context: Previous said Mon-Fri<br/>Question: weekends?
    AI-->>n8n: "No, we're closed on weekends"

    n8n->>Context: Update History
    n8n-->>User: "No, we're closed on weekends"
    deactivate n8n

    Note over User,KB: Chatbot maintains context
```

### Intent Recognition and Routing

```mermaid
flowchart TD
    MESSAGE[User Message] --> ANALYZE[Analyze Intent]

    ANALYZE --> INTENT{Recognized Intent?}

    INTENT -->|FAQ| FAQ[Search Knowledge Base<br/>Return Answer]
    INTENT -->|Support Request| SUPPORT[Create Support Ticket<br/>Notify Team]
    INTENT -->|Product Info| PRODUCT[Query Product Database<br/>Generate Response]
    INTENT -->|Complaint| COMPLAINT[Escalate to Human<br/>High Priority]
    INTENT -->|General Chat| CHAT[Use AI for Response<br/>Casual Conversation]
    INTENT -->|Unclear| CLARIFY[Ask Clarifying Question]

    FAQ --> RESPOND[Send Response]
    SUPPORT --> RESPOND
    PRODUCT --> RESPOND
    COMPLAINT --> HUMAN[Connect to Human Agent]
    CHAT --> AI[Generate AI Response]
    CLARIFY --> RESPOND

    AI --> RESPOND
    HUMAN --> DONE[Conversation Handled]
    RESPOND --> DONE

    style INTENT fill:#EA4B71,stroke:#333,color:#fff
    style COMPLAINT fill:#F44336,stroke:#333,color:#fff
    style HUMAN fill:#FF9800,stroke:#333,color:#fff
    style DONE fill:#4CAF50,stroke:#333,color:#fff
```

### Chatbot Architecture

```mermaid
graph TB
    subgraph "Input Channels"
        SLACK[Slack Bot]
        DISCORD[Discord Bot]
        WHATSAPP[WhatsApp]
        WEBCHAT[Web Chat Widget]
    end

    subgraph "n8n Chatbot Engine"
        NLU[Natural Language<br/>Understanding]
        DM[Dialog Manager<br/>Conversation Flow]
        NLG[Natural Language<br/>Generation]
    end

    subgraph "Backend Services"
        AI[AI Service<br/>OpenAI/Claude]
        KB[Knowledge Base]
        CRM[CRM System<br/>Customer Data]
        TICKET[Ticketing System]
    end

    subgraph "Context & Memory"
        SHORT[Short-term Memory<br/>Current Session]
        LONG[Long-term Memory<br/>User Preferences]
    end

    SLACK --> NLU
    DISCORD --> NLU
    WHATSAPP --> NLU
    WEBCHAT --> NLU

    NLU --> DM
    DM --> NLG

    DM --> AI
    DM --> KB
    DM --> CRM
    DM --> TICKET

    DM <--> SHORT
    DM <--> LONG

    NLG --> SLACK
    NLG --> DISCORD
    NLG --> WHATSAPP
    NLG --> WEBCHAT

    style DM fill:#EA4B71,stroke:#333,color:#fff
    style AI fill:#4CAF50,stroke:#333,color:#fff
```

---

## RAG (Retrieval Augmented Generation)

### RAG Architecture

```mermaid
graph TB
    subgraph "Document Ingestion"
        DOCS[Documents<br/>PDF, Markdown, HTML]
        CHUNK[Text Chunking<br/>Split into Segments]
        EMBED[Generate Embeddings<br/>Text → Vectors]
    end

    subgraph "Vector Storage"
        VECTOR[(Vector Database<br/>Pinecone/Weaviate/Chroma)]
    end

    subgraph "Query Processing"
        QUERY[User Question]
        QUERY_EMBED[Generate Query<br/>Embedding]
        SEARCH[Semantic Search<br/>Find Similar Chunks]
    end

    subgraph "Response Generation"
        CONTEXT[Build Context<br/>Top K Results]
        PROMPT[Create Prompt<br/>Context + Question]
        LLM[Large Language Model<br/>Generate Answer]
        RESPONSE[Formatted Response<br/>With Citations]
    end

    DOCS --> CHUNK
    CHUNK --> EMBED
    EMBED --> VECTOR

    QUERY --> QUERY_EMBED
    QUERY_EMBED --> SEARCH
    SEARCH --> VECTOR
    VECTOR --> CONTEXT

    CONTEXT --> PROMPT
    PROMPT --> LLM
    LLM --> RESPONSE

    style VECTOR fill:#2196F3,stroke:#333,color:#fff
    style LLM fill:#4CAF50,stroke:#333,color:#fff
    style RESPONSE fill:#4CAF50,stroke:#333,color:#fff
```

### RAG Workflow Sequence

```mermaid
sequenceDiagram
    participant User
    participant n8n
    participant Embed as Embedding API
    participant Vector as Vector DB
    participant LLM as Language Model

    Note over User,LLM: Setup Phase (One-time)

    n8n->>n8n: Load Documents
    loop For Each Document
        n8n->>n8n: Split into Chunks<br/>(1000 tokens each)
        n8n->>Embed: Generate Embedding
        Embed-->>n8n: Vector (1536 dimensions)
        n8n->>Vector: Store Chunk + Vector
    end

    Note over User,LLM: Query Phase (User Interaction)

    User->>n8n: "How do I deploy n8n?"
    activate n8n

    n8n->>Embed: Generate Query Embedding
    Embed-->>n8n: Query Vector

    n8n->>Vector: Similarity Search<br/>(Top 5 results)
    Vector-->>n8n: Relevant Chunks:<br/>1. Docker deployment...<br/>2. npm installation...<br/>3. Cloud hosting...

    n8n->>LLM: Generate Answer<br/>Context: [5 chunks]<br/>Question: "How do I deploy n8n?"

    LLM-->>n8n: "There are three main ways<br/>to deploy n8n: 1. Docker..."

    n8n-->>User: Answer + Source Citations
    deactivate n8n
```

### Document Chunking Strategy

```mermaid
flowchart TD
    DOC[Source Document<br/>10,000 words] --> METHOD{Chunking Method?}

    METHOD -->|Fixed Size| FIXED[Fixed Size Chunks<br/>1000 tokens each]
    METHOD -->|Paragraph| PARA[Paragraph Boundaries<br/>Preserve Context]
    METHOD -->|Semantic| SEMANTIC[Semantic Chunking<br/>Topic Changes]

    FIXED --> OVERLAP[Add Overlap<br/>200 token overlap]
    PARA --> OVERLAP
    SEMANTIC --> OVERLAP

    OVERLAP --> EMBED[Generate Embeddings]

    EMBED --> STORE[Store in Vector DB<br/>With Metadata]

    STORE --> INDEXED[Ready for Search ✓]

    style DOC fill:#EA4B71,stroke:#333,color:#fff
    style INDEXED fill:#4CAF50,stroke:#333,color:#fff
```

---

## Content Generation Pipeline

### Automated Content Creation

```mermaid
flowchart TD
    START[Content Request<br/>Topic, Style, Length] --> BRIEF[Generate Content Brief]

    BRIEF --> OUTLINE[Create Outline<br/>Using AI]

    OUTLINE --> APPROVE{Review<br/>Outline?}

    APPROVE -->|Rejected| REVISE[Revise Outline]
    REVISE --> OUTLINE

    APPROVE -->|Approved| GENERATE[Generate Content<br/>Section by Section]

    GENERATE --> ENHANCE[Enhance Content]

    subgraph "Enhancement Steps"
        ENHANCE --> SEO[SEO Optimization<br/>Keywords, Headers]
        SEO --> GRAMMAR[Grammar Check<br/>Style Improvements]
        GRAMMAR --> FACT[Fact Checking<br/>Verify Claims]
    end

    FACT --> REVIEW{Quality<br/>Check?}

    REVIEW -->|Failed| REGENERATE[Regenerate Problem Sections]
    REGENERATE --> ENHANCE

    REVIEW -->|Passed| FORMAT[Format Content<br/>Add Images, Links]

    FORMAT --> PUBLISH{Auto-Publish?}

    PUBLISH -->|No| QUEUE[Add to Review Queue]
    PUBLISH -->|Yes| SCHEDULE[Schedule Publication]

    QUEUE --> HUMAN[Human Review]
    HUMAN --> FINAL[Final Edits]
    FINAL --> SCHEDULE

    SCHEDULE --> DONE[Content Published ✓]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style DONE fill:#4CAF50,stroke:#333,color:#fff
    style REVIEW fill:#FF9800,stroke:#333,color:#fff
```

### Multi-Format Content Generation

```mermaid
graph TB
    subgraph "Input"
        TOPIC[Topic/Keywords]
        AUDIENCE[Target Audience]
        TONE[Tone/Style]
    end

    subgraph "AI Processing"
        RESEARCH[Research Phase<br/>Gather Information]
        ANALYZE[Analyze Requirements]
        GENERATE[Generate Content]
    end

    subgraph "Output Formats"
        BLOG[Blog Post<br/>1500-2000 words]
        SOCIAL[Social Media Posts<br/>Multiple Platforms]
        EMAIL[Email Newsletter<br/>Engaging Format]
        VIDEO[Video Script<br/>Timestamps included]
        INFOGRAPHIC[Infographic Copy<br/>Key Points]
    end

    TOPIC --> RESEARCH
    AUDIENCE --> ANALYZE
    TONE --> ANALYZE

    RESEARCH --> GENERATE
    ANALYZE --> GENERATE

    GENERATE --> BLOG
    GENERATE --> SOCIAL
    GENERATE --> EMAIL
    GENERATE --> VIDEO
    GENERATE --> INFOGRAPHIC

    style GENERATE fill:#EA4B71,stroke:#333,color:#fff
    style BLOG fill:#4CAF50,stroke:#333,color:#fff
    style SOCIAL fill:#2196F3,stroke:#333,color:#fff
```

---

## Document Processing Workflow

### Intelligent Document Processing

```mermaid
flowchart TD
    START[Document Received<br/>PDF, DOCX, Image] --> DETECT{Document Type?}

    DETECT -->|PDF| PDF_PROCESS[Extract Text<br/>from PDF]
    DETECT -->|Image| OCR[OCR Processing<br/>Extract Text]
    DETECT -->|DOCX| DOCX_PROCESS[Extract Text<br/>from DOCX]

    PDF_PROCESS --> CLEAN[Clean & Normalize Text]
    OCR --> CLEAN
    DOCX_PROCESS --> CLEAN

    CLEAN --> CLASSIFY[Classify Document<br/>Using AI]

    CLASSIFY --> TYPE{Document Category}

    TYPE -->|Invoice| INVOICE[Extract Invoice Data<br/>Amount, Date, Vendor]
    TYPE -->|Contract| CONTRACT[Extract Contract Terms<br/>Parties, Dates, Clauses]
    TYPE -->|Resume| RESUME[Extract Candidate Info<br/>Skills, Experience]
    TYPE -->|Support Ticket| TICKET[Extract Issue Details<br/>Priority, Category]

    INVOICE --> VALIDATE[Validate Extracted Data]
    CONTRACT --> VALIDATE
    RESUME --> VALIDATE
    TICKET --> VALIDATE

    VALIDATE --> CONFIDENCE{Confidence<br/>> 90%?}

    CONFIDENCE -->|Yes| PROCESS[Process Automatically]
    CONFIDENCE -->|No| HUMAN_REVIEW[Queue for Human Review]

    HUMAN_REVIEW --> CORRECTIONS[Human Makes Corrections]
    CORRECTIONS --> LEARN[Update AI Model<br/>Learn from Corrections]

    PROCESS --> ROUTE[Route to Appropriate System]
    LEARN --> ROUTE

    ROUTE --> COMPLETE[Processing Complete ✓]

    style START fill:#4CAF50,stroke:#333,color:#fff
    style CLASSIFY fill:#EA4B71,stroke:#333,color:#fff
    style COMPLETE fill:#4CAF50,stroke:#333,color:#fff
```

### Document Analysis Pipeline

```mermaid
sequenceDiagram
    participant User
    participant n8n
    participant OCR as OCR Service
    participant AI as GPT-4 Vision
    participant DB as Database
    participant Storage as File Storage

    User->>n8n: Upload Document
    activate n8n

    n8n->>Storage: Store Original File
    Storage-->>n8n: File ID

    alt PDF or Image
        n8n->>OCR: Extract Text
        OCR-->>n8n: Extracted Text
    end

    n8n->>AI: Analyze Document<br/>Extract Key Information
    activate AI

    AI-->>n8n: Structured Data:<br/>{<br/>  type: "invoice"<br/>  amount: 1250.00<br/>  date: "2024-01-15"<br/>  vendor: "Acme Corp"<br/>}
    deactivate AI

    n8n->>DB: Store Extracted Data
    n8n->>DB: Link to Original File

    n8n-->>User: Processing Complete<br/>Extracted Data
    deactivate n8n
```

---

## Sentiment Analysis System

### Real-Time Sentiment Analysis

```mermaid
flowchart TD
    INPUT[Text Input<br/>Review, Comment, Email] --> PREPROCESS[Preprocess Text<br/>Clean, Tokenize]

    PREPROCESS --> ANALYZE[Sentiment Analysis<br/>Using AI]

    ANALYZE --> SENTIMENT{Sentiment Score}

    SENTIMENT -->|0.8 - 1.0| POSITIVE[Very Positive ⭐⭐⭐⭐⭐]
    SENTIMENT -->|0.5 - 0.8| SOMEWHAT_POS[Positive ⭐⭐⭐⭐]
    SENTIMENT -->|-0.2 - 0.5| NEUTRAL[Neutral ⭐⭐⭐]
    SENTIMENT -->|-0.5 - -0.2| SOMEWHAT_NEG[Negative ⭐⭐]
    SENTIMENT -->|-1.0 - -0.5| NEGATIVE[Very Negative ⭐]

    POSITIVE --> CATEGORIZE[Categorize Topics]
    SOMEWHAT_POS --> CATEGORIZE
    NEUTRAL --> CATEGORIZE
    SOMEWHAT_NEG --> CATEGORIZE
    NEGATIVE --> CATEGORIZE

    CATEGORIZE --> TOPICS[Extract Topics:<br/>• Product Quality<br/>• Customer Service<br/>• Pricing<br/>• Delivery]

    TOPICS --> STORE[Store Analysis]

    STORE --> ROUTE{Requires Action?}

    ROUTE -->|Negative| ALERT[Alert Support Team<br/>Immediate Response]
    ROUTE -->|Positive| MARKETING[Send to Marketing<br/>Potential Testimonial]
    ROUTE -->|Neutral| LOG[Log for Analysis]

    ALERT --> COMPLETE[Analysis Complete]
    MARKETING --> COMPLETE
    LOG --> COMPLETE

    style POSITIVE fill:#4CAF50,stroke:#333,color:#fff
    style NEGATIVE fill:#F44336,stroke:#333,color:#fff
    style NEUTRAL fill:#9E9E9E,stroke:#333,color:#fff
    style ALERT fill:#F44336,stroke:#333,color:#fff
```

### Sentiment Trend Analysis

```mermaid
graph TB
    subgraph "Data Collection"
        REVIEWS[Product Reviews]
        SOCIAL[Social Media Mentions]
        SUPPORT[Support Tickets]
        SURVEYS[Customer Surveys]
    end

    subgraph "Processing"
        AGGREGATE[Aggregate Sentiments]
        CALCULATE[Calculate Metrics<br/>• Average Score<br/>• Trend Direction<br/>• Volume Changes]
    end

    subgraph "Insights"
        WEEKLY[Weekly Sentiment: +0.65<br/>↑ Improved by 0.12]
        TOPICS_POS[Top Positive Topics:<br/>1. Fast Delivery<br/>2. Quality]
        TOPICS_NEG[Top Negative Topics:<br/>1. Price<br/>2. UI Issues]
    end

    subgraph "Actions"
        NOTIFY[Notify Stakeholders]
        IMPROVE[Prioritize Improvements]
        CELEBRATE[Celebrate Wins]
    end

    REVIEWS --> AGGREGATE
    SOCIAL --> AGGREGATE
    SUPPORT --> AGGREGATE
    SURVEYS --> AGGREGATE

    AGGREGATE --> CALCULATE

    CALCULATE --> WEEKLY
    CALCULATE --> TOPICS_POS
    CALCULATE --> TOPICS_NEG

    WEEKLY --> NOTIFY
    TOPICS_POS --> CELEBRATE
    TOPICS_NEG --> IMPROVE

    style WEEKLY fill:#4CAF50,stroke:#333,color:#fff
    style TOPICS_POS fill:#4CAF50,stroke:#333,color:#fff
    style TOPICS_NEG fill:#F44336,stroke:#333,color:#fff
```

---

## AI Cost Optimization

### Token Usage Optimization

```mermaid
flowchart TD
    REQUEST[AI Request] --> CHECK_CACHE{Response<br/>Cached?}

    CHECK_CACHE -->|Yes| RETURN_CACHE[Return from Cache<br/>$0.00]

    CHECK_CACHE -->|No| MODEL{Choose Model}

    MODEL -->|Simple Task| GPT35[GPT-3.5-turbo<br/>~$0.002 per request]
    MODEL -->|Complex Task| GPT4[GPT-4<br/>~$0.06 per request]

    GPT35 --> PROMPT_OPTIMIZE[Optimize Prompt<br/>Minimize Tokens]
    GPT4 --> PROMPT_OPTIMIZE

    PROMPT_OPTIMIZE --> MAKE_REQUEST[Make API Request]

    MAKE_REQUEST --> RESPONSE[Receive Response]

    RESPONSE --> CACHE_STORE[Store in Cache<br/>TTL: 24 hours]

    CACHE_STORE --> LOG[Log Token Usage<br/>Track Costs]

    LOG --> RETURN[Return Response]

    RETURN_CACHE --> END[Complete]
    RETURN --> END

    style CHECK_CACHE fill:#FF9800,stroke:#333,color:#fff
    style RETURN_CACHE fill:#4CAF50,stroke:#333,color:#fff
    style GPT35 fill:#4CAF50,stroke:#333,color:#fff
    style GPT4 fill:#F44336,stroke:#333,color:#fff
```

### Cost Monitoring Dashboard

```mermaid
graph TB
    subgraph "Daily AI Costs"
        TODAY[Today: $127.45<br/>↓ 12% vs yesterday]
        BREAKDOWN[Model Breakdown:<br/>• GPT-4: $89.20 (70%)<br/>• GPT-3.5: $28.15 (22%)<br/>• Embeddings: $10.10 (8%)]
    end

    subgraph "Usage Metrics"
        REQUESTS[Total Requests: 8,456<br/>Cache Hit Rate: 34%]
        AVG_TOKENS[Avg Tokens per Request:<br/>Input: 450<br/>Output: 280]
    end

    subgraph "Optimization Opportunities"
        CACHE_MORE[Increase Cache TTL<br/>Potential Savings: $23/day]
        USE_GPT35[Use GPT-3.5 More<br/>Potential Savings: $15/day]
        OPTIMIZE_PROMPTS[Shorten Prompts<br/>Potential Savings: $8/day]
    end

    style TODAY fill:#4CAF50,stroke:#333,color:#fff
    style CACHE_MORE fill:#FF9800,stroke:#333,color:#fff
```

---

## Quick AI Integration Patterns

### Common AI Use Cases in n8n

```mermaid
mindmap
  root((AI Use Cases))
    Customer Support
      Chatbots
      Ticket Routing
      Response Suggestions
    Content Creation
      Blog Posts
      Social Media
      Product Descriptions
    Data Analysis
      Sentiment Analysis
      Trend Detection
      Categorization
    Document Processing
      OCR & Extraction
      Summarization
      Translation
    Automation Enhancement
      Smart Routing
      Predictive Actions
      Anomaly Detection
```

---

**Use these diagrams to build powerful AI-enhanced workflows in n8n!**
