# Complete Guide to AI Integrations in n8n

## Table of Contents

1. [OpenAI Integration](#openai-integration)
2. [Building AI Chatbots](#building-ai-chatbots)
3. [Document Processing with AI](#document-processing-with-ai)
4. [Sentiment Analysis](#sentiment-analysis)
5. [LangChain & RAG](#langchain--rag)
6. [Content Generation](#content-generation)
7. [Image & Audio Processing](#image--audio-processing)

---

# OpenAI Integration

## Getting Started

### Setup OpenAI Credential

1. Get API key from https://platform.openai.com/api-keys
2. In n8n: Credentials → New → OpenAI
3. Enter API key
4. Test connection

### Basic Chat Completion

```
Manual Trigger
    ↓
OpenAI Chat Model
  Model: gpt-3.5-turbo
  Messages:
    - Role: system
      Content: "You are a helpful assistant."
    - Role: user
      Content: "{{$json.userMessage}}"
    ↓
Function: Parse Response
```

```javascript
// Parse OpenAI response
const response = $input.item.json;

return {
  json: {
    message: response.choices[0].message.content,
    tokensUsed: response.usage.total_tokens,
    cost: calculateCost(response.usage)
  }
};

function calculateCost(usage) {
  // GPT-3.5-turbo pricing
  const inputCost = (usage.prompt_tokens / 1000) * 0.0015;
  const outputCost = (usage.completion_tokens / 1000) * 0.002;
  return (inputCost + outputCost).toFixed(4);
}
```

## Advanced Usage

### Function Calling

```
OpenAI Chat Model
  Model: gpt-4
  Messages:
    - Role: system
      Content: "You are a helpful assistant with access to tools."
    - Role: user
      Content: "What's the weather in San Francisco?"
  Functions:
    - name: get_weather
      description: "Get current weather for a location"
      parameters:
        type: object
        properties:
          location:
            type: string
            description: "City name"
          unit:
            type: string
            enum: ["celsius", "fahrenheit"]
        required: ["location"]
    ↓
IF: Function call requested
    ↓
    Function: Execute function
```

```javascript
// Handle function call
const response = $input.item.json;
const message = response.choices[0].message;

if (message.function_call) {
  const functionName = message.function_call.name;
  const args = JSON.parse(message.function_call.arguments);

  let result;

  switch(functionName) {
    case 'get_weather':
      result = await getWeather(args.location, args.unit);
      break;
    case 'search_database':
      result = await searchDatabase(args.query);
      break;
    // Add more functions
  }

  return {
    json: {
      functionName,
      arguments: args,
      result
    }
  };
}

return { json: message };
```

```
Function: Create follow-up message
    ↓
OpenAI Chat Model (2nd call)
  Messages:
    - [Previous messages]
    - Role: function
      Name: get_weather
      Content: {{functionResult}}
    ↓
Return: Final answer
```

### Streaming Responses

```javascript
// For long responses, use streaming
const OpenAI = require('openai');
const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY
});

async function streamChat(messages) {
  const stream = await openai.chat.completions.create({
    model: 'gpt-4',
    messages: messages,
    stream: true,
  });

  let fullResponse = '';

  for await (const chunk of stream) {
    const content = chunk.choices[0]?.delta?.content || '';
    fullResponse += content;

    // Optionally send partial updates
    await sendPartialUpdate(content);
  }

  return fullResponse;
}
```

### Embeddings

```
OpenAI: Create Embeddings
  Model: text-embedding-ada-002
  Input: {{$json.text}}
    ↓
Function: Process embedding
```

```javascript
// Store embedding in vector database
const embedding = $input.item.json.data[0].embedding;

// Store in Pinecone, Weaviate, or PostgreSQL with pgvector
await storeEmbedding({
  id: $json.documentId,
  text: $json.text,
  embedding: embedding,
  metadata: {
    source: $json.source,
    createdAt: new Date()
  }
});

return {
  json: {
    documentId: $json.documentId,
    embeddingDimensions: embedding.length,
    stored: true
  }
};
```

## Prompt Engineering

### System Prompts

```javascript
const systemPrompts = {
  customerSupport: `You are a friendly and helpful customer support agent.
    - Be empathetic and professional
    - Provide clear, concise answers
    - If you don't know something, admit it
    - Always maintain a positive tone
    - End with asking if there's anything else you can help with`,

  technicalWriter: `You are a technical writer creating documentation.
    - Use clear, simple language
    - Include code examples when relevant
    - Structure with headings and bullet points
    - Be precise and accurate
    - Consider the audience's technical level`,

  dataAnalyst: `You are a data analyst. Analyze the provided data and:
    - Identify key trends and patterns
    - Highlight anomalies or concerns
    - Provide actionable insights
    - Use numbers and percentages
    - Present findings in a structured format`
};
```

### Few-Shot Learning

```javascript
// Build prompt with examples
function buildFewShotPrompt(task, examples, input) {
  let prompt = `Task: ${task}\n\n`;

  // Add examples
  examples.forEach((ex, i) => {
    prompt += `Example ${i + 1}:\n`;
    prompt += `Input: ${ex.input}\n`;
    prompt += `Output: ${ex.output}\n\n`;
  });

  // Add actual input
  prompt += `Now process this:\n`;
  prompt += `Input: ${input}\n`;
  prompt += `Output:`;

  return prompt;
}

// Usage
const prompt = buildFewShotPrompt(
  'Extract key information from customer messages',
  [
    {
      input: 'My order #12345 hasn\'t arrived yet. I ordered it 2 weeks ago.',
      output: JSON.stringify({
        orderId: '12345',
        issue: 'delivery_delay',
        timeframe: '2 weeks',
        sentiment: 'frustrated'
      })
    },
    {
      input: 'I love the product! Can I order 5 more?',
      output: JSON.stringify({
        orderId: null,
        issue: 'reorder_request',
        quantity: 5,
        sentiment: 'positive'
      })
    }
  ],
  $json.customerMessage
);
```

### Chain of Thought

```javascript
// Encourage step-by-step reasoning
const prompt = `Solve this problem step by step:

Problem: ${$json.problem}

Let's approach this systematically:
1. First, identify what we know
2. Then, determine what we need to find
3. Next, outline the steps to solve it
4. Finally, calculate the answer

Show your work for each step.`;
```

## Cost Optimization

### Token Counting

```javascript
// Estimate tokens before API call
function estimateTokens(text) {
  // Rough estimate: 4 characters ≈ 1 token
  return Math.ceil(text.length / 4);
}

function shouldUseGPT4(prompt, response_needed) {
  const estimatedInputTokens = estimateTokens(prompt);
  const estimatedOutputTokens = response_needed ? 500 : 100;

  // Use GPT-4 only if needed and budget allows
  const gpt4Cost = ((estimatedInputTokens / 1000) * 0.03) +
                   ((estimatedOutputTokens / 1000) * 0.06);

  if (gpt4Cost > 0.10) {
    // Too expensive, use GPT-3.5
    return false;
  }

  // Complex tasks that benefit from GPT-4
  const complexPatterns = [
    'analyze', 'reason', 'explain why', 'compare',
    'evaluate', 'critique', 'code review'
  ];

  return complexPatterns.some(pattern =>
    prompt.toLowerCase().includes(pattern)
  );
}
```

### Response Caching

```
Function: Check cache
  Hash input prompt
  Query cache database
    ↓
IF: Cache hit
    Return: Cached response
ELSE:
    Call OpenAI API
    ↓
    Store in cache
    ↓
    Return: New response
```

```javascript
// Simple cache implementation
const crypto = require('crypto');

async function getCachedOrGenerate(prompt, options = {}) {
  // Create hash of prompt + options
  const cacheKey = crypto
    .createHash('sha256')
    .update(JSON.stringify({ prompt, options }))
    .digest('hex');

  // Check cache
  const cached = await getCached(cacheKey);
  if (cached && !isExpired(cached)) {
    return {
      ...cached.response,
      fromCache: true,
      cachedAt: cached.createdAt
    };
  }

  // Call AI
  const response = await callOpenAI(prompt, options);

  // Store in cache (24 hour expiry)
  await storeCache(cacheKey, response, 86400);

  return {
    ...response,
    fromCache: false
  };
}
```

---

# Building AI Chatbots

## Architecture

```
User Message (Slack/Discord/Web)
    ↓
Webhook/Trigger
    ↓
Load Conversation History
    ↓
Build Context
    ↓
OpenAI Chat Completion
    ↓
Save to History
    ↓
Send Response to User
```

## Slack Chatbot Implementation

### Webhook Handler

```
Webhook: POST /slack/events
  Body: Slack event payload
    ↓
Function: Validate Slack signature
```

```javascript
// Verify Slack request
const crypto = require('crypto');

function verifySlackRequest(headers, body) {
  const slackSignature = headers['x-slack-signature'];
  const timestamp = headers['x-slack-request-timestamp'];
  const slackSigningSecret = process.env.SLACK_SIGNING_SECRET;

  // Prevent replay attacks
  const time = Math.floor(Date.now() / 1000);
  if (Math.abs(time - timestamp) > 60 * 5) {
    throw new Error('Request too old');
  }

  // Verify signature
  const sigBasestring = `v0:${timestamp}:${body}`;
  const mySignature = 'v0=' +
    crypto.createHmac('sha256', slackSigningSecret)
          .update(sigBasestring)
          .digest('hex');

  if (mySignature !== slackSignature) {
    throw new Error('Invalid signature');
  }

  return true;
}
```

```
Function: Parse event
    ↓
IF: Event type = 'url_verification'
    Return: Challenge response
    ↓
IF: Event type = 'message' AND not bot
    → Process message
```

### Conversation Management

```
PostgreSQL: Load conversation history
  SELECT * FROM conversations
  WHERE user_id = {{userId}}
    AND channel_id = {{channelId}}
  ORDER BY created_at DESC
  LIMIT 10
    ↓
Function: Build context
```

```javascript
// Build conversation context
function buildConversationContext(history, currentMessage) {
  const messages = [
    {
      role: 'system',
      content: `You are a helpful AI assistant in a Slack workspace.
        - Be friendly and professional
        - Keep responses concise (Slack-appropriate)
        - Use emojis sparingly
        - If asked about something you don't know, say so
        - You can see the last 10 messages for context`
    }
  ];

  // Add conversation history
  history.forEach(msg => {
    messages.push({
      role: msg.role,
      content: msg.content
    });
  });

  // Add current message
  messages.push({
    role: 'user',
    content: currentMessage
  });

  return messages;
}
```

```
OpenAI: Chat completion
  Model: gpt-3.5-turbo
  Messages: {{context}}
  Max tokens: 500
  Temperature: 0.7
    ↓
Function: Process response
    ↓
PostgreSQL: Save to history
  INSERT INTO conversations
    (user_id, channel_id, role, content)
  VALUES
    ({{userId}}, {{channelId}}, 'user', {{userMessage}}),
    ({{userId}}, {{channelId}}, 'assistant', {{aiResponse}})
    ↓
Slack: Send message
  Channel: {{channelId}}
  Text: {{aiResponse}}
  Thread: {{threadTs}} (if reply)
```

### Intent Recognition

```
Function: Classify intent
```

```javascript
// Use AI to classify user intent
async function classifyIntent(message) {
  const prompt = `Classify the intent of this user message.

Possible intents:
- question: User is asking a question
- command: User wants to perform an action
- feedback: User is providing feedback
- greeting: User is saying hello/goodbye
- other: None of the above

User message: "${message}"

Respond with just the intent name.`;

  const response = await callOpenAI(prompt, {
    temperature: 0.1,  // Low temp for consistency
    max_tokens: 10
  });

  return response.trim().toLowerCase();
}
```

```
Switch: Route by intent

  Case 'question':
    → Answer with AI

  Case 'command':
    → Execute command workflow
    (e.g., "create ticket", "schedule meeting")

  Case 'feedback':
    → Store feedback
    → Thank user

  Case 'greeting':
    → Friendly greeting response

  Case 'other':
    → General AI response
```

### Context-Aware Responses

```javascript
// Add context from external systems
async function enrichContext(userId, message) {
  const context = {
    userInfo: await getUserInfo(userId),
    recentTickets: await getRecentTickets(userId),
    accountStatus: await getAccountStatus(userId)
  };

  // Add relevant context to system message
  const systemMessage = `You are a support assistant. Context about the user:
    - Name: ${context.userInfo.name}
    - Account type: ${context.accountStatus.plan}
    - Recent tickets: ${context.recentTickets.length}
    ${context.recentTickets.length > 0 ? `
    - Latest ticket: ${context.recentTickets[0].subject} (${context.recentTickets[0].status})
    ` : ''}

    Use this context when relevant to provide personalized help.`;

  return systemMessage;
}
```

## Discord Bot

Similar to Slack, but using Discord.js:

```javascript
// Discord bot using discord.js
const { Client, GatewayIntentBits } = require('discord.js');

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.MessageContent,
  ]
});

client.on('messageCreate', async (message) => {
  // Ignore bot messages
  if (message.author.bot) return;

  // Only respond to mentions or DMs
  if (!message.mentions.has(client.user) && message.channel.type !== 'DM') {
    return;
  }

  // Process with n8n webhook
  const response = await fetch('http://n8n:5678/webhook/discord-bot', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({
      userId: message.author.id,
      username: message.author.username,
      channelId: message.channel.id,
      message: message.content,
      timestamp: message.createdTimestamp
    })
  });

  const data = await response.json();

  // Send AI response
  await message.reply(data.response);
});

client.login(process.env.DISCORD_TOKEN);
```

---

# Document Processing with AI

## PDF Text Extraction

```
HTTP Request: Download PDF
  OR
  Receive webhook with PDF URL
    ↓
Code: Extract text from PDF
```

```javascript
// Extract text from PDF
const pdf = require('pdf-parse');
const axios = require('axios');

async function extractPDFText(pdfUrl) {
  // Download PDF
  const response = await axios.get(pdfUrl, {
    responseType: 'arraybuffer'
  });

  const dataBuffer = Buffer.from(response.data);

  // Parse PDF
  const data = await pdf(dataBuffer);

  return {
    text: data.text,
    pages: data.numpages,
    info: data.info
  };
}

const result = await extractPDFText($json.pdfUrl);
return { json: result };
```

## Document Summarization

```
Function: Chunk document
  (Long documents need to be split)
    ↓
OpenAI: Summarize each chunk
    ↓
Function: Combine summaries
    ↓
OpenAI: Create final summary
```

```javascript
// Chunk long document
function chunkText(text, maxTokens = 3000) {
  const chunks = [];
  const sentences = text.match(/[^.!?]+[.!?]+/g) || [text];

  let currentChunk = '';
  let currentTokens = 0;

  for (const sentence of sentences) {
    const sentenceTokens = estimateTokens(sentence);

    if (currentTokens + sentenceTokens > maxTokens) {
      if (currentChunk) {
        chunks.push(currentChunk.trim());
      }
      currentChunk = sentence;
      currentTokens = sentenceTokens;
    } else {
      currentChunk += ' ' + sentence;
      currentTokens += sentenceTokens;
    }
  }

  if (currentChunk) {
    chunks.push(currentChunk.trim());
  }

  return chunks;
}

// Summarize each chunk
async function summarizeDocument(text) {
  const chunks = chunkText(text);

  const chunkSummaries = await Promise.all(
    chunks.map(chunk => summarizeChunk(chunk))
  );

  // Combine chunk summaries
  const combinedSummary = chunkSummaries.join('\n\n');

  // Final summarization
  const finalSummary = await callOpenAI(
    `Here are summaries of different sections of a document.
     Create a cohesive overall summary (max 250 words):

     ${combinedSummary}`,
    { max_tokens: 500 }
  );

  return {
    chunkCount: chunks.length,
    chunkSummaries,
    finalSummary
  };
}

async function summarizeChunk(chunk) {
  return await callOpenAI(
    `Summarize this text in 2-3 sentences:\n\n${chunk}`,
    { max_tokens: 150 }
  );
}
```

## Information Extraction

```
OpenAI: Extract structured data
  Prompt: Extract specific fields
  Response format: JSON
```

```javascript
// Extract structured information
async function extractInformation(document, schema) {
  const prompt = `Extract the following information from this document.
    Return a JSON object with the extracted data.

    Fields to extract:
    ${JSON.stringify(schema, null, 2)}

    Document:
    ${document}

    Important:
    - Extract only factual information present in the document
    - Use null if information is not found
    - Maintain original spelling and formatting
    - Return valid JSON only

    JSON:`;

  const response = await callOpenAI(prompt, {
    temperature: 0.1,
    max_tokens: 1000
  });

  try {
    return JSON.parse(response);
  } catch (error) {
    // If JSON parsing fails, try to extract JSON from response
    const jsonMatch = response.match(/\{[\s\S]*\}/);
    if (jsonMatch) {
      return JSON.parse(jsonMatch[0]);
    }
    throw new Error('Failed to extract JSON');
  }
}

// Usage
const schema = {
  invoice_number: "string",
  date: "string (YYYY-MM-DD)",
  vendor: "string",
  total_amount: "number",
  line_items: [{
    description: "string",
    quantity: "number",
    price: "number"
  }]
};

const extracted = await extractInformation($json.documentText, schema);
```

## Document Q&A

```
User submits question about document
    ↓
Load document text
    ↓
OpenAI: Answer question
```

```javascript
// Answer questions about document
async function answerDocumentQuestion(documentText, question) {
  const prompt = `Based on the following document, answer this question.
    If the answer is not in the document, say "I cannot find this information in the document."

    Document:
    """
    ${documentText}
    """

    Question: ${question}

    Answer:`;

  const answer = await callOpenAI(prompt, {
    temperature: 0.2,  // Low temp for factual answers
    max_tokens: 500
  });

  return answer;
}

// For very long documents, use embeddings + RAG
async function answerLongDocumentQuestion(documentId, question) {
  // 1. Create embedding of question
  const questionEmbedding = await createEmbedding(question);

  // 2. Find relevant document chunks
  const relevantChunks = await findSimilarChunks(
    documentId,
    questionEmbedding,
    topK = 3
  );

  // 3. Build context from relevant chunks
  const context = relevantChunks
    .map(chunk => chunk.text)
    .join('\n\n---\n\n');

  // 4. Answer question using relevant context
  const prompt = `Answer this question based on the provided context.

    Context:
    ${context}

    Question: ${question}

    Answer (cite specific parts of the context):`;

  return await callOpenAI(prompt);
}
```

## Document Classification

```
OpenAI: Classify document
```

```javascript
// Classify documents into categories
async function classifyDocument(documentText, categories) {
  const prompt = `Classify this document into one of the following categories:

    Categories:
    ${categories.map((cat, i) => `${i + 1}. ${cat.name}: ${cat.description}`).join('\n')}

    Document excerpt:
    ${documentText.substring(0, 2000)}...

    Respond with:
    1. The category name
    2. Confidence level (high/medium/low)
    3. Brief reasoning

    Format: JSON
    {
      "category": "category name",
      "confidence": "high",
      "reasoning": "explanation"
    }`;

  const response = await callOpenAI(prompt, {
    temperature: 0.1,
    max_tokens: 200
  });

  return JSON.parse(response);
}

// Usage
const categories = [
  { name: 'invoice', description: 'Financial invoices and bills' },
  { name: 'contract', description: 'Legal contracts and agreements' },
  { name: 'resume', description: 'Job applications and CVs' },
  { name: 'report', description: 'Business or research reports' },
  { name: 'other', description: 'Other document types' }
];

const classification = await classifyDocument($json.text, categories);
```

---

# Sentiment Analysis

## Basic Sentiment Detection

```
OpenAI: Analyze sentiment
```

```javascript
// Simple sentiment analysis
async function analyzeSentiment(text) {
  const prompt = `Analyze the sentiment of this text.

    Text: "${text}"

    Provide:
    1. Overall sentiment (positive/neutral/negative)
    2. Confidence score (0-1)
    3. Key emotions detected
    4. Brief explanation

    Respond in JSON format.`;

  const response = await callOpenAI(prompt, {
    temperature: 0.2,
    max_tokens: 200
  });

  return JSON.parse(response);
}

// Example response:
// {
//   "sentiment": "negative",
//   "confidence": 0.85,
//   "emotions": ["frustration", "disappointment"],
//   "explanation": "Customer expresses frustration with delayed delivery"
// }
```

## Customer Feedback Analysis

```
Workflow: Process Customer Feedback

Webhook: New feedback received
    ↓
OpenAI: Analyze sentiment
    ↓
OpenAI: Extract key points
    ↓
Database: Store analysis
    ↓
IF: Negative sentiment with high confidence
    Create urgent ticket
    Notify support manager
    ↓
Google Sheets: Update dashboard
```

```javascript
// Comprehensive feedback analysis
async function analyzeFeedback(feedback) {
  const prompt = `Analyze this customer feedback comprehensively.

    Feedback: "${feedback}"

    Provide analysis in JSON format:
    {
      "sentiment": {
        "overall": "positive/neutral/negative",
        "score": 0.0 to 1.0 (-1 to 1 scale),
        "confidence": 0.0 to 1.0
      },
      "category": "product/service/support/billing/other",
      "priority": "low/medium/high/urgent",
      "key_points": ["point 1", "point 2"],
      "action_items": ["action 1", "action 2"],
      "customer_emotion": "frustrated/happy/confused/angry/satisfied",
      "requires_response": true/false
    }`;

  return JSON.parse(await callOpenAI(prompt, {
    temperature: 0.2,
    max_tokens: 400
  }));
}
```

## Social Media Monitoring

```
Schedule: Every 15 minutes
    ↓
Twitter/Reddit API: Fetch mentions
    ↓
For each mention:
    ↓
    OpenAI: Analyze sentiment
    ↓
    Database: Store
    ↓
    IF: Negative sentiment
        Alert social media team
        Suggest response
```

```javascript
// Monitor brand mentions
async function analyzeMention(mention) {
  const analysis = await callOpenAI(
    `Analyze this social media mention of our brand.

     Post: "${mention.text}"
     Author: ${mention.author}
     Platform: ${mention.platform}

     Analyze:
     1. Sentiment towards our brand
     2. Is this a complaint, question, praise, or neutral mention?
     3. Does this require a response?
     4. Suggested response tone (if response needed)
     5. Priority level

     JSON format:`,
    { temperature: 0.2 }
  );

  const parsed = JSON.parse(analysis);

  // Generate suggested response if needed
  if (parsed.requires_response) {
    parsed.suggested_response = await generateResponse(
      mention.text,
      parsed.sentiment,
      parsed.response_tone
    );
  }

  return parsed;
}
```

---

# LangChain & RAG

## RAG (Retrieval Augmented Generation)

### Architecture

```
User Question
    ↓
Create Embedding of Question
    ↓
Search Vector Database
    ↓
Retrieve Top K Similar Documents
    ↓
Build Prompt with Context
    ↓
OpenAI Generate Answer
    ↓
Return Answer + Sources
```

### Implementation

```javascript
// Complete RAG implementation
const { OpenAI } = require('openai');
const { PineconeClient } = require('@pinecone-database/pinecone');

class RAGSystem {
  constructor() {
    this.openai = new OpenAI();
    this.pinecone = new PineconeClient();
  }

  async initialize() {
    await this.pinecone.init({
      apiKey: process.env.PINECONE_API_KEY,
      environment: process.env.PINECONE_ENVIRONMENT
    });
    this.index = this.pinecone.Index('knowledge-base');
  }

  // Add document to knowledge base
  async addDocument(document) {
    // 1. Chunk document
    const chunks = this.chunkDocument(document.text);

    // 2. Create embeddings for each chunk
    const embeddings = await this.createEmbeddings(chunks);

    // 3. Store in vector database
    const vectors = chunks.map((chunk, i) => ({
      id: `${document.id}_chunk_${i}`,
      values: embeddings[i],
      metadata: {
        text: chunk,
        documentId: document.id,
        title: document.title,
        source: document.source
      }
    }));

    await this.index.upsert({ vectors });

    return { chunksCreated: chunks.length };
  }

  // Answer question using RAG
  async answerQuestion(question) {
    // 1. Create embedding of question
    const questionEmbedding = await this.createEmbedding(question);

    // 2. Search for similar chunks
    const searchResults = await this.index.query({
      vector: questionEmbedding,
      topK: 5,
      includeMetadata: true
    });

    // 3. Extract relevant context
    const context = searchResults.matches
      .map(match => match.metadata.text)
      .join('\n\n---\n\n');

    const sources = searchResults.matches.map(match => ({
      title: match.metadata.title,
      source: match.metadata.source,
      similarity: match.score
    }));

    // 4. Generate answer with context
    const prompt = `Answer the question based on the context below. Be specific and cite the sources.

      Context:
      ${context}

      Question: ${question}

      Answer:`;

    const answer = await this.openai.chat.completions.create({
      model: 'gpt-4',
      messages: [
        {
          role: 'system',
          content: 'You are a helpful assistant that answers questions based on provided context. Always cite your sources.'
        },
        {
          role: 'user',
          content: prompt
        }
      ],
      temperature: 0.3,
      max_tokens: 800
    });

    return {
      answer: answer.choices[0].message.content,
      sources: sources,
      confidence: this.calculateConfidence(searchResults)
    };
  }

  chunkDocument(text, maxChunkSize = 1000) {
    // Split into sentences
    const sentences = text.match(/[^.!?]+[.!?]+/g) || [text];
    const chunks = [];
    let currentChunk = '';

    for (const sentence of sentences) {
      if ((currentChunk + sentence).length > maxChunkSize) {
        if (currentChunk) chunks.push(currentChunk.trim());
        currentChunk = sentence;
      } else {
        currentChunk += ' ' + sentence;
      }
    }

    if (currentChunk) chunks.push(currentChunk.trim());
    return chunks;
  }

  async createEmbedding(text) {
    const response = await this.openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: text
    });
    return response.data[0].embedding;
  }

  async createEmbeddings(texts) {
    const response = await this.openai.embeddings.create({
      model: 'text-embedding-ada-002',
      input: texts
    });
    return response.data.map(d => d.embedding);
  }

  calculateConfidence(searchResults) {
    if (searchResults.matches.length === 0) return 0;

    const topScore = searchResults.matches[0].score;
    const avgScore = searchResults.matches.reduce((sum, m) => sum + m.score, 0) / searchResults.matches.length;

    // Confidence based on top match score and consistency
    return (topScore * 0.7 + avgScore * 0.3);
  }
}

