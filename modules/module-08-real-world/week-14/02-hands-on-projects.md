# Week 14: Hands-On Projects - AI and Advanced Integrations

## Overview

This document provides detailed specifications, implementation guides, and evaluation criteria for the three hands-on projects in Week 14. Each project is designed to help you apply AI integration concepts in real-world scenarios.

---

## Project 1: AI-Powered Customer Support System

### Project Overview

Build an intelligent, multi-channel customer support chatbot that can handle common queries, provide personalized responses, and escalate complex issues to human agents while continuously learning from interactions.

### Learning Objectives

- Integrate AI with multiple communication platforms
- Implement conversation state management
- Build knowledge base integration with RAG
- Create intelligent routing and escalation logic
- Monitor and optimize AI performance

### Technical Requirements

**Channels to Support:**
- Slack workspace integration
- Email support (IMAP/SMTP)
- Web chat widget (webhook-based)

**Core Features:**
1. Intent recognition and classification
2. Context-aware responses using conversation history
3. Knowledge base Q&A using RAG
4. Sentiment analysis for escalation triggers
5. Human agent escalation workflow
6. Analytics dashboard

**Technology Stack:**
- OpenAI GPT-4 or GPT-3.5-turbo
- Vector database (Pinecone or Weaviate)
- PostgreSQL for conversation history
- n8n workflows for orchestration

### Implementation Steps

#### Step 1: Setup Infrastructure (Day 1)

**1.1 Create Database Schema**

```sql
-- Conversations table
CREATE TABLE conversations (
  id SERIAL PRIMARY KEY,
  user_id VARCHAR(255) NOT NULL,
  channel VARCHAR(50) NOT NULL,
  channel_thread_id VARCHAR(255),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW(),
  status VARCHAR(50) DEFAULT 'active',
  escalated BOOLEAN DEFAULT FALSE,
  sentiment_score DECIMAL(3, 2)
);

-- Messages table
CREATE TABLE messages (
  id SERIAL PRIMARY KEY,
  conversation_id INTEGER REFERENCES conversations(id),
  role VARCHAR(20) NOT NULL, -- 'user' or 'assistant'
  content TEXT NOT NULL,
  tokens_used INTEGER,
  created_at TIMESTAMP DEFAULT NOW(),
  metadata JSONB
);

-- Knowledge base documents
CREATE TABLE kb_documents (
  id SERIAL PRIMARY KEY,
  title VARCHAR(500) NOT NULL,
  content TEXT NOT NULL,
  category VARCHAR(100),
  source VARCHAR(500),
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

-- AI usage tracking
CREATE TABLE ai_usage_log (
  id SERIAL PRIMARY KEY,
  workflow_id VARCHAR(255),
  model VARCHAR(100),
  input_tokens INTEGER,
  output_tokens INTEGER,
  cost DECIMAL(10, 6),
  created_at TIMESTAMP DEFAULT NOW()
);

-- Escalations
CREATE TABLE escalations (
  id SERIAL PRIMARY KEY,
  conversation_id INTEGER REFERENCES conversations(id),
  reason TEXT,
  assigned_to VARCHAR(255),
  status VARCHAR(50) DEFAULT 'pending',
  created_at TIMESTAMP DEFAULT NOW(),
  resolved_at TIMESTAMP
);
```

**1.2 Setup OpenAI Credentials**
- Add OpenAI API key to n8n credentials
- Configure rate limiting
- Set up budget alerts

**1.3 Initialize Vector Database**
- Set up Pinecone or Weaviate account
- Create index for knowledge base
- Configure embedding model

#### Step 2: Build Knowledge Base (Day 1-2)

**Workflow: Import Knowledge Base**

```
Manual Trigger / Schedule
    ↓
Google Sheets: Load KB articles
    OR
    Airtable: Load KB articles
    ↓
Function: Process each article
    Split into chunks
    ↓
OpenAI: Create embeddings
    Model: text-embedding-ada-002
    ↓
Pinecone: Store vectors
    With metadata (title, category, source)
    ↓
PostgreSQL: Store original documents
```

**Sample Knowledge Base Structure:**
```json
{
  "articles": [
    {
      "id": "kb-001",
      "title": "How to reset your password",
      "category": "Account Management",
      "content": "To reset your password: 1. Go to login page...",
      "keywords": ["password", "reset", "login", "forgot"]
    },
    {
      "id": "kb-002",
      "title": "Billing and subscription FAQ",
      "category": "Billing",
      "content": "Billing information...",
      "keywords": ["billing", "payment", "subscription", "invoice"]
    }
  ]
}
```

#### Step 3: Slack Integration (Day 2-3)

**Workflow: Slack Bot Handler**

```
Webhook: /slack/events
    ↓
Function: Verify Slack signature
    ↓
IF: Event type = 'url_verification'
    Return: Challenge response
    ↓
IF: Event type = 'message' AND not bot
    ↓
    PostgreSQL: Load or create conversation
    ↓
    PostgreSQL: Get conversation history (last 10 messages)
    ↓
    Function: Classify intent
    ↓
    Switch: Route by intent
        ↓
        Case 'question':
            Execute: RAG Q&A workflow
        ↓
        Case 'complaint':
            Execute: Sentiment analysis
            → IF negative: Escalate
        ↓
        Case 'request':
            Execute: Action handler
    ↓
    OpenAI: Generate response
        Context: conversation history + knowledge base
    ↓
    PostgreSQL: Save messages
    ↓
    Slack: Send response
    ↓
    Function: Log AI usage
```

**Intent Classification Code:**

```javascript
// Classify user intent
async function classifyIntent(message) {
  const prompt = `Classify this customer support message into one category:

Categories:
- question: Asking for information or how-to
- complaint: Expressing dissatisfaction or reporting a problem
- request: Asking for action to be taken
- feedback: Providing positive or neutral feedback
- greeting: Hello, goodbye, or small talk
- other: Anything else

Message: "${message}"

Respond with just the category name and confidence (0-1):
Format: {"intent": "category", "confidence": 0.95}`;

  const response = await callOpenAI(prompt, {
    model: 'gpt-3.5-turbo',
    temperature: 0.1,
    max_tokens: 50
  });

  return JSON.parse(response);
}
```

#### Step 4: RAG Q&A System (Day 3-4)

**Workflow: Knowledge Base Q&A**

```
Input: User question
    ↓
OpenAI: Create question embedding
    Model: text-embedding-ada-002
    ↓
Pinecone: Search similar documents
    topK: 3
    ↓
Function: Build context from results
    ↓
OpenAI: Generate answer
    Model: gpt-4
    Context: Retrieved documents
    System: "Answer based on knowledge base"
    ↓
Function: Validate answer quality
    ↓
IF: Low confidence OR "I don't know"
    Set flag for escalation
    ↓
Return: Answer + sources + confidence
```

**RAG Implementation Code:**

