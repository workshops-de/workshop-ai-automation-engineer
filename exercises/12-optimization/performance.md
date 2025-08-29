# Exercise 12.1: Performance Optimization

## 🎯 Learning Goals
- Optimize API usage and costs
- Implement caching strategies
- Improve response times
- Batch processing techniques

## 📋 Prerequisites
- ContentFlow AI system running
- Understanding of performance metrics
- API cost awareness

## 🔨 Task Description

Optimize ContentFlow AI for maximum performance and minimum cost while maintaining quality.

### Part 1: API Cost Optimization (20 min)

#### Implement Cost-Aware Processing

```javascript
// Cost Optimization Manager
class CostOptimizer {
  constructor() {
    this.costs = {
      claude: {
        haiku: 0.25 / 1000000,    // per token
        sonnet: 3.00 / 1000000,    // per token  
        opus: 15.00 / 1000000      // per token
      },
      falai: {
        flux_schnell: 0.003,       // per image
        flux_dev: 0.025,           // per image
        video_5s: 0.50,            // per video
        audio_minute: 0.10         // per minute
      },
      unsplash: {
        free_tier: 0,              // 50/hour
        paid: 0.001                // per request after limit
      }
    };
    
    this.budget = {
      daily: 50,
      monthly: 1000
    };
    
    this.usage = new UsageTracker();
  }
  
  selectOptimalModel(task) {
    const requirements = this.analyzeTask(task);
    
    // For simple tasks, use cheaper models
    if (requirements.complexity === 'low') {
      return {
        llm: 'claude-haiku',
        image: 'flux-schnell',
        rationale: 'Simple task - using cost-effective models'
      };
    }
    
    // For quality-critical tasks, use better models
    if (requirements.quality === 'critical') {
      return {
        llm: 'claude-sonnet',
        image: 'flux-dev',
        rationale: 'Quality critical - using premium models'
      };
    }
    
    // Default balanced approach
    return {
      llm: 'claude-haiku',
      image: 'flux-schnell',
      rationale: 'Balanced cost/quality approach'
    };
  }
  
  async optimizePrompt(prompt) {
    // Compress prompt to reduce tokens
    const optimized = {
      original: prompt,
      compressed: this.compressPrompt(prompt),
      savings: 0
    };
    
    // Calculate token savings
    const originalTokens = this.countTokens(prompt);
    const compressedTokens = this.countTokens(optimized.compressed);
    optimized.savings = ((originalTokens - compressedTokens) / originalTokens) * 100;
    
    return optimized;
  }
  
  compressPrompt(prompt) {
    // Remove unnecessary whitespace
    let compressed = prompt.replace(/\s+/g, ' ').trim();
    
    // Use abbreviations for common terms
    const abbreviations = {
      'generate': 'gen',
      'create': 'make',
      'professional': 'pro',
      'approximately': '~',
      'information': 'info'
    };
    
    Object.entries(abbreviations).forEach(([full, abbr]) => {
      compressed = compressed.replace(new RegExp(full, 'gi'), abbr);
    });
    
    // Remove redundant instructions
    compressed = compressed.replace(/Please |Kindly |Could you /gi, '');
    
    return compressed;
  }
  
  batchRequests(requests) {
    // Group similar requests for batch processing
    const batches = {};
    
    requests.forEach(req => {
      const key = `${req.type}_${req.model}`;
      if (!batches[key]) {
        batches[key] = [];
      }
      batches[key].push(req);
    });
    
    // Process batches with rate limiting
    return Object.entries(batches).map(([key, items]) => ({
      type: key.split('_')[0],
      model: key.split('_')[1],
      items: items,
      estimatedCost: this.calculateBatchCost(items)
    }));
  }
}
```

### Part 2: Caching Strategy (25 min)

#### Multi-Layer Caching System