// Usage in n8n workflow
const rag = new RAGSystem();
await rag.initialize();

// Add document
await rag.addDocument({
  id: '123',
  title: 'Product Documentation',
  text: $json.documentText,
  source: 'docs.example.com'
});

// Answer question
const result = await rag.answerQuestion($json.question);
return { json: result };
```

## LangChain Chains

```javascript
// Using LangChain for complex chains
const { ChatOpenAI } = require("langchain/chat_models/openai");
const { PromptTemplate } = require("langchain/prompts");
const { LLMChain } = require("langchain/chains");

// Multi-step chain example
async function researchAndSummarize(topic) {
  const model = new ChatOpenAI({
    temperature: 0.7,
    modelName: "gpt-4"
  });

  // Step 1: Generate research questions
  const questionPrompt = new PromptTemplate({
    template: "Generate 3 key research questions about: {topic}",
    inputVariables: ["topic"]
  });

  const questionChain = new LLMChain({
    llm: model,
    prompt: questionPrompt
  });

  const questions = await questionChain.call({ topic });

  // Step 2: Research each question
  // (In real scenario, this would search databases or web)

  // Step 3: Synthesize findings
  const synthesisPrompt = new PromptTemplate({
    template: `Based on these research findings, create a comprehensive summary about {topic}:

      {findings}

      Summary:`,
    inputVariables: ["topic", "findings"]
  });

  const synthesisChain = new LLMChain({
    llm: model,
    prompt: synthesisPrompt
  });

  const summary = await synthesisChain.call({
    topic,
    findings: questions.text
  });

  return summary.text;
}
```

---

# Content Generation

## Blog Post Creation

```
Workflow: Generate Blog Post