```javascript
// RAG Q&A function
async function answerQuestion(question, conversationHistory = []) {
  // 1. Create question embedding
  const questionEmbedding = await createEmbedding(question);

  // 2. Search vector database
  const searchResults = await pinecone.query({
    vector: questionEmbedding,
    topK: 3,
    includeMetadata: true
  });

  // 3. Build context
  const context = searchResults.matches
    .map((match, i) => `[Source ${i + 1}: ${match.metadata.title}]\n${match.metadata.content}`)
    .join('\n\n---\n\n');

  // 4. Build conversation context
  const messages = [
    {
      role: 'system',
      content: `You are a helpful customer support AI assistant.
        Use the provided knowledge base context to answer questions accurately.

        Guidelines:
        - Answer based on the context provided
        - If the answer is not in the context, say "I don't have information about that in my knowledge base. Let me connect you with a human agent."
        - Be friendly and professional
        - Keep answers concise but complete
        - Cite sources when relevant

        Knowledge Base Context:
        ${context}`
    }
  ];

  // Add conversation history
  conversationHistory.slice(-5).forEach(msg => {
    messages.push({
      role: msg.role,
      content: msg.content
    });
  });

  // Add current question
  messages.push({
    role: 'user',
    content: question
  });

  // 5. Generate answer
  const response = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: messages,
    temperature: 0.3,
    max_tokens: 500
  });

  const answer = response.choices[0].message.content;

  // 6. Calculate confidence
  const confidence = calculateConfidence(searchResults, answer);

  return {
    answer,
    sources: searchResults.matches.map(m => ({
      title: m.metadata.title,
      category: m.metadata.category,
      similarity: m.score
    })),
    confidence,
    tokensUsed: response.usage.total_tokens,
    cost: calculateCost(response.usage, 'gpt-4')
  };
}

function calculateConfidence(searchResults, answer) {
  // Check if answer indicates uncertainty
  const uncertainPhrases = [
    "i don't have",
    "i'm not sure",
    "i don't know",
    "cannot find",
    "no information"
  ];

  const hasUncertainty = uncertainPhrases.some(phrase =>
    answer.toLowerCase().includes(phrase)
  );

  if (hasUncertainty) return 0.3;

  // Base confidence on top search result similarity
  const topSimilarity = searchResults.matches[0]?.score || 0;

  if (topSimilarity > 0.85) return 0.95;
  if (topSimilarity > 0.75) return 0.80;
  if (topSimilarity > 0.65) return 0.65;
  return 0.50;
}
```

#### Step 5: Sentiment Analysis & Escalation (Day 4)

**Workflow: Sentiment Analysis**

```
Function: Analyze sentiment
    ↓
OpenAI: Detect sentiment + emotion
    ↓
PostgreSQL: Update conversation sentiment
    ↓
IF: Negative sentiment + keywords (urgent, frustrated, cancel)
    OR
    IF: Consecutive negative messages > 2
    OR
    IF: Confidence < 0.5 on answer
    ↓
    Create escalation
        ↓
        PostgreSQL: Insert escalation record
        ↓
        Slack: Notify support team
        ↓
        Email: Alert assigned agent
        ↓
        Update: Conversation status = 'escalated'
```

**Sentiment Analysis Code:**

```javascript
// Comprehensive sentiment analysis
async function analyzeSentimentForEscalation(message, conversationHistory) {
  const prompt = `Analyze this customer support conversation for escalation.

Recent conversation:
${conversationHistory.map(m => `${m.role}: ${m.content}`).join('\n')}

Latest message: "${message}"

Analyze and respond in JSON:
{
  "sentiment": {
    "overall": "positive/neutral/negative",
    "score": -1.0 to 1.0,
    "confidence": 0.0 to 1.0
  },
  "emotion": "satisfied/neutral/frustrated/angry/confused",
  "urgency": "low/medium/high/critical",
  "escalation_recommended": true/false,
  "escalation_reason": "explanation if true",
  "key_concerns": ["concern 1", "concern 2"],
  "suggested_action": "recommendation"
}`;

  const response = await callOpenAI(prompt, {
    model: 'gpt-3.5-turbo',
    temperature: 0.2,
    max_tokens: 400
  });

  const analysis = JSON.parse(response);

  // Check escalation triggers
  const shouldEscalate =
    analysis.escalation_recommended ||
    analysis.urgency === 'critical' ||
    (analysis.sentiment.overall === 'negative' && analysis.sentiment.score < -0.6) ||
    analysis.emotion === 'angry';

  return {
    ...analysis,
    shouldEscalate
  };
}

// Escalation handler
async function createEscalation(conversationId, analysis) {
  // Insert escalation record
  const escalation = await db.query(`
    INSERT INTO escalations (conversation_id, reason, status)
    VALUES ($1, $2, 'pending')
    RETURNING id
  `, [conversationId, analysis.escalation_reason]);

  // Update conversation
  await db.query(`
    UPDATE conversations
    SET status = 'escalated', escalated = true
    WHERE id = $1
  `, [conversationId]);

  // Notify team
  await notifyTeam({
    escalationId: escalation.id,
    conversationId,
    urgency: analysis.urgency,
    reason: analysis.escalation_reason,
    sentiment: analysis.sentiment
  });

  return escalation;
}
```

#### Step 6: Email Integration (Day 5)

**Workflow: Email Support**

```
Email Trigger: IMAP (poll inbox every 5 min)
    ↓
Function: Parse email
    Extract: sender, subject, body
    ↓
PostgreSQL: Find or create conversation
    Based on: sender email
    ↓
Execute: Standard AI workflow
    (Same as Slack - intent, RAG, sentiment)
    ↓
Function: Format email response
    Include: signature, helpful links
    ↓
Email: Send reply (SMTP)
    ↓
IMAP: Move to 'Processed' folder
```

#### Step 7: Analytics Dashboard (Day 6)

**Workflow: Generate Daily Report**

```
Schedule: Every day at 9 AM
    ↓
PostgreSQL: Query statistics
    - Total conversations
    - Messages by channel
    - Escalation rate
    - Average sentiment
    - Top categories
    - AI cost
    ↓
Function: Format report
    ↓
Google Sheets: Update dashboard
    ↓
Slack: Post summary
```

**Analytics Queries:**

```sql
-- Daily statistics
SELECT
  DATE(created_at) as date,
  channel,
  COUNT(*) as total_conversations,
  COUNT(*) FILTER (WHERE escalated = true) as escalations,
  AVG(sentiment_score) as avg_sentiment,
  COUNT(*) FILTER (WHERE status = 'resolved') as resolved
FROM conversations
WHERE created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY DATE(created_at), channel
ORDER BY date DESC;

-- AI cost tracking
SELECT
  DATE(created_at) as date,
  model,
  SUM(input_tokens) as total_input_tokens,
  SUM(output_tokens) as total_output_tokens,
  SUM(cost) as total_cost
FROM ai_usage_log
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY DATE(created_at), model
ORDER BY date DESC;

-- Top categories
SELECT
  category,
  COUNT(*) as query_count
FROM kb_documents kd
JOIN messages m ON m.content ILIKE '%' || kd.title || '%'
WHERE m.created_at >= CURRENT_DATE - INTERVAL '7 days'
GROUP BY category
ORDER BY query_count DESC
LIMIT 10;
```

### Testing Checklist

- [ ] Slack bot responds to mentions
- [ ] Slack bot handles threads correctly
- [ ] Email responses are sent and formatted properly
- [ ] Knowledge base returns relevant answers
- [ ] Low confidence triggers escalation
- [ ] Negative sentiment triggers escalation
- [ ] Conversation history is maintained
- [ ] Analytics dashboard updates daily
- [ ] Cost tracking is accurate
- [ ] Multi-turn conversations work correctly

### Evaluation Criteria

**Functionality (40 points):**
- All three channels working (15 pts)
- RAG knowledge base integration (10 pts)
- Escalation logic functioning (10 pts)
- Analytics dashboard complete (5 pts)

