# Exercise 0: HTTP Request Node - API Integration Mastery

## 🎯 Learning Goals
- Master the HTTP Request node in n8n
- Implement various authentication methods
- Handle API pagination effectively
- Process different response formats
- Build robust error handling for API calls

## 📋 Prerequisites
- Basic understanding of REST APIs
- Knowledge of HTTP methods (GET, POST, PUT, DELETE)
- JSON data structure comprehension

## 🔨 Task Description

Learn to integrate external APIs into your ContentFlow AI workflows using the powerful HTTP Request node. Build connections to multiple services including fal.ai for media generation and other content services.

### Part 1: Basic API Requests (20 min)

#### Step 1: Simple GET Request

```yaml
Basic API Call:
  1. Add HTTP Request node
  2. Method: GET
  3. URL: https://api.github.com/users/{username}
  4. Test with different usernames
```

#### Step 2: POST Request with Body

```json
// POST to webhook.site for testing
{
  "method": "POST",
  "url": "https://webhook.site/unique-url",
  "body": {
    "title": "Test Content",
    "description": "Created from n8n",
    "tags": ["automation", "api", "test"],
    "timestamp": "={{$now.toISO()}}"
  }
}
```

#### Step 3: Headers and Authentication

```yaml
Authentication Types:
  1. Basic Auth:
     - Username & Password
  2. Bearer Token:
     - Header: Authorization
     - Value: Bearer YOUR_TOKEN
  3. API Key:
     - Header: X-API-Key
     - Value: YOUR_API_KEY
  4. OAuth2:
     - Configure in Credentials
```

### Part 2: Advanced Configuration (25 min)

#### Request Options Setup

```javascript
// Dynamic URL construction
const baseUrl = "https://api.contentservice.com";
const endpoint = "/v1/content";
const contentId = "{{$json.id}}";

// Full URL: https://api.contentservice.com/v1/content/123
const url = `${baseUrl}${endpoint}/${contentId}`;

// Query Parameters
const queryParams = {
  limit: 10,
  offset: 0,
  sort: "created_at",
  order: "desc",
  filter: "status:published"
};

// Headers
const headers = {
  "Content-Type": "application/json",
  "Accept": "application/json",
  "X-API-Version": "2.0",
  "X-Request-ID": "{{$guid}}"
};
```

#### Body Type Configurations

```yaml
Body Content Types:
  1. JSON:
     - Standard JSON objects
     - Arrays of data
  2. Form-Data:
     - File uploads
     - Multi-part forms
  3. Form-URL Encoded:
     - Simple key-value pairs
  4. Raw:
     - XML, Plain text
     - Custom formats
  5. Binary:
     - File contents
     - Image data
```

### Part 3: Pagination Implementation (30 min)

#### Pattern 1: Response Contains Next URL

```javascript
// Pagination with Next URL
{
  "pagination": {
    "mode": "responseContainsNextUrl",
    "nextUrl": "={{$response.body.next}}",
    "limitPagesFetched": 10,
    "requestInterval": 1000
  }
}

// API Response Example:
{
  "data": [...],
  "next": "https://api.example.com/items?page=2",
  "hasMore": true
}
```

#### Pattern 2: Update Parameter in Each Request

```javascript
// Pagination with parameters
{
  "pagination": {
    "mode": "updateAParameterInEachRequest",
    "parameters": {
      "page": {
        "type": "query",
        "name": "page",
        "value": "={{$pageCount}}"
      },
      "limit": {
        "type": "query", 
        "name": "limit",
        "value": "50"
      }
    },
    "continue": "={{$response.body.hasMore}}",
    "limitPagesFetched": 20
  }
}
```

#### Pattern 3: Cursor-Based Pagination

```javascript
// Using cursor for pagination
const paginationConfig = {
  method: "GET",
  url: "https://api.example.com/items",
  qs: {
    limit: 100,
    cursor: "={{$response.body.metadata.nextCursor}}"
  },
  pagination: {
    continue: "={{$response.body.metadata.hasNext}}",
    request: {
      qs: {
        cursor: "={{$response.body.metadata.nextCursor}}"
      }
    }
  }
};
```