Input: Topic, target audience, keywords
    ↓
OpenAI: Generate outline
    ↓
For each section:
    OpenAI: Write section content
    ↓
OpenAI: Create introduction
    ↓
OpenAI: Write conclusion
    ↓
Function: Combine all parts
    ↓
OpenAI: Generate meta description
    ↓
Output: Complete blog post
```

```javascript
// Blog post generator
async function generateBlogPost(topic, keywords, audience) {
  // 1. Generate outline
  const outline = await callOpenAI(
    `Create a detailed blog post outline about "${topic}".

     Target audience: ${audience}
     Include these keywords: ${keywords.join(', ')}

     Format:
     1. Catchy title
     2. Introduction hook
     3. Main sections (4-5 sections with subsections)
     4. Conclusion
     5. Call to action

     Provide the outline:`,
    { temperature: 0.8 }
  );

  // 2. Write each section
  const sections = parseOutline(outline);
  const content = [];

  for (const section of sections) {
    const sectionContent = await callOpenAI(
      `Write the "${section.title}" section for a blog post about "${topic}".

       Context: ${section.context}
       Target audience: ${audience}
       Tone: Professional but conversational
       Length: 200-300 words

       Include:
       - Specific examples
       - Actionable insights
       - Natural keyword usage: ${keywords.join(', ')}

       Content:`,
      {
        temperature: 0.7,
        max_tokens: 600
      }
    );

    content.push({
      title: section.title,
      content: sectionContent
    });
  }

  // 3. Create meta description
  const metaDescription = await callOpenAI(
    `Write a compelling meta description (max 160 characters) for a blog post about "${topic}".

     Include main keyword: ${keywords[0]}

     Meta description:`,
    {
      temperature: 0.6,
      max_tokens: 100
    }
  );

  return {
    title: sections[0].title,
    metaDescription,
    content,
    fullText: content.map(s => `## ${s.title}\n\n${s.content}`).join('\n\n'),
    wordCount: calculateWordCount(content)
  };
}
```

## Marketing Copy

```javascript
// Generate marketing copy variations
async function generateMarketingCopy(product, platform, goal) {
  const prompt = `Create compelling marketing copy for this product:

    Product: ${product.name}
    Description: ${product.description}
    Key benefits: ${product.benefits.join(', ')}

    Platform: ${platform} (${getPlatformConstraints(platform)})
    Goal: ${goal}

    Generate 5 variations with different angles:
    1. Benefit-focused
    2. Problem-solving
    3. Social proof
    4. Urgency/scarcity
    5. Value proposition

    Format as JSON array.`;

  const variations = await callOpenAI(prompt, {
    temperature: 0.9,  // Higher for creativity
    max_tokens: 1000
  });

  return JSON.parse(variations);
}