**Code Quality (20 points):**
- Clean, organized workflows
- Proper error handling
- Efficient database queries
- Code documentation

**User Experience (20 points):**
- Natural, helpful responses
- Appropriate escalations
- Reasonable response times
- Professional formatting

**Technical Implementation (20 points):**
- Proper use of embeddings and RAG
- Effective prompt engineering
- Cost optimization strategies
- Scalable architecture

---

## Project 2: Automated Content Generation Pipeline

### Project Overview

Build a comprehensive content generation system that creates blog posts, social media content, and marketing copy with AI assistance, including quality control, SEO optimization, and multi-language support.

### Learning Objectives

- Implement multi-step AI workflows
- Build quality control mechanisms
- Create content for multiple formats
- Integrate with publishing platforms
- Optimize for SEO and engagement

### Technical Requirements

**Content Types:**
1. Long-form blog posts (1500-2500 words)
2. Social media posts (Twitter, LinkedIn, Instagram)
3. Email marketing campaigns
4. Product descriptions

**Core Features:**
1. Content brief processing
2. Automated outline generation
3. Section-by-section writing
4. SEO optimization
5. Fact-checking and quality control
6. Multi-language translation
7. Publishing automation
8. Performance tracking

**Technology Stack:**
- OpenAI GPT-4 for long-form content
- GPT-3.5-turbo for social media
- WordPress/Medium API for publishing
- Google Sheets for content calendar
- Grammarly API (optional) for quality

### Implementation Steps

#### Step 1: Content Brief System (Day 1)

**Workflow: Process Content Brief**

```
Google Form: Submit content brief
    OR
    Google Sheets: New row in content calendar
    ↓
Webhook Trigger
    ↓
Function: Validate brief
    Required fields: topic, audience, keywords, format
    ↓
IF: Missing fields
    Slack: Request more info
    STOP
    ↓
PostgreSQL: Store content request
    Status: 'pending'
    ↓
Execute: Content generation workflow
```

**Content Brief Schema:**

```javascript
{
  "id": "content-001",
  "topic": "How to automate customer support with AI",
  "primaryKeyword": "AI customer support",
  "secondaryKeywords": ["chatbots", "automation", "customer service"],
  "targetAudience": "SaaS founders and customer support managers",
  "tone": "professional but approachable",
  "format": "blog_post",
  "targetWordCount": 2000,
  "includeImages": true,
  "seoTitle": "AI Customer Support: Complete Automation Guide",
  "metaDescription": "Learn how to implement AI-powered customer support...",
  "publishDate": "2025-02-15",
  "language": "en",
  "additionalLanguages": ["es", "fr"]
}
```

#### Step 2: Blog Post Generation (Day 1-2)

**Workflow: Generate Blog Post**

```
Input: Content brief
    ↓
Step 1: Research & Outline
    OpenAI: Generate comprehensive outline
        Model: gpt-4
        Temperature: 0.8
    ↓
    Function: Parse outline into sections
    ↓
Step 2: Write Introduction
    OpenAI: Write engaging intro
        Hook + context + preview
    ↓
Step 3: Write Each Section (Loop)
    For each section in outline:
        ↓
        OpenAI: Write section content
            Include: examples, data, actionable tips
        ↓
        OpenAI: Generate section image prompt
        ↓
        DALL-E: Create image (optional)
    ↓
Step 4: Write Conclusion
    OpenAI: Summarize + CTA
    ↓
Step 5: SEO Optimization
    OpenAI: Optimize for keywords
    ↓
    Function: Check keyword density
    ↓
    OpenAI: Generate meta description
    ↓
Step 6: Quality Check
    OpenAI: Review for quality
    ↓
    Function: Calculate readability score
    ↓
Step 7: Human Review
    Google Docs: Create draft
    ↓
    Slack: Notify editor
    ↓
    Wait for approval
```

**Outline Generation Code:**

```javascript
// Generate blog post outline
async function generateOutline(brief) {
  const prompt = `Create a detailed blog post outline for this topic.

Topic: ${brief.topic}
Target audience: ${brief.targetAudience}
Primary keyword: ${brief.primaryKeyword}
Secondary keywords: ${brief.secondaryKeywords.join(', ')}
Target word count: ${brief.targetWordCount}
Tone: ${brief.tone}

Create an outline with:
1. Compelling H1 title (include primary keyword)
2. Engaging introduction hook (2-3 sentences)
3. 5-7 main sections with H2 headings
4. 2-3 subsections (H3) under each main section
5. Conclusion with call-to-action
6. Suggested FAQ section (3-5 questions)

For each section, provide:
- Heading
- Brief description of what to cover
- Key points to include
- Suggested word count

Format as JSON.`;

  const response = await callOpenAI(prompt, {
    model: 'gpt-4',
    temperature: 0.8,
    max_tokens: 1500
  });

  return JSON.parse(response);
}

// Example outline structure
const outline = {
  "title": "AI Customer Support: Complete Automation Guide for SaaS",
  "hook": "What if your support team could handle 10x more queries without hiring anyone new?",
  "sections": [
    {
      "heading": "What is AI Customer Support?",
      "description": "Define AI customer support and its benefits",
      "keyPoints": [
        "Definition and overview",
        "Key technologies involved",
        "Benefits vs traditional support"
      ],
      "subsections": [
        {
          "heading": "How AI Customer Support Works",
          "points": ["Technology overview", "Integration process"]
        },
        {
          "heading": "Types of AI Support Tools",
          "points": ["Chatbots", "Email automation", "Voice AI"]
        }
      ],
      "wordCount": 300
    }
    // ... more sections
  ],
  "faq": [
    {
      "question": "How much does AI customer support cost?",
      "answerPoints": ["Pricing models", "ROI calculation"]
    }
  ]
};
```

**Section Writing Code:**

```javascript
// Write blog section
async function writeSection(section, brief, previousContent = '') {
  const prompt = `Write the "${section.heading}" section for a blog post.

Blog topic: ${brief.topic}
Target audience: ${brief.targetAudience}
Tone: ${brief.tone}
Primary keyword: ${brief.primaryKeyword} (use naturally)

Section details:
${section.description}

Key points to cover:
${section.keyPoints.map((p, i) => `${i + 1}. ${p}`).join('\n')}

${section.subsections ? `
Subsections to include:
${section.subsections.map(s => `- ${s.heading}: ${s.points.join(', ')}`).join('\n')}
` : ''}

Target word count: ${section.wordCount}

Guidelines:
- Start with a clear H2 heading: ## ${section.heading}
- Write in ${brief.tone} tone
- Include specific examples
- Make it actionable and valuable
- Use markdown formatting
- Include relevant statistics or data points (if applicable)
- Natural keyword usage
${previousContent ? '- Ensure smooth transition from previous section' : ''}

${previousContent ? `
Previous section ending:
${previousContent.slice(-500)}
` : ''}

Write the complete section:`;

  const content = await callOpenAI(prompt, {
    model: 'gpt-4',
    temperature: 0.7,
    max_tokens: Math.ceil(section.wordCount * 1.5)
  });

  return {
    heading: section.heading,
    content: content,
    wordCount: content.split(/\s+/).length
  };
}
```

#### Step 3: SEO Optimization (Day 2)

**Workflow: SEO Optimization**