### Part 4: fal.ai Integration (25 min)

#### Setting up fal.ai for Media Generation

```yaml
fal.ai Configuration:
  1. Get API Key from fal.ai
  2. Configure HTTP Request node:
     Method: POST
     URL: https://fal.run/fal-ai/[model-name]
     Headers:
       - Authorization: Key YOUR_FAL_KEY
       - Content-Type: application/json
```

#### Image Generation Request

```javascript
// Generate image with fal.ai
const imageRequest = {
  method: "POST",
  url: "https://fal.run/fal-ai/flux/dev",
  headers: {
    "Authorization": "Key {{$credentials.falApiKey}}",
    "Content-Type": "application/json"
  },
  body: {
    "prompt": "{{$json.imagePrompt}}",
    "image_size": {
      "width": 1024,
      "height": 1024
    },
    "num_inference_steps": 28,
    "guidance_scale": 3.5,
    "num_images": 1,
    "enable_safety_checker": true,
    "seed": Math.floor(Math.random() * 1000000)
  }
};

// Handle async generation
const pollForResult = {
  method: "GET",
  url: "{{$json.result_url}}",
  retry: {
    maxAttempts: 10,
    waitBetween: 2000
  }
};
```

#### Video Generation with fal.ai

```javascript
// Generate video content
const videoRequest = {
  method: "POST",
  url: "https://fal.run/fal-ai/minimax/video-01",
  headers: {
    "Authorization": "Key {{$credentials.falApiKey}}"
  },
  body: {
    "prompt": "{{$json.videoDescription}}",
    "prompt_optimizer": true,
    "duration": 5,
    "aspect_ratio": "16:9"
  }
};
```

### Part 5: Error Handling & Retries (20 min)

#### Implementing Retry Logic

```javascript
// Configure retry settings
const retryConfig = {
  retry: {
    maxAttempts: 3,
    waitBetween: 1000,
    onFailedAttempt: {
      // Exponential backoff
      waitBetween: "={{1000 * Math.pow(2, $attempt)}}"
    }
  },
  continueOnFail: true,
  alwaysOutputData: true
};
```

#### Response Validation

```javascript
// Validate API response
const validateResponse = (response) => {
  // Check status code
  if (response.statusCode !== 200) {
    throw new Error(`API returned ${response.statusCode}`);
  }
  
  // Validate required fields
  const requiredFields = ['id', 'data', 'status'];
  const missingFields = requiredFields.filter(
    field => !response.body.hasOwnProperty(field)
  );
  
  if (missingFields.length > 0) {
    throw new Error(`Missing fields: ${missingFields.join(', ')}`);
  }
  
  // Validate data types
  if (!Array.isArray(response.body.data)) {
    throw new Error('Data must be an array');
  }
  
  return response.body;
};
```

### Part 6: Batch Processing (15 min)

#### Batching Multiple Requests

```yaml
Batch Configuration:
  Options:
    - Items per Batch: 10
    - Batch Interval: 500ms
  Use Cases:
    - Avoid rate limits
    - Process large datasets
    - Reduce server load
```

```javascript
// Batch processing implementation
const batchConfig = {
  batching: {
    batchSize: 10,
    batchInterval: 500
  }
};

// Process items in batches
const processBatch = async (items) => {
  const results = [];
  const batchSize = 10;
  
  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    const batchResults = await Promise.all(
      batch.map(item => makeApiCall(item))
    );
    results.push(...batchResults);
    
    // Wait between batches
    if (i + batchSize < items.length) {
      await new Promise(resolve => setTimeout(resolve, 500));
    }
  }
  
  return results;
};
```

### Part 7: Response Optimization (15 min)

#### Optimize for AI Tools

```yaml
Response Optimization (Tool Mode):
  JSON Response:
    - Field Containing Data: data.items
    - Include Fields: Selected
    - Fields: id,title,content,tags
  
  HTML Response:
    - CSS Selector: article.content
    - Return Only Content: true
    - Elements to Omit: script,style
  
  Text Response:
    - Truncate Response: true
    - Max Characters: 1000
```

#### Transform Response Data

