# Exercise 14: AI Agent Memory & Persistence

## 🎯 Learning Goals
- Understand AI agent memory concepts
- Implement conversation persistence
- Configure different memory types
- Handle memory edge cases

## 📋 Prerequisites
- N8N with AI Agent node
- Claude AI or OpenAI API key
- Understanding of AI agents

## 🔨 Task Description

Build AI agents with memory capabilities to maintain conversation context and create personalized experiences.

### Part 1: Basic Memory Implementation (20 min)

#### Setting Up Simple Memory

```javascript
// N8N AI Agent Configuration with Memory
const agentConfig = {
  name: "Customer Support Agent",
  model: "claude-3-haiku",
  memory: {
    type: "simple",
    windowSize: 5, // Remember last 5 interactions
    storage: "instance" // Store in n8n instance
  },
  systemPrompt: `You are a helpful customer support agent.
    Remember customer information and previous issues discussed.
    Use conversation history to provide personalized support.`
};
```

#### Testing Memory Retention

1. **Initial Conversation**
```
User: "Hi, my name is Sarah and I ordered product #12345"
Agent: "Hello Sarah! I see you ordered product #12345. How can I help?"

User: "It hasn't arrived yet"
Agent: "I understand your product #12345 hasn't arrived, Sarah. Let me check..."

User: "What was my order number again?"
Agent: "Your order number is #12345, Sarah."
```

2. **Verify Memory Works**
- Agent remembers the name
- Agent recalls the product number
- Agent maintains context

### Part 2: Advanced Memory Types (25 min)

#### Buffer Memory Implementation

```javascript
// Buffer Memory - Fixed Size Window
class BufferMemory {
  constructor(size = 10) {
    this.buffer = [];
    this.maxSize = size;
  }
  
  add(interaction) {
    this.buffer.push({
      timestamp: Date.now(),
      user: interaction.user,
      agent: interaction.agent,
      metadata: interaction.metadata
    });
    
    // Keep only recent interactions
    if (this.buffer.length > this.maxSize) {
      this.buffer.shift(); // Remove oldest
    }
  }
  
  getContext() {
    return this.buffer.map(item => ({
      role: "user",
      content: item.user
    }, {
      role: "assistant", 
      content: item.agent
    })).flat();
  }
  
  clear() {
    this.buffer = [];
  }
}
```

#### Summary Memory Implementation

```javascript
// Summary Memory - Compress Old Conversations
class SummaryMemory {
  constructor() {
    this.currentConversation = [];
    this.summaries = [];
    this.maxConversationLength = 20;
  }
  
  async add(interaction) {
    this.currentConversation.push(interaction);
    
    // Summarize when conversation gets long
    if (this.currentConversation.length >= this.maxConversationLength) {
      const summary = await this.summarize(this.currentConversation);
      this.summaries.push(summary);
      this.currentConversation = this.currentConversation.slice(-5); // Keep last 5
    }
  }
  
  async summarize(conversation) {
    const prompt = `Summarize this conversation, keeping key facts and context:
      ${JSON.stringify(conversation)}
      
      Focus on:
      - User information
      - Main topics discussed
      - Decisions made
      - Action items`;
    
    const summary = await callClaude(prompt);
    return {
      summary: summary,
      timestamp: Date.now(),
      messageCount: conversation.length
    };
  }
  
  getContext() {
    // Combine summaries with current conversation
    const context = [];
    
    if (this.summaries.length > 0) {
      context.push({
        role: "system",
        content: `Previous conversation summaries: ${
          this.summaries.map(s => s.summary).join('\n')
        }`
      });
    }
    
    // Add current conversation
    return context.concat(this.currentConversation);
  }
}
```

### Part 3: Persistent Storage (20 min)

#### Database-Backed Memory

```javascript
// PostgreSQL Memory Storage
class PersistentMemory {
  constructor(db) {
    this.db = db;
  }
  
  async saveConversation(sessionId, interaction) {
    await this.db.query(`
      INSERT INTO conversations (
        session_id, 
        user_message, 
        agent_response, 
        timestamp,
        metadata
      ) VALUES ($1, $2, $3, $4, $5)
    `, [
      sessionId,
      interaction.user,
      interaction.agent,
      new Date(),
      JSON.stringify(interaction.metadata)
    ]);
  }
  
  async loadConversation(sessionId, limit = 10) {
    const result = await this.db.query(`
      SELECT * FROM conversations
      WHERE session_id = $1
      ORDER BY timestamp DESC
      LIMIT $2
    `, [sessionId, limit]);
    
    return result.rows.reverse(); // Chronological order
  }
  
  async searchMemory(sessionId, query) {
    // Vector search for relevant memories
    const result = await this.db.query(`
      SELECT * FROM conversations
      WHERE session_id = $1
      AND to_tsvector('english', user_message || ' ' || agent_response) 
        @@ plainto_tsquery('english', $2)
      ORDER BY timestamp DESC
      LIMIT 5
    `, [sessionId, query]);
    
    return result.rows;
  }
}
```

#### Redis Session Memory

```javascript
// Redis for Fast Session Storage
class RedisMemory {
  constructor(redis) {
    this.redis = redis;
    this.ttl = 3600; // 1 hour default
  }
  
  async store(sessionId, conversation) {
    const key = `session:${sessionId}`;
    await this.redis.setex(
      key,
      this.ttl,
      JSON.stringify(conversation)
    );
  }
  
  async retrieve(sessionId) {
    const key = `session:${sessionId}`;
    const data = await this.redis.get(key);
    return data ? JSON.parse(data) : [];
  }
  
  async append(sessionId, interaction) {
    const conversation = await this.retrieve(sessionId);
    conversation.push(interaction);
    await this.store(sessionId, conversation);
  }
  
  async extend(sessionId) {
    const key = `session:${sessionId}`;
    await this.redis.expire(key, this.ttl);
  }
}
```