```
Input: Draft blog post
    ↓
Function: Analyze keyword usage
    Check density: 1-2% for primary
    ↓
Function: Analyze headings
    H1: includes primary keyword
    H2s: include semantic keywords
    ↓
OpenAI: Generate meta description
    Max 160 characters
    Include primary keyword
    Compelling CTA
    ↓
OpenAI: Suggest internal links
    Based on content topic
    ↓
Function: Calculate readability
    Flesch-Kincaid score
    Average sentence length
    ↓
OpenAI: Improve readability if needed
    ↓
Return: Optimized content + SEO metadata
```

**SEO Analysis Code:**

```javascript
// SEO optimization
async function optimizeForSEO(content, brief) {
  // 1. Analyze keyword density
  const keywordAnalysis = analyzeKeywords(content, brief);

  // 2. If density too low, enhance content
  let optimizedContent = content;
  if (keywordAnalysis.primaryKeywordDensity < 0.01) {
    optimizedContent = await enhanceKeywordUsage(
      content,
      brief.primaryKeyword,
      brief.secondaryKeywords
    );
  }

  // 3. Generate meta description
  const metaDescription = await generateMetaDescription(
    optimizedContent,
    brief.primaryKeyword
  );

  // 4. Generate slug
  const slug = brief.topic
    .toLowerCase()
    .replace(/[^a-z0-9]+/g, '-')
    .replace(/^-|-$/g, '');

  // 5. Suggest internal links
  const internalLinks = await suggestInternalLinks(optimizedContent);

  // 6. Readability score
  const readability = calculateReadability(optimizedContent);

  return {
    content: optimizedContent,
    seo: {
      title: brief.seoTitle,
      metaDescription,
      slug,
      primaryKeyword: brief.primaryKeyword,
      secondaryKeywords: brief.secondaryKeywords,
      keywordDensity: keywordAnalysis,
      readabilityScore: readability,
      internalLinks
    }
  };
}

function analyzeKeywords(content, brief) {
  const words = content.toLowerCase().split(/\s+/);
  const totalWords = words.length;

  // Count primary keyword
  const primaryCount = countOccurrences(
    content.toLowerCase(),
    brief.primaryKeyword.toLowerCase()
  );

  // Count secondary keywords
  const secondaryCount = brief.secondaryKeywords.reduce((acc, keyword) => {
    acc[keyword] = countOccurrences(
      content.toLowerCase(),
      keyword.toLowerCase()
    );
    return acc;
  }, {});

  return {
    totalWords,
    primaryKeyword: {
      keyword: brief.primaryKeyword,
      count: primaryCount,
      density: (primaryCount / totalWords) * 100
    },
    secondaryKeywords: Object.entries(secondaryCount).map(([keyword, count]) => ({
      keyword,
      count,
      density: (count / totalWords) * 100
    }))
  };
}

function calculateReadability(text) {
  const sentences = text.split(/[.!?]+/).filter(s => s.trim());
  const words = text.split(/\s+/);
  const syllables = words.reduce((sum, word) => sum + countSyllables(word), 0);

  // Flesch-Kincaid Grade Level
  const avgWordsPerSentence = words.length / sentences.length;
  const avgSyllablesPerWord = syllables / words.length;

  const fleschKincaid =
    (0.39 * avgWordsPerSentence) +
    (11.8 * avgSyllablesPerWord) -
    15.59;

  return {
    score: fleschKincaid,
    level: getReadingLevel(fleschKincaid),
    avgWordsPerSentence: avgWordsPerSentence.toFixed(1),
    avgSyllablesPerWord: avgSyllablesPerWord.toFixed(2)
  };
}

function getReadingLevel(score) {
  if (score < 6) return 'Elementary';
  if (score < 9) return 'Middle School';
  if (score < 13) return 'High School';
  return 'College';
}
```

#### Step 4: Social Media Content (Day 3)

**Workflow: Generate Social Media Posts**

```
Input: Blog post + brief
    ↓
For each platform (Twitter, LinkedIn, Instagram):
    ↓
    OpenAI: Generate platform-specific post
        Consider: character limits, tone, hashtags
    ↓
    OpenAI: Generate variations (3-5)
        Different angles and hooks
    ↓
    IF platform = Instagram:
        DALL-E: Generate image
    ↓
    Google Sheets: Add to content calendar
        With suggested posting times
    ↓
Slack: Send for review
```

**Social Media Generation Code:**

```javascript
// Generate social media content
async function generateSocialMedia(blogPost, brief) {
  const platforms = ['twitter', 'linkedin', 'instagram'];
  const results = {};

  for (const platform of platforms) {
    results[platform] = await generateForPlatform(
      platform,
      blogPost,
      brief
    );
  }

  return results;
}

async function generateForPlatform(platform, blogPost, brief) {
  const platformSpecs = {
    twitter: {
      maxLength: 280,
      tone: 'casual and engaging',
      includeHashtags: true,
      maxHashtags: 2,
      includeEmojis: true
    },
    linkedin: {
      maxLength: 3000,
      tone: 'professional and insightful',
      includeHashtags: true,
      maxHashtags: 5,
      includeEmojis: false
    },
    instagram: {
      maxLength: 2200,
      tone: 'visual and engaging',
      includeHashtags: true,
      maxHashtags: 30,
      includeEmojis: true
    }
  };

  const spec = platformSpecs[platform];

  const prompt = `Create ${platform} posts promoting this blog article.

Blog title: ${blogPost.title}
Blog summary: ${blogPost.sections[0].content.substring(0, 300)}...
Key takeaways: ${blogPost.keyTakeaways.join(', ')}

Platform: ${platform}
Character limit: ${spec.maxLength}
Tone: ${spec.tone}
${spec.includeHashtags ? `Hashtags: yes (max ${spec.maxHashtags})` : ''}
${spec.includeEmojis ? 'Emojis: yes, use sparingly' : 'Emojis: no'}

Create 5 different variations:
1. Statistics/data-driven
2. Question-based
3. Story/example
4. List/tips
5. Call-to-action focused

For each, include:
- Main post text
- Suggested hashtags
- Best time to post
- Expected engagement level

Format as JSON array.`;

  const variations = await callOpenAI(prompt, {
    model: 'gpt-3.5-turbo',
    temperature: 0.9,
    max_tokens: 2000
  });

  return {
    platform,
    variations: JSON.parse(variations),
    blogUrl: blogPost.url
  };
}
```

#### Step 5: Translation Pipeline (Day 4)

**Workflow: Multi-Language Translation**

```
Input: Final English content
    ↓
For each target language:
    ↓
    OpenAI: Translate content
        Maintain formatting
        Preserve markdown
    ↓
    OpenAI: Quality check translation
        Score 0-1
    ↓
    IF quality < 0.8:
        Retry with enhanced prompt
    ↓
    PostgreSQL: Store translation
        Link to original
    ↓
    WordPress: Create translated post
        Set language meta
```

**Translation Code** (see 01-ai-integrations-guide.md for detailed implementation)

#### Step 6: Publishing Automation (Day 5)

**Workflow: Publish to WordPress**