function getPlatformConstraints(platform) {
  const constraints = {
    twitter: 'max 280 characters',
    facebook: 'max 125 characters for link descriptions',
    instagram: 'max 2200 characters, use emojis',
    linkedin: 'professional tone, max 3000 characters',
    email_subject: 'max 50 characters'
  };

  return constraints[platform] || 'no specific constraints';
}
```

## Translation Pipeline

```
Input: Content + target languages
    ↓
For each language:
    ↓
    OpenAI: Translate
    ↓
    OpenAI: Review translation quality
    ↓
    IF quality score < 0.8:
        Retry with different prompt
    ↓
    Store translation
```

```javascript
// Translation with quality control
async function translateContent(content, targetLanguage) {
  // 1. Translate
  const translation = await callOpenAI(
    `Translate this text to ${targetLanguage}.
     Maintain tone, style, and formatting.
     Preserve any markdown formatting.

     Original text:
     ${content}

     ${targetLanguage} translation:`,
    {
      temperature: 0.3,  // Low for accuracy
      max_tokens: content.length * 2
    }
  );

  // 2. Quality check
  const qualityCheck = await callOpenAI(
    `Rate the quality of this translation (0-1 scale):

     Original (English): ${content}
     Translation (${targetLanguage}): ${translation}

     Consider:
     - Accuracy of meaning
     - Natural flow in target language
     - Preservation of tone
     - Grammar and spelling

     Respond with JSON:
     {
       "score": 0.0-1.0,
       "issues": ["issue 1", "issue 2"],
       "suggestions": "improvement suggestions"
     }`,
    { temperature: 0.2 }
  );

  const quality = JSON.parse(qualityCheck);

  return {
    translation,
    language: targetLanguage,
    qualityScore: quality.score,
    issues: quality.issues,
    verified: quality.score >= 0.8
  };
}
```

---

# Image & Audio Processing

## Image Analysis

```javascript
// Analyze image with GPT-4 Vision
async function analyzeImage(imageUrl) {
  const response = await this.openai.chat.completions.create({
    model: "gpt-4-vision-preview",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: "Describe this image in detail. Include: main subjects, setting, colors, mood, and any text visible."
          },
          {
            type: "image_url",
            image_url: {
              url: imageUrl
            }
          }
        ]
      }
    ],
    max_tokens: 500
  });

  return response.choices[0].message.content;
}

