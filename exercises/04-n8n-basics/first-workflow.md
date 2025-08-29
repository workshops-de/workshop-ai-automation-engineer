# Exercise 4.1: Building Your First Content Generation Workflow

## 🎯 Learning Goals
- Create your first N8N workflow
- Connect nodes and pass data
- Implement error handling
- Test and debug workflows

## 📋 Prerequisites
- N8N installed and running (http://localhost:5678)
- Claude AI API key
- Basic understanding of workflow concepts

## 🔨 Task Description

Build a complete blog post generation workflow in N8N that takes a topic and creates a full article with title, content, and metadata.

### Part 1: Workflow Setup (10 min)

#### Step 1: Create New Workflow
1. Open N8N in your browser
2. Click "New Workflow"
3. Name it: "ContentFlow - Blog Generator v1"
4. Add description: "Automated blog post generation from topic"

#### Step 2: Add Webhook Trigger
```json
// Test payload for webhook
{
  "topic": "The Future of AI in Marketing",
  "keywords": ["AI", "marketing", "automation"],
  "targetLength": 800,
  "audience": "Marketing professionals"
}
```

### Part 2: Building the Flow (30 min)

#### Workflow Architecture
```
Webhook → Set Variables → Claude AI → Format Output → Response
           ↓ (on error)
        Error Handler
```

#### Node 1: Webhook
- Type: Webhook
- Method: POST
- Path: /generate-blog
- Response Mode: "On last node"

#### Node 2: Set Variables
Configure variables for the workflow:
```javascript
// Function Node
return {
  topic: $input.item.json.topic,
  keywords: $input.item.json.keywords.join(', '),
  length: $input.item.json.targetLength || 800,
  audience: $input.item.json.audience || 'general',
  timestamp: new Date().toISOString()
};
```

#### Node 3: Claude AI Integration
HTTP Request node configuration:
```javascript
// URL
https://api.anthropic.com/v1/messages

// Headers
{
  "x-api-key": "YOUR_API_KEY",
  "anthropic-version": "2023-06-01",
  "content-type": "application/json"
}

// Body
{
  "model": "claude-3-haiku-20240307",
  "max_tokens": 2000,
  "system": "You are a professional blog writer...",
  "messages": [{
    "role": "user",
    "content": "Write a blog post about {{$json.topic}}"
  }]
}
```

#### Node 4: Parse AI Response
Extract and structure the content:
```javascript
// Function Node
const response = $input.item.json;
const content = response.content[0].text;

// Parse sections (assuming markdown format)
const lines = content.split('\n');
const title = lines[0].replace('#', '').trim();
const sections = [];
let currentSection = '';

for (let i = 1; i < lines.length; i++) {
  if (lines[i].startsWith('##')) {
    if (currentSection) sections.push(currentSection);
    currentSection = lines[i];
  } else {
    currentSection += '\n' + lines[i];
  }
}

return {
  title: title,
  content: content,
  sections: sections,
  wordCount: content.split(' ').length,
  generatedAt: new Date().toISOString()
};
```

#### Node 5: Error Handling
Add IF node to check for errors:
```javascript
// Condition
{{ $json.error !== undefined }}

// True branch: Send error notification
// False branch: Continue to output
```

### Part 3: Advanced Features (20 min)

#### Add Content Enhancement
1. **SEO Optimization Node**
```javascript
// Add meta description
const title = $json.title;
const content = $json.content;

return {
  ...$json,
  metaDescription: content.substring(0, 155) + "...",
  slug: title.toLowerCase().replace(/\s+/g, '-'),
  readingTime: Math.ceil($json.wordCount / 200)
};
```

2. **Keyword Density Check**
```javascript
// Check if keywords appear in content
const keywords = $node["Set Variables"].json.keywords.split(',');
const content = $json.content.toLowerCase();

const keywordDensity = keywords.map(keyword => {
  const regex = new RegExp(keyword.trim(), 'gi');
  const matches = content.match(regex);
  return {
    keyword: keyword.trim(),
    count: matches ? matches.length : 0,
    density: matches ? (matches.length / $json.wordCount * 100).toFixed(2) + '%' : '0%'
  };
});

return {
  ...$json,
  keywordAnalysis: keywordDensity
};
```

### Part 4: Testing & Debugging (15 min)

#### Test Cases
1. **Valid Input Test**
   - Send proper webhook request
   - Verify blog post generation
   - Check word count accuracy

2. **Missing Parameters Test**
   - Send request without topic
   - Verify error handling

3. **API Failure Simulation**
   - Use invalid API key
   - Check fallback behavior

#### Debug Checklist
- [ ] Enable "Save Execution" for debugging
- [ ] Check node outputs at each step
- [ ] Monitor execution time
- [ ] Verify data transformations
- [ ] Test error notifications

### Part 5: Workflow Optimization (15 min)

#### Performance Improvements
1. **Add Caching**
```javascript
// Check if similar content exists
const cache = $getWorkflowStaticData('cache') || {};
const cacheKey = $json.topic.toLowerCase().replace(/\s+/g, '-');

if (cache[cacheKey] && Date.now() - cache[cacheKey].timestamp < 3600000) {
  return cache[cacheKey].data;
}
```

2. **Implement Rate Limiting**
```javascript
// Track API calls
const rateLimiter = $getWorkflowStaticData('rateLimiter') || {
  calls: [],
  limit: 100
};

const now = Date.now();
rateLimiter.calls = rateLimiter.calls.filter(time => now - time < 60000);

if (rateLimiter.calls.length >= rateLimiter.limit) {
  throw new Error('Rate limit exceeded. Please try again later.');
}

rateLimiter.calls.push(now);
$setWorkflowStaticData('rateLimiter', rateLimiter);
```

## 💡 Pro Tips

- Use "Pin Data" feature for testing without API calls
- Implement circuit breakers for external services
- Add logging nodes for monitoring
- Use environment variables for API keys
- Test with different input variations

## ✅ Success Criteria

- [ ] Workflow generates complete blog posts
- [ ] Error handling works correctly
- [ ] Response time under 30 seconds
- [ ] Keyword density tracked
- [ ] Metadata generated automatically

## 🚀 Bonus Challenge

Extend the workflow to:
1. Generate a featured image prompt
2. Create social media posts from the blog
3. Translate to multiple languages
4. Save to a database
5. Send notification on completion

## 📊 Expected Output

```json
{
  "success": true,
  "data": {
    "title": "The Future of AI in Marketing",
    "slug": "the-future-of-ai-in-marketing",
    "content": "Full blog post content...",
    "metaDescription": "Discover how AI is transforming...",
    "wordCount": 823,
    "readingTime": 4,
    "keywordAnalysis": [
      {"keyword": "AI", "count": 12, "density": "1.46%"},
      {"keyword": "marketing", "count": 8, "density": "0.97%"}
    ],
    "generatedAt": "2024-01-15T10:30:00Z"
  }
}
```

## 🔗 Resources

- [N8N Documentation](https://docs.n8n.io)
- [Claude API Reference](https://docs.anthropic.com)
- [Workflow Templates](../../resources/n8n-templates/)

## 📝 Solution

<details>
<summary>Click to reveal complete workflow JSON</summary>

```json
{
  "name": "ContentFlow - Blog Generator v1",
  "nodes": [
    {
      "id": "webhook",
      "type": "n8n-nodes-base.webhook",
      "position": [250, 300],
      "parameters": {
        "path": "generate-blog",
        "responseMode": "onLastNode",
        "options": {}
      }
    },
    {
      "id": "setVariables",
      "type": "n8n-nodes-base.function",
      "position": [450, 300],
      "parameters": {
        "functionCode": "// Workflow variables setup code here"
      }
    },
    {
      "id": "claudeAI",
      "type": "n8n-nodes-base.httpRequest",
      "position": [650, 300],
      "parameters": {
        "url": "https://api.anthropic.com/v1/messages",
        "method": "POST",
        "authentication": "genericCredentialType",
        "genericAuthType": "httpHeaderAuth"
      }
    }
  ],
  "connections": {
    "webhook": {
      "main": [["setVariables"]]
    },
    "setVariables": {
      "main": [["claudeAI"]]
    }
  }
}
```

</details>

## Next Exercise
[Exercise 5.1: Multi-Agent Content System →](../05-first-workflow/multi-agent-system.md)