```
Trigger: Manual approval OR scheduled
    ↓
WordPress: Create post
    Title: {{seo.title}}
    Content: {{optimizedContent}}
    Slug: {{seo.slug}}
    Meta: {{seo.metaDescription}}
    Categories: {{categories}}
    Tags: {{keywords}}
    Featured image: {{imageUrl}}
    Status: publish/schedule
    ↓
WordPress: Set SEO meta (Yoast)
    Focus keyword
    Meta description
    ↓
Social media platforms: Schedule posts
    Buffer/Hootsuite API
    ↓
Google Sheets: Update status
    Published: true
    URL: {{postUrl}}
    Date: {{publishDate}}
    ↓
Slack: Notify team
```

**WordPress Publishing Code:**

```javascript
// Publish to WordPress
async function publishToWordPress(content, seo, images) {
  const wp = new WordPressAPI({
    url: process.env.WP_URL,
    username: process.env.WP_USERNAME,
    password: process.env.WP_APP_PASSWORD
  });

  // 1. Upload featured image
  let featuredImageId;
  if (images.featured) {
    const imageBuffer = await downloadImage(images.featured);
    const uploadedImage = await wp.media.create({
      file: imageBuffer,
      title: seo.title,
      alt_text: seo.imageAlt
    });
    featuredImageId = uploadedImage.id;
  }

  // 2. Create post
  const post = await wp.posts.create({
    title: seo.title,
    content: content,
    slug: seo.slug,
    excerpt: seo.metaDescription,
    status: 'publish', // or 'draft'
    featured_media: featuredImageId,
    categories: seo.categories,
    tags: seo.tags,
    meta: {
      // Yoast SEO fields
      _yoast_wpseo_focuskw: seo.primaryKeyword,
      _yoast_wpseo_metadesc: seo.metaDescription,
      _yoast_wpseo_title: seo.title
    }
  });

  return {
    id: post.id,
    url: post.link,
    publishedAt: post.date
  };
}
```

#### Step 7: Performance Tracking (Day 6)

**Workflow: Track Content Performance**

```
Schedule: Daily at 10 AM
    ↓
For each published post (last 30 days):
    ↓
    WordPress: Get post stats
        Views, comments
    ↓
    Google Analytics: Get metrics
        Page views, time on page, bounce rate
    ↓
    Social Media APIs: Get engagement
        Likes, shares, comments
    ↓
    PostgreSQL: Store metrics
    ↓
Google Sheets: Update dashboard
    ↓
OpenAI: Analyze performance
    Insights and recommendations
    ↓
Slack: Weekly report
```

### Testing Checklist

- [ ] Content brief validates all required fields
- [ ] Outline generated matches brief requirements
- [ ] Blog post meets word count target
- [ ] SEO elements optimized (title, meta, keywords)
- [ ] Readability score is appropriate
- [ ] Social media posts fit platform constraints
- [ ] Translations maintain quality (score > 0.8)
- [ ] WordPress publishing successful
- [ ] Images uploaded correctly
- [ ] Analytics tracking working

### Evaluation Criteria

**Functionality (40 points):**
- Complete blog post generation (15 pts)
- Social media content creation (10 pts)
- SEO optimization (10 pts)
- Publishing automation (5 pts)

**Content Quality (25 points):**
- Well-structured and coherent
- Appropriate tone and style
- Valuable and actionable
- Proper keyword usage
- Good readability

**Technical Implementation (20 points):**
- Effective prompt engineering
- Quality control mechanisms
- Error handling
- API integrations

**Innovation (15 points):**
- Creative features
- Unique optimizations
- Efficient workflows

---

## Project 3: Document Analysis and Classification Workflow

### Project Overview

Build an intelligent document processing system that handles multiple document formats, extracts structured information, classifies content, routes documents to appropriate teams, and enables searchable knowledge management.

### Learning Objectives

- Process various document formats
- Extract structured data with AI
- Implement document classification
- Build searchable document database
- Create automated routing workflows

### Technical Requirements

**Supported Formats:**
- PDF documents
- Microsoft Word (DOCX)
- Images (with OCR)
- Text files
- Emails

**Core Features:**
1. Document upload and processing
2. Text extraction (including OCR)
3. Information extraction
4. Document classification
5. Automated routing
6. Full-text search
7. Version control
8. Access control

**Technology Stack:**
- OpenAI for classification and extraction
- Tesseract OCR for image processing
- pdf-parse or PyPDF2 for PDF extraction
- PostgreSQL with full-text search
- Amazon S3 or Google Drive for storage
- Vector database for semantic search

### Implementation Steps

#### Step 1: Document Intake (Day 1)

**Workflow: Upload Document**

```
Trigger Options:
  - Webhook: File upload
  - Email: Attachment received
  - Google Drive: New file in folder
  - Dropbox: File added
    ↓
Function: Validate file
    Check: file type, size, virus scan
    ↓
IF: Invalid
    Notify: Uploader
    STOP
    ↓
Storage: Upload to S3/Drive
    Generate unique ID
    ↓
PostgreSQL: Create document record
    Status: 'processing'
    ↓
Trigger: Processing workflow
```

**Database Schema:**

```sql
-- Documents table
CREATE TABLE documents (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  filename VARCHAR(500) NOT NULL,
  original_filename VARCHAR(500),
  file_type VARCHAR(50),
  file_size BIGINT,
  storage_url TEXT,
  uploaded_by VARCHAR(255),
  uploaded_at TIMESTAMP DEFAULT NOW(),
  status VARCHAR(50) DEFAULT 'processing',
  processed_at TIMESTAMP,
  metadata JSONB
);

-- Document content (for full-text search)
CREATE TABLE document_content (
  id SERIAL PRIMARY KEY,
  document_id UUID REFERENCES documents(id),
  content TEXT,
  page_number INTEGER,
  tsv tsvector GENERATED ALWAYS AS (to_tsvector('english', content)) STORED
);

-- Create index for full-text search
CREATE INDEX idx_document_content_tsv ON document_content USING GIN(tsv);

-- Extracted information
CREATE TABLE extracted_data (
  id SERIAL PRIMARY KEY,
  document_id UUID REFERENCES documents(id),
  field_name VARCHAR(255),
  field_value TEXT,
  confidence DECIMAL(3, 2),
  extracted_at TIMESTAMP DEFAULT NOW()
);

-- Document classifications
CREATE TABLE classifications (
  id SERIAL PRIMARY KEY,
  document_id UUID REFERENCES documents(id),
  category VARCHAR(100),
  subcategory VARCHAR(100),
  confidence DECIMAL(3, 2),
  tags TEXT[],
  classified_at TIMESTAMP DEFAULT NOW()
);

-- Routing and assignments
CREATE TABLE document_routing (
  id SERIAL PRIMARY KEY,
  document_id UUID REFERENCES documents(id),
  assigned_to VARCHAR(255),
  team VARCHAR(100),
  priority VARCHAR(50),
  status VARCHAR(50) DEFAULT 'pending',
  routed_at TIMESTAMP DEFAULT NOW(),
  completed_at TIMESTAMP
);
```

#### Step 2: Text Extraction (Day 1-2)

**Workflow: Extract Text**

```
Input: Document record
    ↓
Switch: By file type
    ↓
    Case 'pdf':
        Code: PDF text extraction
    ↓
    Case 'docx':
        Code: Word extraction
    ↓
    Case 'image' (jpg, png):
        Tesseract: OCR
    ↓
    Case 'email':
        Code: Parse email body
    ↓
Function: Clean extracted text
    Remove noise
    Fix encoding
    ↓
PostgreSQL: Store content
    Full-text indexed
    ↓
OpenAI: Create embeddings
    For semantic search
    ↓
Pinecone: Store vectors
    ↓
Update: Document status = 'text_extracted'
```

