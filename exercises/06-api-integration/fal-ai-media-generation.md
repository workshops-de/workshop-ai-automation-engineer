# Exercise 6.1: Media Generation with fal.ai

## 🎯 Learning Goals
- Integrate fal.ai API for image generation
- Implement video and audio creation
- Handle async generation workflows
- Optimize for cost and quality

## 📋 Prerequisites
- fal.ai API key (get from https://fal.ai)
- N8N workflow basics completed
- Understanding of REST APIs

## 🔨 Task Description

Build a complete media generation pipeline using fal.ai that creates images, videos, and audio for your content campaigns.

### Part 1: fal.ai Setup (10 min)

#### Get Your API Credentials
1. Sign up at https://fal.ai
2. Navigate to API Keys section
3. Generate new API key
4. Note your rate limits and pricing

#### Test API Connection
```bash
curl -X POST https://fal.run/fal-ai/flux/dev \
  -H "Authorization: Key YOUR_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "A modern office workspace with AI robots collaborating with humans"
  }'
```

### Part 2: Image Generation Agent (25 min)

#### Build Image Generation Workflow

##### Node 1: Parse Content Requirements
```javascript
// Extract image requirements from blog post
const content = $json.blogContent;
const title = $json.title;

// Generate image prompts based on content
const prompts = {
  hero: `Professional blog header image for article about ${title}, modern, clean design, high quality`,
  social: `Social media image for ${title}, eye-catching, vibrant colors, 1200x630px`,
  thumbnail: `Thumbnail image for ${title}, clear focal point, readable at small size`
};

return {
  prompts: prompts,
  style: "photorealistic",
  model: "flux-dev"  // or "flux-schnell" for faster generation
};
```

##### Node 2: Call fal.ai Image API
```javascript
// HTTP Request Configuration
{
  method: "POST",
  url: "https://fal.run/fal-ai/flux/dev",
  headers: {
    "Authorization": "Key {{$credentials.falApiKey}}",
    "Content-Type": "application/json"
  },
  body: {
    "prompt": "{{$json.prompts.hero}}",
    "image_size": "landscape_16_9",
    "num_inference_steps": 28,
    "guidance_scale": 3.5,
    "num_images": 1,
    "enable_safety_checker": true
  }
}
```

##### Node 3: Handle Async Generation
```javascript
// Check generation status
const requestId = $json.request_id;

// Poll for completion (implement with Wait node)
const checkStatus = async () => {
  const response = await $http.get({
    url: `https://fal.run/status/${requestId}`,
    headers: {
      "Authorization": `Key ${$credentials.falApiKey}`
    }
  });
  
  return response.status === 'completed' ? response.result : null;
};

// Implement retry logic
const maxRetries = 30;
const retryDelay = 2000; // 2 seconds

for (let i = 0; i < maxRetries; i++) {
  const result = await checkStatus();
  if (result) return result;
  await new Promise(resolve => setTimeout(resolve, retryDelay));
}