```javascript
// Advanced Caching System
class CacheManager {
  constructor() {
    this.layers = {
      memory: new MemoryCache(100),      // In-memory cache
      redis: new RedisCache(),           // Redis cache
      cdn: new CDNCache()                // CDN for media
    };
    
    this.strategies = {
      content: 'memory-first',
      media: 'cdn-first',
      api: 'redis-first'
    };
  }
  
  async get(key, type = 'content') {
    const strategy = this.strategies[type];
    
    // Try cache layers based on strategy
    if (strategy === 'memory-first') {
      // Check memory cache
      let value = await this.layers.memory.get(key);
      if (value) return { value, source: 'memory' };
      
      // Check Redis
      value = await this.layers.redis.get(key);
      if (value) {
        // Populate memory cache
        await this.layers.memory.set(key, value);
        return { value, source: 'redis' };
      }
    }
    
    if (strategy === 'cdn-first') {
      // Check CDN for media
      const cdnUrl = await this.layers.cdn.get(key);
      if (cdnUrl) return { value: cdnUrl, source: 'cdn' };
    }
    
    return null;
  }
  
  async set(key, value, options = {}) {
    const ttl = options.ttl || 3600; // Default 1 hour
    const type = options.type || 'content';
    
    // Store in appropriate layers
    if (type === 'content') {
      await Promise.all([
        this.layers.memory.set(key, value, ttl),
        this.layers.redis.set(key, value, ttl * 2)
      ]);
    } else if (type === 'media') {
      await this.layers.cdn.upload(key, value);
      await this.layers.redis.set(key, value.url, ttl * 24); // 24 hours
    }
  }
  
  // Intelligent cache warming
  async warmCache(predictions) {
    console.log('Warming cache for predicted content...');
    
    for (const prediction of predictions) {
      if (prediction.probability > 0.7) {
        // Pre-generate likely content
        const content = await this.preGenerate(prediction);
        await this.set(prediction.key, content, {
          ttl: 7200, // 2 hours
          type: prediction.type
        });
      }
    }
  }
  
  // Cache invalidation strategy
  async invalidate(pattern) {
    const keys = await this.findKeys(pattern);
    
    await Promise.all([
      this.layers.memory.deleteMany(keys),
      this.layers.redis.deleteMany(keys),
      this.layers.cdn.purge(keys)
    ]);
    
    return keys.length;
  }
}

// Smart Content Deduplication
class ContentDeduplication {
  constructor() {
    this.hashes = new Map();
    this.threshold = 0.85; // Similarity threshold
  }
  
  async findSimilar(content) {
    const contentHash = this.generateHash(content);
    const similar = [];
    
    for (const [hash, cached] of this.hashes) {
      const similarity = this.calculateSimilarity(contentHash, hash);
      if (similarity > this.threshold) {
        similar.push({
          content: cached,
          similarity: similarity
        });
      }
    }
    
    // Return most similar if exists
    if (similar.length > 0) {
      return similar.sort((a, b) => b.similarity - a.similarity)[0];
    }
    
    return null;
  }
  
  generateHash(content) {
    // Generate semantic hash for content
    const words = content.toLowerCase().split(/\s+/);
    const frequencies = {};
    
    words.forEach(word => {
      frequencies[word] = (frequencies[word] || 0) + 1;
    });
    
    return frequencies;
  }
  
  calculateSimilarity(hash1, hash2) {
    // Cosine similarity between content hashes
    const keys = new Set([...Object.keys(hash1), ...Object.keys(hash2)]);
    let dotProduct = 0;
    let norm1 = 0;
    let norm2 = 0;
    
    keys.forEach(key => {
      const val1 = hash1[key] || 0;
      const val2 = hash2[key] || 0;
      dotProduct += val1 * val2;
      norm1 += val1 * val1;
      norm2 += val2 * val2;
    });
    
    return dotProduct / (Math.sqrt(norm1) * Math.sqrt(norm2));
  }
}
```

### Part 3: Response Time Optimization (20 min)

#### Implement Performance Improvements