**Text Extraction Code:**

```javascript
// PDF extraction
const pdf = require('pdf-parse');

async function extractPDFText(fileBuffer) {
  const data = await pdf(fileBuffer);

  return {
    text: data.text,
    pages: data.numpages,
    info: data.info,
    metadata: data.metadata
  };
}

// Image OCR
const Tesseract = require('tesseract.js');

async function extractImageText(imageBuffer) {
  const result = await Tesseract.recognize(
    imageBuffer,
    'eng',
    {
      logger: m => console.log(m)
    }
  );

  return {
    text: result.data.text,
    confidence: result.data.confidence,
    words: result.data.words.length
  };
}

// DOCX extraction
const mammoth = require('mammoth');

async function extractDocxText(fileBuffer) {
  const result = await mammoth.extractRawText({ buffer: fileBuffer });

  return {
    text: result.value,
    messages: result.messages
  };
}

// Main extraction router
async function extractText(document) {
  const fileBuffer = await downloadFile(document.storage_url);

  let extracted;

  switch(document.file_type) {
    case 'pdf':
      extracted = await extractPDFText(fileBuffer);
      break;
    case 'docx':
      extracted = await extractDocxText(fileBuffer);
      break;
    case 'jpg':
    case 'png':
      extracted = await extractImageText(fileBuffer);
      break;
    default:
      // Plain text
      extracted = { text: fileBuffer.toString('utf8') };
  }

  // Clean text
  const cleanedText = cleanText(extracted.text);

  // Store in database
  await storeDocumentContent(document.id, cleanedText);

  // Create embeddings for semantic search
  const embedding = await createEmbedding(cleanedText.substring(0, 8000));
  await storeEmbedding(document.id, embedding);

  return {
    documentId: document.id,
    textLength: cleanedText.length,
    ...extracted
  };
}

function cleanText(text) {
  return text
    .replace(/\r\n/g, '\n')  // Normalize line endings
    .replace(/\s+/g, ' ')     // Collapse whitespace
    .replace(/[^\x20-\x7E\n]/g, '')  // Remove non-printable chars
    .trim();
}
```

#### Step 3: Document Classification (Day 2-3)

**Workflow: Classify Document**

```
Input: Document with extracted text
    ↓
OpenAI: Classify document
    Categories: invoice, contract, resume, report, etc.
    ↓
Function: Parse classification
    ↓
IF: Confidence < 0.7
    Flag for human review
    ↓
OpenAI: Extract tags
    Generate relevant tags
    ↓
OpenAI: Determine priority
    Based on content and type
    ↓
PostgreSQL: Store classification
    ↓
Update: Document status = 'classified'
    ↓
Trigger: Routing workflow
```

**Classification Code:**

```javascript
// Document classification
async function classifyDocument(documentText, filename) {
  const categories = [
    {
      name: 'invoice',
      description: 'Financial invoices, bills, and payment documents',
      subcat: ['purchase_order', 'receipt', 'invoice', 'credit_note']
    },
    {
      name: 'contract',
      description: 'Legal contracts, agreements, and terms',
      subcat: ['employment', 'nda', 'service_agreement', 'partnership']
    },
    {
      name: 'resume',
      description: 'CVs, resumes, and job applications',
      subcat: ['cv', 'cover_letter', 'portfolio']
    },
    {
      name: 'report',
      description: 'Business reports, analytics, and research documents',
      subcat: ['financial_report', 'research', 'analysis', 'whitepaper']
    },
    {
      name: 'correspondence',
      description: 'Emails, letters, and other communications',
      subcat: ['email', 'letter', 'memo']
    },
    {
      name: 'technical',
      description: 'Technical documentation, specifications, and manuals',
      subcat: ['manual', 'specification', 'documentation']
    },
    {
      name: 'other',
      description: 'Other document types',
      subcat: []
    }
  ];

  const prompt = `Analyze and classify this document.

Filename: ${filename}

Document content (first 3000 characters):
${documentText.substring(0, 3000)}

Available categories:
${categories.map(c => `- ${c.name}: ${c.description}${c.subcat.length ? '\n  Subcategories: ' + c.subcat.join(', ') : ''}`).join('\n')}

Analyze and respond in JSON:
{
  "category": "primary category name",
  "subcategory": "specific subcategory or null",
  "confidence": 0.0 to 1.0,
  "reasoning": "brief explanation",
  "tags": ["tag1", "tag2", "tag3"],
  "priority": "low/medium/high/urgent",
  "priority_reason": "why this priority",
  "sensitive": true/false,
  "language": "detected language code",
  "summary": "one-sentence summary of document"
}`;

  const response = await callOpenAI(prompt, {
    model: 'gpt-4',
    temperature: 0.2,
    max_tokens: 500
  });

  const classification = JSON.parse(response);

  // Store classification
  await db.query(`
    INSERT INTO classifications
      (document_id, category, subcategory, confidence, tags)
    VALUES ($1, $2, $3, $4, $5)
  `, [
    documentId,
    classification.category,
    classification.subcategory,
    classification.confidence,
    classification.tags
  ]);

  return classification;
}
```

#### Step 4: Information Extraction (Day 3-4)

**Workflow: Extract Structured Data**

```
Input: Classified document
    ↓
Switch: By category
    ↓
    Case 'invoice':
        Extract: invoice #, date, vendor, amount, line items
    ↓
    Case 'contract':
        Extract: parties, dates, terms, amounts
    ↓
    Case 'resume':
        Extract: name, contact, experience, skills
    ↓
    Case 'report':
        Extract: title, author, date, key findings
    ↓
OpenAI: Extract information
    Schema based on category
    ↓
Function: Validate extracted data
    Check required fields
    Validate formats
    ↓
PostgreSQL: Store extracted data
    ↓
Update: Document metadata
```

**Information Extraction Code:**

```javascript
// Schema definitions for each document type
const extractionSchemas = {
  invoice: {
    invoice_number: { type: 'string', required: true },
    invoice_date: { type: 'date', required: true },
    due_date: { type: 'date', required: false },
    vendor: {
      name: { type: 'string', required: true },
      address: { type: 'string', required: false },
      tax_id: { type: 'string', required: false }
    },
    customer: {
      name: { type: 'string', required: true },
      address: { type: 'string', required: false }
    },
    line_items: [{
      description: { type: 'string', required: true },
      quantity: { type: 'number', required: true },
      unit_price: { type: 'number', required: true },
      total: { type: 'number', required: true }
    }],
    subtotal: { type: 'number', required: true },
    tax: { type: 'number', required: false },
    total: { type: 'number', required: true },
    currency: { type: 'string', required: false }
  },

  resume: {
    personal: {
      name: { type: 'string', required: true },
      email: { type: 'string', required: true },
      phone: { type: 'string', required: false },
      location: { type: 'string', required: false },
      linkedin: { type: 'string', required: false }
    },
    summary: { type: 'string', required: false },
    experience: [{
      company: { type: 'string', required: true },
      title: { type: 'string', required: true },
      start_date: { type: 'string', required: true },
      end_date: { type: 'string', required: false },
      description: { type: 'string', required: false }
    }],
    education: [{
      institution: { type: 'string', required: true },
      degree: { type: 'string', required: true },
      field: { type: 'string', required: false },
      graduation_date: { type: 'string', required: false }
    }],
    skills: [{ type: 'string' }]
  },

  contract: {
    contract_type: { type: 'string', required: true },
    effective_date: { type: 'date', required: true },
    expiration_date: { type: 'date', required: false },
    parties: [{
      name: { type: 'string', required: true },
      role: { type: 'string', required: true }
    }],
    key_terms: [{
      term: { type: 'string', required: true },
      description: { type: 'string', required: true }
    }],
    financial_terms: {
      amount: { type: 'number', required: false },
      currency: { type: 'string', required: false },
      payment_schedule: { type: 'string', required: false }
    }
  }
};

// Extract information based on schema
async function extractInformation(documentText, category) {
  const schema = extractionSchemas[category];

  if (!schema) {
    console.log(`No extraction schema for category: ${category}`);
    return null;
  }

  const prompt = `Extract structured information from this ${category} document.