### Part 4: Memory Management (20 min)

#### Smart Memory Pruning

```javascript
// Intelligent Memory Management
class SmartMemory {
  constructor() {
    this.memory = [];
    this.importance = new Map();
  }
  
  async add(interaction) {
    // Calculate importance score
    const score = await this.calculateImportance(interaction);
    
    this.memory.push(interaction);
    this.importance.set(interaction.id, score);
    
    // Prune if needed
    if (this.memory.length > 100) {
      await this.pruneMemory();
    }
  }
  
  async calculateImportance(interaction) {
    let score = 1.0;
    
    // Recent messages are more important
    const age = Date.now() - interaction.timestamp;
    score *= Math.exp(-age / (24 * 60 * 60 * 1000)); // Decay over 24h
    
    // User data is important
    if (interaction.user.match(/my name is|I am|email|phone/i)) {
      score *= 2.0;
    }
    
    // Decisions and actions are important
    if (interaction.agent.match(/confirmed|scheduled|ordered|completed/i)) {
      score *= 1.5;
    }
    
    // Problems are important
    if (interaction.user.match(/problem|issue|broken|help/i)) {
      score *= 1.8;
    }
    
    return score;
  }
  
  async pruneMemory() {
    // Sort by importance
    const sorted = this.memory.sort((a, b) => 
      this.importance.get(b.id) - this.importance.get(a.id)
    );
    
    // Keep top 50 most important
    this.memory = sorted.slice(0, 50);
    
    // Rebuild importance map
    const newImportance = new Map();
    this.memory.forEach(m => {
      newImportance.set(m.id, this.importance.get(m.id));
    });
    this.importance = newImportance;
  }
}
```

### Part 5: Privacy & Security (15 min)

#### Secure Memory Implementation

```javascript
// Privacy-Conscious Memory
class SecureMemory {
  constructor(encryption) {
    this.encryption = encryption;
    this.memory = [];
    this.piiPatterns = [
      /\b\d{3}-\d{2}-\d{4}\b/, // SSN
      /\b[A-Z0-9._%+-]+@[A-Z0-9.-]+\.[A-Z]{2,}\b/i, // Email
      /\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/, // Credit card
      /\b\d{3}[-.]?\d{3}[-.]?\d{4}\b/ // Phone
    ];
  }
  
  sanitize(text) {
    let sanitized = text;
    
    // Mask PII
    this.piiPatterns.forEach(pattern => {
      sanitized = sanitized.replace(pattern, '[REDACTED]');
    });
    
    return sanitized;
  }
  
  async store(interaction) {
    // Sanitize before storing
    const sanitized = {
      user: this.sanitize(interaction.user),
      agent: this.sanitize(interaction.agent),
      timestamp: interaction.timestamp
    };
    
    // Encrypt sensitive data
    const encrypted = await this.encryption.encrypt(
      JSON.stringify(sanitized)
    );
    
    this.memory.push({
      id: interaction.id,
      data: encrypted,
      expires: Date.now() + (7 * 24 * 60 * 60 * 1000) // 7 days
    });
  }
  
  async retrieve(id) {
    const item = this.memory.find(m => m.id === id);
    
    if (!item) return null;
    
    // Check expiration
    if (Date.now() > item.expires) {
      this.delete(id);
      return null;
    }
    
    // Decrypt
    const decrypted = await this.encryption.decrypt(item.data);
    return JSON.parse(decrypted);
  }
  
  delete(id) {
    this.memory = this.memory.filter(m => m.id !== id);
  }
  
  purgeExpired() {
    const now = Date.now();
    this.memory = this.memory.filter(m => m.expires > now);
  }
}
```

## 💡 Best Practices

1. **Memory Size**: Balance between context and performance
2. **Privacy First**: Always sanitize sensitive information
3. **Persistence Strategy**: Choose based on use case
4. **Error Handling**: Gracefully handle memory failures
5. **Testing**: Verify memory across sessions

## ✅ Success Criteria

- [ ] Agent remembers user information
- [ ] Context maintained across interactions
- [ ] Memory persists between sessions
- [ ] Privacy measures implemented
- [ ] Performance optimized

## 🚀 Bonus Challenge

Create a "Memory Analytics" system that:
1. Tracks conversation patterns
2. Identifies frequently discussed topics
3. Measures memory effectiveness
4. Suggests conversation improvements
5. Generates user insights

## 📊 Expected Output

```json
{
  "session": "sess-789",
  "memory_stats": {
    "interactions_stored": 25,
    "memory_used": "2.3KB",
    "oldest_memory": "2024-01-15T08:00:00Z",
    "importance_scores": {
      "high": 5,
      "medium": 12,
      "low": 8
    }
  },
  "conversation_test": {
    "name_recalled": true,
    "context_maintained": true,
    "preferences_remembered": true,
    "history_accessible": true
  },
  "performance": {
    "retrieval_time": "12ms",
    "storage_time": "8ms",
    "memory_type": "redis_backed"
  }
}
```

## 🔗 Resources

- [n8n AI Agent Documentation](https://docs.n8n.io/advanced-ai/intro-tutorial/)
- [Memory Patterns Guide](../../resources/memory-patterns.md)
- [Privacy Best Practices](../../resources/privacy-guide.md)

## Next Steps
Continue exploring advanced AI agent features like tool use, multi-agent collaboration, and production deployment strategies!