```javascript
// Performance Optimizer
class PerformanceOptimizer {
  constructor() {
    this.metrics = new PerformanceMetrics();
    this.optimizations = [];
  }
  
  // Lazy Loading Implementation
  async lazyLoadContent(request) {
    // Return critical content immediately
    const critical = await this.getCriticalContent(request);
    
    // Queue non-critical content
    this.queueNonCritical(request);
    
    return {
      immediate: critical,
      deferred: 'Loading additional content...',
      status: 'partial'
    };
  }
  
  // Request Batching
  batchProcessor = {
    queue: [],
    timeout: null,
    maxBatchSize: 10,
    maxWaitTime: 100, // ms
    
    add(request) {
      this.queue.push(request);
      
      if (this.queue.length >= this.maxBatchSize) {
        this.flush();
      } else if (!this.timeout) {
        this.timeout = setTimeout(() => this.flush(), this.maxWaitTime);
      }
    },
    
    async flush() {
      if (this.queue.length === 0) return;
      
      const batch = this.queue.splice(0, this.maxBatchSize);
      clearTimeout(this.timeout);
      this.timeout = null;
      
      // Process batch in parallel
      const results = await Promise.all(
        batch.map(req => this.processSingle(req))
      );
      
      // Return results to callers
      batch.forEach((req, i) => {
        req.resolve(results[i]);
      });
    }
  };
  
  // Connection Pooling
  connectionPool = {
    pools: new Map(),
    
    getConnection(service) {
      if (!this.pools.has(service)) {
        this.pools.set(service, {
          connections: [],
          maxSize: 10,
          inUse: 0
        });
      }
      
      const pool = this.pools.get(service);
      
      // Reuse existing connection
      if (pool.connections.length > 0) {
        const conn = pool.connections.pop();
        pool.inUse++;
        return conn;
      }
      
      // Create new connection if under limit
      if (pool.inUse < pool.maxSize) {
        const conn = this.createConnection(service);
        pool.inUse++;
        return conn;
      }
      
      // Wait for available connection
      return this.waitForConnection(service);
    },
    
    releaseConnection(service, conn) {
      const pool = this.pools.get(service);
      pool.connections.push(conn);
      pool.inUse--;
    }
  };
  
  // Query Optimization
  async optimizeQueries(queries) {
    // Analyze query patterns
    const patterns = this.analyzePatterns(queries);
    
    // Combine similar queries
    const combined = this.combineQueries(patterns);
    
    // Execute optimized queries
    const results = await Promise.all(
      combined.map(q => this.executeOptimized(q))
    );
    
    // Map results back
    return this.mapResults(results, queries);
  }
  
  // Compression
  compress(data) {
    // Use appropriate compression based on content type
    if (typeof data === 'string' && data.length > 1000) {
      return {
        compressed: true,
        algorithm: 'gzip',
        data: this.gzipCompress(data),
        originalSize: data.length,
        compressedSize: this.gzipCompress(data).length
      };
    }
    
    return {
      compressed: false,
      data: data
    };
  }
}
```

### Part 4: Batch Processing (20 min)

#### Efficient Batch Operations

```javascript
// Batch Processing System
class BatchProcessor {
  constructor() {
    this.queues = {
      high: [],
      medium: [],
      low: []
    };
    
    this.processing = false;
    this.batchSize = 25;
  }
  
  async addToBatch(task, priority = 'medium') {
    this.queues[priority].push(task);
    
    if (!this.processing) {
      this.startProcessing();
    }
  }
  
  async startProcessing() {
    this.processing = true;
    
    while (this.hasItems()) {
      // Process high priority first
      if (this.queues.high.length > 0) {
        await this.processBatch('high');
      } else if (this.queues.medium.length > 0) {
        await this.processBatch('medium');
      } else if (this.queues.low.length > 0) {
        await this.processBatch('low');
      }
      
      // Small delay between batches
      await this.delay(100);
    }
    
    this.processing = false;
  }
  
  async processBatch(priority) {
    const batch = this.queues[priority].splice(0, this.batchSize);
    
    // Group by type for efficient processing
    const grouped = this.groupByType(batch);
    
    // Process each group in parallel
    const results = await Promise.all(
      Object.entries(grouped).map(([type, tasks]) => 
        this.processGroup(type, tasks)
      )
    );
    
    // Flatten and return results
    return results.flat();
  }
  
  groupByType(tasks) {
    return tasks.reduce((groups, task) => {
      const type = task.type;
      if (!groups[type]) groups[type] = [];
      groups[type].push(task);
      return groups;
    }, {});
  }
  
  async processGroup(type, tasks) {
    switch(type) {
      case 'text':
        return await this.batchTextGeneration(tasks);
      case 'image':
        return await this.batchImageGeneration(tasks);
      case 'analysis':
        return await this.batchAnalysis(tasks);
      default:
        return await Promise.all(tasks.map(t => this.processSingle(t)));
    }
  }
  
  async batchTextGeneration(tasks) {
    // Combine prompts for single API call
    const combinedPrompt = tasks.map((t, i) => 
      `Task ${i + 1}: ${t.prompt}`
    ).join('\n\n');
    
    const response = await this.callClaudeAPI(combinedPrompt);
    
    // Split response back to individual tasks
    return this.splitResponse(response, tasks.length);
  }
}
```