Document content:
${documentText}

Extract the following information as JSON:
${JSON.stringify(schema, null, 2)}

Important:
- Extract only factual information present in the document
- Use null for missing optional fields
- Maintain original formatting and spelling
- For dates, use YYYY-MM-DD format if possible
- For numbers, use numeric values (not strings)
- Return valid JSON only

Extracted data:`;

  const response = await callOpenAI(prompt, {
    model: 'gpt-4',
    temperature: 0.1,
    max_tokens: 2000
  });

  try {
    const extracted = JSON.parse(response);

    // Validate extracted data
    const validation = validateExtraction(extracted, schema);

    if (!validation.valid) {
      console.log('Validation errors:', validation.errors);
      // Flag for review but still store what we got
    }

    // Store in database
    await storeExtractedData(documentId, category, extracted, validation.valid);

    return {
      data: extracted,
      valid: validation.valid,
      errors: validation.errors
    };

  } catch (error) {
    console.error('Failed to parse extracted JSON:', error);
    return null;
  }
}

function validateExtraction(data, schema, path = '') {
  const errors = [];

  for (const [key, rules] of Object.entries(schema)) {
    const value = data[key];
    const fieldPath = path ? `${path}.${key}` : key;

    // Check required fields
    if (rules.required && (value === null || value === undefined)) {
      errors.push(`Missing required field: ${fieldPath}`);
      continue;
    }

    if (value === null || value === undefined) continue;

    // Type validation
    if (rules.type) {
      const actualType = Array.isArray(value) ? 'array' : typeof value;

      if (rules.type === 'date') {
        if (!/^\d{4}-\d{2}-\d{2}$/.test(value)) {
          errors.push(`Invalid date format for ${fieldPath}: ${value}`);
        }
      } else if (rules.type === 'number') {
        if (typeof value !== 'number') {
          errors.push(`Expected number for ${fieldPath}, got ${actualType}`);
        }
      } else if (rules.type === 'string') {
        if (typeof value !== 'string') {
          errors.push(`Expected string for ${fieldPath}, got ${actualType}`);
        }
      }
    }

    // Nested object validation
    if (typeof rules === 'object' && !Array.isArray(rules) && !rules.type) {
      const nestedValidation = validateExtraction(value, rules, fieldPath);
      errors.push(...nestedValidation.errors);
    }

    // Array validation
    if (Array.isArray(rules)) {
      if (!Array.isArray(value)) {
        errors.push(`Expected array for ${fieldPath}`);
      } else {
        value.forEach((item, index) => {
          const itemValidation = validateExtraction(
            item,
            rules[0],
            `${fieldPath}[${index}]`
          );
          errors.push(...itemValidation.errors);
        });
      }
    }
  }

  return {
    valid: errors.length === 0,
    errors
  };
}
```

#### Step 5: Document Routing (Day 4-5)

**Workflow: Route Document**

```
Input: Classified document with extracted data
    ↓
Function: Determine routing
    Based on: category, priority, content
    ↓
Switch: By category
    ↓
    Case 'invoice':
        Assign to: Accounting team
        IF amount > $10000:
            Set priority: high
            Notify: CFO
    ↓
    Case 'resume':
        Assign to: HR team
        Extract: Position applied for
        Match: Open positions
    ↓
    Case 'contract':
        Assign to: Legal team
        IF new contract:
            Notify: Department head
    ↓
PostgreSQL: Create routing record
    ↓
Slack/Email: Notify assigned team
    Include: Document summary, priority, link
    ↓
Create: Task in project management system
    ↓
Update: Document status = 'routed'
```

**Routing Logic Code:**

```javascript
// Intelligent document routing
async function routeDocument(document, classification, extractedData) {
  const routing = {
    documentId: document.id,
    category: classification.category,
    priority: classification.priority,
    assignedTo: null,
    team: null,
    reason: '',
    actions: []
  };

  // Route based on category
  switch(classification.category) {
    case 'invoice':
      routing.team = 'accounting';

      if (extractedData.total > 10000) {
        routing.priority = 'high';
        routing.assignedTo = 'cfo@company.com';
        routing.actions.push({
          type: 'notify',
          target: 'cfo@company.com',
          message: `High-value invoice received: $${extractedData.total}`
        });
      } else {
        routing.assignedTo = 'accounts-payable@company.com';
      }

      // Check if vendor exists
      const vendorExists = await checkVendor(extractedData.vendor.name);
      if (!vendorExists) {
        routing.actions.push({
          type: 'create_vendor',
          vendorInfo: extractedData.vendor
        });
      }

      routing.reason = `Invoice from ${extractedData.vendor.name} for $${extractedData.total}`;
      break;

    case 'resume':
      routing.team = 'hr';
      routing.assignedTo = 'recruiting@company.com';

      // Try to match to open positions
      const position = await matchToPosition(extractedData.skills);
      if (position) {
        routing.actions.push({
          type: 'link_to_position',
          positionId: position.id,
          matchScore: position.matchScore
        });
        routing.assignedTo = position.hiringManager;
      }

      routing.reason = `Resume from ${extractedData.personal.name}`;
      break;

    case 'contract':
      routing.team = 'legal';
      routing.assignedTo = 'legal@company.com';
      routing.priority = 'high';

      // Check if renewal or new contract
      const isRenewal = documentText.toLowerCase().includes('renewal');
      if (isRenewal) {
        routing.actions.push({
          type: 'find_original_contract',
          parties: extractedData.parties
        });
      }

      // Notify stakeholders
      extractedData.parties.forEach(party => {
        if (party.role === 'internal') {
          routing.actions.push({
            type: 'notify',
            target: party.email,
            message: 'Contract requiring your review'
          });
        }
      });

      routing.reason = `${extractedData.contract_type} contract`;
      break;

    case 'report':
      routing.team = 'management';
      routing.assignedTo = 'reports@company.com';
      routing.reason = classification.summary;
      break;

    default:
      routing.team = 'general';
      routing.assignedTo = 'inbox@company.com';
      routing.reason = 'General document for review';
  }

  // Store routing
  await db.query(`
    INSERT INTO document_routing
      (document_id, assigned_to, team, priority, status)
    VALUES ($1, $2, $3, $4, 'pending')
  `, [
    routing.documentId,
    routing.assignedTo,
    routing.team,
    routing.priority
  ]);

  // Execute routing actions
  for (const action of routing.actions) {
    await executeRoutingAction(action, document, routing);
  }

  // Send notification
  await notifyAssignee(routing, document, classification);

  return routing;
}

async function notifyAssignee(routing, document, classification) {
  const message = {
    to: routing.assignedTo,
    subject: `New ${classification.category} document: ${document.filename}`,
    body: `