```javascript
// Transform API response for workflow
const transformResponse = {
  responseFormat: "json",
  options: {
    response: {
      response: {
        responseFormat: "json",
        fullResponse: false
      }
    }
  },
  // Custom transformation
  transform: "={{$response.body.data.map(item => ({" +
    "id: item.id," +
    "title: item.attributes.title," +
    "content: item.attributes.body," +
    "publishedAt: item.attributes.published_at," +
    "author: item.relationships.author.data.name" +
    "}))}}"
};
```

### Part 8: Complete Integration Workflow (20 min)

#### Build Multi-API Content Pipeline

```yaml
ContentFlow API Integration:
  1. Trigger: Webhook or Schedule
  
  2. Fetch Content Ideas (API 1):
     - GET trending topics
     - Parse response
  
  3. Generate Content (API 2):
     - POST to AI service
     - Handle async generation
  
  4. Create Images (fal.ai):
     - Generate featured image
     - Generate social media images
  
  5. Publish Content (API 3):
     - POST to CMS
     - Update status
  
  6. Share on Social (API 4):
     - POST to social APIs
     - Track engagement
```

#### Complete Example with Error Handling

```javascript
// Full HTTP Request configuration
const completeRequest = {
  method: "POST",
  url: "https://api.contentplatform.com/v2/articles",
  authentication: "genericCredentialType",
  genericAuthType: "bearerToken",
  sendHeaders: true,
  headerParameters: {
    parameters: [
      {
        name: "X-API-Version",
        value: "2.0"
      },
      {
        name: "X-Client-ID",
        value: "{{$credentials.clientId}}"
      }
    ]
  },
  sendBody: true,
  bodyContentType: "json",
  specifyBody: "json",
  jsonBody: {
    title: "{{$json.title}}",
    content: "{{$json.content}}",
    featured_image: "{{$json.imageUrl}}",
    tags: "={{$json.tags}}",
    status: "draft",
    metadata: {
      ai_generated: true,
      workflow_id: "{{$workflow.id}}",
      generation_date: "={{$now.toISO()}}"
    }
  },
  options: {
    batching: {
      batchSize: 5,
      batchInterval: 1000
    },
    retry: {
      maxAttempts: 3,
      waitBetween: 2000
    },
    timeout: 30000,
    response: {
      fullResponse: true,
      neverError: false
    }
  }
};
```

## 💡 Best Practices

1. **Always use credentials** instead of hardcoding API keys
2. **Implement pagination** for large datasets
3. **Add retry logic** for unreliable APIs
4. **Validate responses** before processing
5. **Use batching** to respect rate limits
6. **Handle errors gracefully** with try-catch
7. **Log API calls** for debugging
8. **Cache responses** when appropriate

## ✅ Success Criteria

- [ ] Successfully called multiple APIs
- [ ] Implemented pagination for large datasets
- [ ] Integrated fal.ai for media generation
- [ ] Added proper error handling and retries
- [ ] Used different authentication methods
- [ ] Processed various response formats

## 🚀 Bonus Challenge

Create an "API Orchestrator" that:
1. Manages multiple API endpoints dynamically
2. Implements smart caching strategies
3. Handles rate limiting across services
4. Provides fallback APIs for failures
5. Generates API performance analytics

## 📊 Expected Output

```json
{
  "api_calls": {
    "total": 150,
    "successful": 147,
    "failed": 3,
    "retried": 8
  },
  "services_integrated": [
    "fal.ai",
    "content_api",
    "cms_platform",
    "social_media"
  ],
  "pagination": {
    "pages_fetched": 15,
    "total_items": 745
  },
  "media_generated": {
    "images": 25,
    "videos": 5
  },
  "performance": {
    "average_response_time": "245ms",
    "cache_hit_rate": "35%",
    "rate_limit_encounters": 2
  }
}
```

## 🔗 Resources

- [n8n HTTP Request Documentation](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.httprequest/)
- [fal.ai API Documentation](https://fal.ai/docs)
- [REST API Best Practices](https://restfulapi.net/resource-naming/)
- [HTTP Status Codes](https://httpstatuses.com/)

## Next Steps
With HTTP Request mastery, you can integrate any API into your ContentFlow AI workflows!