// Product image analysis
async function analyzeProductImage(imageUrl) {
  const response = await this.openai.chat.completions.create({
    model: "gpt-4-vision-preview",
    messages: [
      {
        role: "user",
        content: [
          {
            type: "text",
            text: `Analyze this product image and extract:
              1. Product category
              2. Key features visible
              3. Colors
              4. Condition (new/used)
              5. Suggested product title
              6. Suggested description

              Respond in JSON format.`
          },
          {
            type: "image_url",
            image_url: { url: imageUrl }
          }
        ]
      }
    ],
    max_tokens: 400
  });

  return JSON.parse(response.choices[0].message.content);
}
```

## Image Generation with DALL-E

```
OpenAI: Generate image
  Model: dall-e-3
  Prompt: {{description}}
  Size: 1024x1024
  Quality: standard/hd
  ↓
Download image
  ↓
Upload to storage (S3, Cloudinary)
  ↓
Return public URL
```

```javascript
// Generate image with DALL-E
async function generateImage(description, style = 'natural') {
  const enhancedPrompt = enhanceImagePrompt(description, style);

  const response = await this.openai.images.generate({
    model: "dall-e-3",
    prompt: enhancedPrompt,
    n: 1,
    size: "1024x1024",
    quality: "standard"  // or "hd" for better quality
  });

  const imageUrl = response.data[0].url;

  // Download and store permanently
  const permanentUrl = await downloadAndStore(imageUrl);

  return {
    url: permanentUrl,
    prompt: enhancedPrompt,
    revisedPrompt: response.data[0].revised_prompt
  };
}