A new document has been assigned to you:

Document: ${document.filename}
Category: ${classification.category}
Priority: ${routing.priority}
Reason: ${routing.reason}

Summary: ${classification.summary}

View document: ${generateDocumentUrl(document.id)}

Tags: ${classification.tags.join(', ')}
    `
  };

  // Send email
  await sendEmail(message);

  // Post to Slack
  await postToSlack({
    channel: getSlackChannel(routing.team),
    text: `New ${routing.priority} priority ${classification.category} document assigned`,
    attachments: [{
      title: document.filename,
      text: classification.summary,
      fields: [
        { title: 'Category', value: classification.category, short: true },
        { title: 'Priority', value: routing.priority, short: true },
        { title: 'Assigned to', value: routing.assignedTo, short: true }
      ],
      actions: [{
        type: 'button',
        text: 'View Document',
        url: generateDocumentUrl(document.id)
      }]
    }]
  });
}
```

#### Step 6: Search and Retrieval (Day 5-6)

**Workflow: Search Documents**

```
API Endpoint: /search
    Query params: query, filters, page
    ↓
IF: Semantic search requested
    OpenAI: Create query embedding
    ↓
    Pinecone: Search vectors
    topK: 20
    ↓
ELSE: Keyword search
    PostgreSQL: Full-text search
    ↓
Apply filters:
    Category, date range, tags
    ↓
PostgreSQL: Get document metadata
    ↓
Function: Build search results
    Highlight: Matching text
    ↓
Return: Paginated results
```

**Search Implementation:**

```javascript
// Search documents
async function searchDocuments(query, options = {}) {
  const {
    searchType = 'hybrid', // 'keyword', 'semantic', or 'hybrid'
    filters = {},
    page = 1,
    pageSize = 20
  } = options;

  let results = [];

  if (searchType === 'semantic' || searchType === 'hybrid') {
    // Semantic search using embeddings
    const queryEmbedding = await createEmbedding(query);

    const vectorResults = await pinecone.query({
      vector: queryEmbedding,
      topK: pageSize * 2, // Get more for filtering
      includeMetadata: true,
      filter: buildVectorFilter(filters)
    });

    results = vectorResults.matches.map(match => ({
      documentId: match.id,
      score: match.score,
      searchType: 'semantic',
      ...match.metadata
    }));
  }

  if (searchType === 'keyword' || searchType === 'hybrid') {
    // Full-text search in PostgreSQL
    const keywordResults = await db.query(`
      SELECT
        d.id,
        d.filename,
        d.file_type,
        d.uploaded_at,
        c.category,
        c.tags,
        dc.content,
        ts_rank(dc.tsv, plainto_tsquery('english', $1)) as rank,
        ts_headline('english', dc.content, plainto_tsquery('english', $1)) as highlight
      FROM documents d
      JOIN document_content dc ON dc.document_id = d.id
      LEFT JOIN classifications c ON c.document_id = d.id
      WHERE dc.tsv @@ plainto_tsquery('english', $1)
        ${buildSQLFilter(filters)}
      ORDER BY rank DESC
      LIMIT $2 OFFSET $3
    `, [query, pageSize, (page - 1) * pageSize]);

    const keywordDocs = keywordResults.rows.map(row => ({
      documentId: row.id,
      filename: row.filename,
      category: row.category,
      tags: row.tags,
      score: row.rank,
      searchType: 'keyword',
      highlight: row.highlight
    }));

    if (searchType === 'hybrid') {
      // Merge and re-rank results
      results = mergeSearchResults(results, keywordDocs);
    } else {
      results = keywordDocs;
    }
  }

  // Get full document details
  const enrichedResults = await enrichResults(results);

  return {
    query,
    total: enrichedResults.length,
    page,
    pageSize,
    results: enrichedResults.slice(0, pageSize)
  };
}

function mergeSearchResults(semanticResults, keywordResults) {
  const merged = new Map();

  // Add semantic results
  semanticResults.forEach(result => {
    merged.set(result.documentId, {
      ...result,
      semanticScore: result.score,
      keywordScore: 0
    });
  });

  // Add/update with keyword results
  keywordResults.forEach(result => {
    if (merged.has(result.documentId)) {
      const existing = merged.get(result.documentId);
      existing.keywordScore = result.score;
      existing.highlight = result.highlight;
    } else {
      merged.set(result.documentId, {
        ...result,
        semanticScore: 0,
        keywordScore: result.score
      });
    }
  });

  // Calculate combined score and sort
  const combined = Array.from(merged.values()).map(result => ({
    ...result,
    score: (result.semanticScore * 0.6) + (result.keywordScore * 0.4)
  }));

  return combined.sort((a, b) => b.score - a.score);
}
```

### Testing Checklist

- [ ] All file formats upload successfully
- [ ] PDF text extraction working
- [ ] OCR extracts text from images
- [ ] Documents classified correctly
- [ ] Information extraction accurate
- [ ] Routing logic assigns to correct teams
- [ ] Search returns relevant results
- [ ] Semantic search working
- [ ] Keyword search working
- [ ] Filters apply correctly

### Evaluation Criteria

**Functionality (40 points):**
- Multi-format support (10 pts)
- Accurate classification (10 pts)
- Information extraction (10 pts)
- Search functionality (10 pts)

**Accuracy (25 points):**
- Classification accuracy > 85%
- Extraction accuracy > 80%
- Search relevance
- Routing correctness

**Technical Implementation (20 points):**
- Proper error handling
- Scalable architecture
- Database optimization
- API design

**User Experience (15 points):**
- Easy upload process
- Clear search results
- Helpful notifications
- Good documentation

---

## General Project Guidelines

### Submission Requirements

For each project, submit:

1. **Documentation:**
   - README with setup instructions
   - Architecture diagram
   - API documentation (if applicable)
   - User guide

2. **Code:**
   - All n8n workflows (exported JSON)
   - Custom function code
   - Database schemas
   - Configuration files

3. **Demo:**
   - Video walkthrough (5-10 minutes)
   - Screenshots of key features
   - Sample test data

4. **Analysis:**
   - Performance metrics
   - Cost analysis
   - Lessons learned
   - Future improvements

### Best Practices

1. **Error Handling:**
   - Implement try-catch in all functions
   - Log errors to database
   - Set up error notifications
   - Provide graceful degradation

2. **Security:**
   - Never log sensitive data
   - Sanitize all inputs
   - Use environment variables for secrets
   - Implement rate limiting

3. **Performance:**
   - Cache frequent AI responses
   - Use appropriate batch sizes
   - Optimize database queries
   - Monitor API usage

4. **Cost Management:**
   - Track all AI API costs
   - Set budget alerts
   - Use cheaper models when possible
   - Implement caching strategies

### Getting Help

- Review the AI Integrations Guide (01-ai-integrations-guide.md)
- Check n8n documentation and community forum
- Test prompts in OpenAI Playground first
- Ask for code review before final submission

### Timeline

- **Days 1-2:** Setup and core functionality
- **Days 3-4:** Advanced features
- **Days 5-6:** Testing, optimization, documentation
- **Day 7:** Final testing and submission

Good luck with your projects! Remember, the goal is to build practical, real-world solutions that demonstrate your understanding of AI integrations in workflow automation.