### Part 5: Monitoring & Analysis (15 min)

#### Performance Monitoring Dashboard

```javascript
// Performance Monitor
class PerformanceMonitor {
  constructor() {
    this.metrics = {
      responseTime: [],
      throughput: [],
      errorRate: [],
      apiCosts: [],
      cacheHitRate: []
    };
    
    this.alerts = [];
  }
  
  track(metric, value) {
    if (!this.metrics[metric]) {
      this.metrics[metric] = [];
    }
    
    this.metrics[metric].push({
      value: value,
      timestamp: Date.now()
    });
    
    // Keep only last hour of data
    this.pruneOldData(metric);
    
    // Check for anomalies
    this.checkAnomaly(metric, value);
  }
  
  getStats(metric, period = 3600000) {
    const data = this.metrics[metric] || [];
    const recent = data.filter(d => 
      Date.now() - d.timestamp < period
    );
    
    if (recent.length === 0) return null;
    
    const values = recent.map(d => d.value);
    
    return {
      min: Math.min(...values),
      max: Math.max(...values),
      avg: values.reduce((a, b) => a + b, 0) / values.length,
      median: this.median(values),
      p95: this.percentile(values, 95),
      p99: this.percentile(values, 99),
      trend: this.calculateTrend(recent)
    };
  }
  
  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      performance: {},
      recommendations: []
    };
    
    // Collect all metrics
    Object.keys(this.metrics).forEach(metric => {
      report.performance[metric] = this.getStats(metric);
    });
    
    // Generate recommendations
    if (report.performance.responseTime?.avg > 2000) {
      report.recommendations.push({
        issue: 'High response time',
        suggestion: 'Consider implementing caching or optimizing queries'
      });
    }
    
    if (report.performance.cacheHitRate?.avg < 0.5) {
      report.recommendations.push({
        issue: 'Low cache hit rate',
        suggestion: 'Review cache key strategy and TTL settings'
      });
    }
    
    if (report.performance.apiCosts?.trend > 0.1) {
      report.recommendations.push({
        issue: 'Rising API costs',
        suggestion: 'Implement request batching and use cheaper models where possible'
      });
    }
    
    return report;
  }
}
```

## 💡 Optimization Strategies

1. **Progressive Enhancement**: Deliver critical content first
2. **Predictive Caching**: Pre-generate likely requests
3. **Smart Batching**: Group similar operations
4. **Model Selection**: Use appropriate models for task complexity
5. **Connection Reuse**: Pool and reuse connections

## ✅ Success Criteria

- [ ] API costs reduced by 40%
- [ ] Response time < 2 seconds
- [ ] Cache hit rate > 60%
- [ ] Batch processing implemented
- [ ] Performance dashboard active

## 🚀 Bonus Challenge

Implement:
1. **ML-based Prediction**: Predict next user requests
2. **Auto-tuning**: Self-adjusting performance parameters
3. **Cost Alerts**: Real-time budget monitoring
4. **A/B Testing**: Compare optimization strategies
5. **Load Testing**: Automated performance testing

## 📊 Expected Output

```json
{
  "optimization_results": {
    "cost_reduction": "42%",
    "speed_improvement": "3.2x",
    "cache_hit_rate": "67%",
    "api_calls_saved": 1250,
    "money_saved": "$145.30"
  },
  "performance_metrics": {
    "avg_response_time": 1850,
    "p95_response_time": 2200,
    "throughput": "450 req/min",
    "error_rate": 0.002
  },
  "optimizations_applied": [
    "prompt_compression",
    "request_batching",
    "intelligent_caching",
    "model_downgrading",
    "connection_pooling"
  ],
  "recommendations": [
    "Implement CDN for media files",
    "Consider edge caching",
    "Optimize database queries"
  ]
}
```

## 🔗 Resources

- [Performance Best Practices](../../resources/performance.md)
- [Caching Strategies](../../resources/caching.md)
- [Cost Optimization Guide](../../resources/cost-optimization.md)

## Next Exercise
[Exercise 13.1: Final Integration →](../13-complete-system/final-integration.md)