function enhanceImagePrompt(description, style) {
  const styleModifiers = {
    natural: 'photorealistic, natural lighting',
    artistic: 'artistic, creative interpretation',
    minimalist: 'minimalist style, clean lines',
    vintage: 'vintage aesthetic, retro colors',
    professional: 'professional product photography'
  };

  return `${description}, ${styleModifiers[style] || styleModifiers.natural}`;
}
```

## Audio Transcription with Whisper

```
HTTP Request: Download audio file
  OR
  Receive webhook with audio URL
    ↓
OpenAI: Transcribe with Whisper
  Model: whisper-1
  File: audio file
  Language: en (optional)
    ↓
Return: Transcription
```

```javascript
// Transcribe audio
async function transcribeAudio(audioUrl) {
  // Download audio file
  const audioFile = await downloadFile(audioUrl);

  const response = await this.openai.audio.transcriptions.create({
    file: audioFile,
    model: "whisper-1",
    language: "en",  // optional
    response_format: "verbose_json"  // includes timestamps
  });

  return {
    text: response.text,
    duration: response.duration,
    segments: response.segments  // timestamped segments
  };
}

// Transcribe and summarize
async function transcribeAndSummarize(audioUrl) {
  const transcription = await transcribeAudio(audioUrl);

  const summary = await callOpenAI(
    `Summarize this transcript of a ${transcription.duration}s audio:

     Transcript:
     ${transcription.text}

     Provide:
     1. Main topics discussed
     2. Key points (bullet list)
     3. Action items (if any)
     4. Overall summary (2-3 sentences)

     JSON format:`,
    { temperature: 0.3 }
  );

  return {
    transcription: transcription.text,
    duration: transcription.duration,
    analysis: JSON.parse(summary)
  };
}
```

---

## Cost Monitoring

```
Workflow: Track AI Usage