throw new Error('Image generation timeout');
```

### Part 3: Video Generation (20 min)

#### Create Video Content with fal.ai

##### Video Generation Request
```javascript
// Generate video from script
{
  method: "POST",
  url: "https://fal.run/fal-ai/kling-video/v1/standard/text-to-video",
  headers: {
    "Authorization": "Key {{$credentials.falApiKey}}"
  },
  body: {
    "prompt": "{{$json.videoScript}}",
    "duration": 5,  // 5 or 10 seconds
    "aspect_ratio": "16:9",
    "style": "professional"
  }
}
```

##### Handle Video Processing
```javascript
// Video generation takes longer, implement proper waiting
const videoWorkflow = {
  maxWaitTime: 300000,  // 5 minutes
  checkInterval: 5000,   // Check every 5 seconds
  
  async process(requestId) {
    const startTime = Date.now();
    
    while (Date.now() - startTime < this.maxWaitTime) {
      const status = await this.checkStatus(requestId);
      
      if (status.completed) {
        return {
          videoUrl: status.result.url,
          duration: status.result.duration,
          resolution: status.result.resolution
        };
      }
      
      await this.wait(this.checkInterval);
    }
    
    throw new Error('Video generation timeout');
  }
};
```

### Part 4: Audio/Voice Generation (20 min)

#### Text-to-Speech Implementation

##### Generate Voiceover
```javascript
// Create voiceover for video
{
  method: "POST",
  url: "https://fal.run/fal-ai/playai/tts",
  body: {
    "text": "{{$json.scriptText}}",
    "voice": "professional-male",  // or choose specific voice
    "speed": 1.0,
    "emotion": "neutral",
    "format": "mp3"
  }
}
```

##### Sync Audio with Video
```javascript
// Coordinate audio and video generation
const mediaCoordinator = {
  async generateMediaSet(content) {
    // Parallel generation for efficiency
    const [image, video, audio] = await Promise.all([
      this.generateImage(content.imagePrompt),
      this.generateVideo(content.videoScript),
      this.generateAudio(content.voiceoverText)
    ]);
    
    return {
      image: image,
      video: video,
      audio: audio,
      generatedAt: new Date().toISOString()
    };
  }
};
```

### Part 5: Stock Photo Integration (15 min)

#### Integrate Unsplash/Pexels as Fallback

##### Unsplash API Integration
```javascript
// When AI generation isn't needed, use stock photos
{
  method: "GET",
  url: "https://api.unsplash.com/search/photos",
  headers: {
    "Authorization": "Client-ID {{$credentials.unsplashAccessKey}}"
  },
  qs: {
    "query": "{{$json.searchQuery}}",
    "per_page": 5,
    "orientation": "landscape"
  }
}
```

##### Smart Media Selection
```javascript
// Decide between AI generation and stock photos
function selectMediaStrategy(requirements) {
  const strategies = {
    custom: requirements.brandSpecific || requirements.unique,
    stock: requirements.generic && !requirements.brandSpecific,
    hybrid: requirements.speed === 'fast'
  };
  
  if (strategies.custom) {
    return {
      method: 'ai_generation',
      provider: 'fal.ai',
      model: 'flux-dev'
    };
  } else if (strategies.stock) {
    return {
      method: 'stock_photo',
      provider: 'unsplash',
      fallback: 'pexels'
    };
  }
  
  // Hybrid: Try stock first, generate if not found
  return {
    method: 'hybrid',
    primary: 'stock_photo',
    fallback: 'ai_generation'
  };
}
```

### Part 6: Cost Optimization (10 min)

#### Implement Cost Tracking
```javascript
// Track API usage and costs
const costTracker = {
  falai: {
    flux_dev: 0.025,      // $ per image
    flux_schnell: 0.003,  // $ per image
    video_5s: 0.50,       // $ per video
    audio_minute: 0.10    // $ per minute
  },
  
  trackUsage(service, model, quantity = 1) {
    const cost = this[service][model] * quantity;
    
    // Store in workflow static data
    const usage = $getWorkflowStaticData('usage') || {
      total: 0,
      breakdown: {}
    };
    
    usage.total += cost;
    usage.breakdown[model] = (usage.breakdown[model] || 0) + quantity;
    
    $setWorkflowStaticData('usage', usage);
    
    return {
      cost: cost,
      totalToDate: usage.total
    };
  }
};
```

## 💡 Best Practices

1. **Caching Generated Media**
```javascript
// Cache media URLs to avoid regeneration
const mediaCache = {
  set(prompt, url) {
    const cache = $getWorkflowStaticData('mediaCache') || {};
    const key = crypto.createHash('md5').update(prompt).digest('hex');
    cache[key] = { url, timestamp: Date.now() };
    $setWorkflowStaticData('mediaCache', cache);
  },
  
  get(prompt) {
    const cache = $getWorkflowStaticData('mediaCache') || {};
    const key = crypto.createHash('md5').update(prompt).digest('hex');
    const cached = cache[key];
    
    // Cache for 24 hours
    if (cached && Date.now() - cached.timestamp < 86400000) {
      return cached.url;
    }
    return null;
  }
};
```

2. **Error Handling**
- Implement retry logic for failed generations
- Have fallback options (different models, stock photos)
- Monitor rate limits
- Log all API calls for debugging

## ✅ Success Criteria

- [ ] Successfully generated images with fal.ai
- [ ] Created at least one video
- [ ] Implemented audio generation
- [ ] Integrated stock photo fallback
- [ ] Cost tracking implemented

## 🚀 Bonus Challenge

Create a "Smart Media Director" that:
1. Analyzes content to determine media needs
2. Chooses optimal generation strategy
3. Generates multiple variations
4. A/B tests different styles
5. Learns from engagement metrics

## 📊 Expected Output

```json
{
  "campaign": "AI Marketing Future",
  "media": {
    "images": {
      "hero": "https://fal.ai/generated/hero-image.jpg",
      "social": "https://fal.ai/generated/social-image.jpg",
      "thumbnail": "https://fal.ai/generated/thumbnail.jpg"
    },
    "video": {
      "url": "https://fal.ai/generated/promo-video.mp4",
      "duration": 10,
      "resolution": "1920x1080"
    },
    "audio": {
      "voiceover": "https://fal.ai/generated/voiceover.mp3",
      "duration": 45
    }
  },
  "costs": {
    "images": 0.075,
    "video": 0.50,
    "audio": 0.075,
    "total": 0.65
  },
  "generationTime": 47.3
}
```

## 🔗 Resources

- [fal.ai Documentation](https://fal.ai/docs)
- [fal.ai Model Gallery](https://fal.ai/models)
- [Unsplash API Docs](https://unsplash.com/developers)
- [Pexels API Docs](https://www.pexels.com/api/)

## 📝 Solution

<details>
<summary>Click to reveal complete media generation workflow</summary>

```javascript
class MediaGenerationPipeline {
  constructor(credentials) {
    this.falApiKey = credentials.falApiKey;
    this.unsplashKey = credentials.unsplashKey;
  }
  
  async generateCampaignMedia(campaign) {
    const mediaRequirements = this.analyzeRequirements(campaign);
    
    const results = await Promise.all([
      this.generateImages(mediaRequirements.images),
      this.generateVideo(mediaRequirements.video),
      this.generateAudio(mediaRequirements.audio)
    ]);
    
    return this.packageResults(results);
  }
  
  async generateImages(requirements) {
    const images = {};
    
    for (const [type, prompt] of Object.entries(requirements)) {
      // Check cache first
      const cached = this.checkCache(prompt);
      if (cached) {
        images[type] = cached;
        continue;
      }
      
      // Generate new image
      try {
        const response = await this.callFalAI('flux/dev', {
          prompt: prompt,
          image_size: this.getImageSize(type)
        });
        
        images[type] = response.url;
        this.cacheResult(prompt, response.url);
      } catch (error) {
        // Fallback to stock photo
        images[type] = await this.getStockPhoto(prompt);
      }
    }
    
    return images;
  }
}
```

</details>

## Next Exercise
[Exercise 7.1: Multi-Agent Orchestration →](../07-advanced-n8n/agent-orchestration.md)