After each AI API call:
    ↓
PostgreSQL: Log usage
  INSERT INTO ai_usage_log
    (workflow_id, model, input_tokens, output_tokens, cost, timestamp)
    ↓
Function: Calculate running total
    ↓
IF: Daily spend > threshold
    Slack: Alert team
    Email: Usage report
```

```javascript
// Cost tracking
async function logAIUsage(callDetails) {
  const cost = calculateCost(
    callDetails.model,
    callDetails.usage.prompt_tokens,
    callDetails.usage.completion_tokens
  );

  await db.query(`
    INSERT INTO ai_usage_log
      (workflow_id, model, input_tokens, output_tokens, cost, timestamp)
    VALUES ($1, $2, $3, $4, $5, NOW())
  `, [
    callDetails.workflowId,
    callDetails.model,
    callDetails.usage.prompt_tokens,
    callDetails.usage.completion_tokens,
    cost
  ]);

  // Check daily spend
  const dailySpend = await getDailySpend();
  if (dailySpend > process.env.DAILY_BUDGET) {
    await alertTeam('AI budget exceeded', dailySpend);
  }

  return { cost, dailySpend };
}

function calculateCost(model, inputTokens, outputTokens) {
  const pricing = {
    'gpt-4': { input: 0.03, output: 0.06 },
    'gpt-3.5-turbo': { input: 0.0015, output: 0.002 },
    'text-embedding-ada-002': { input: 0.0001, output: 0 }
  };

  const rates = pricing[model] || pricing['gpt-3.5-turbo'];

  return (
    (inputTokens / 1000) * rates.input +
    (outputTokens / 1000) * rates.output
  );
}
```

This comprehensive guide covers all major AI integration topics for Week 14. Students can use these examples as building blocks for their hands-on projects!