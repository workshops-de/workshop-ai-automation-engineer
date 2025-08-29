# Exercise 5.1: Building Your First Intelligent Content Workflow

## 🎯 Learning Goals
- Combine AI with N8N workflows
- Implement intelligent decision-making
- Handle complex content scenarios
- Create adaptive workflows

## 📋 Prerequisites
- Completed exercises 1-4
- N8N workflow basics understood
- Claude AI integrated

## 🔨 Task Description

Build an intelligent content workflow that adapts based on input requirements and generates appropriate content types automatically.

### Part 1: Intelligent Content Router (20 min)

#### Build Smart Decision Logic
```javascript
// Intelligent Router Node
const contentRouter = {
  async route(request) {
    const { type, urgency, audience, length } = request;
    
    // Analyze requirements
    const analysis = {
      complexity: this.assessComplexity(request),
      contentType: this.determineType(type, audience),
      workflow: this.selectWorkflow(urgency, length),
      agents: this.assignAgents(request)
    };
    
    // Route to appropriate workflow
    if (analysis.complexity === 'high') {
      return 'multi-agent-workflow';
    } else if (urgency === 'immediate') {
      return 'fast-track-workflow';
    } else {
      return 'standard-workflow';
    }
  },
  
  assessComplexity(request) {
    let score = 0;
    if (request.languages && request.languages.length > 1) score += 2;
    if (request.media && request.media.includes('video')) score += 3;
    if (request.length > 2000) score += 2;
    if (request.technical) score += 1;
    
    return score > 4 ? 'high' : score > 2 ? 'medium' : 'low';
  }
};
```

### Part 2: Adaptive Content Generation (25 min)

#### Create Context-Aware Generation
```javascript
// Adaptive Content Generator
const adaptiveGenerator = {
  async generate(topic, context) {
    // Analyze context for best approach
    const strategy = await this.analyzeContext(context);
    
    // Prepare dynamic prompt
    const prompt = this.buildDynamicPrompt(topic, strategy);
    
    // Generate with appropriate model
    const model = strategy.requiresCreativity ? 'claude-3-opus' : 'claude-3-haiku';
    
    const response = await callClaude({
      model: model,
      system: prompt.system,
      messages: [{
        role: 'user',
        content: prompt.user
      }],
      max_tokens: strategy.expectedLength * 2
    });
    
    return this.processResponse(response, strategy);
  },
  
  buildDynamicPrompt(topic, strategy) {
    const templates = {
      creative: "Create engaging, creative content that captures imagination...",
      informative: "Provide detailed, factual information with sources...",
      persuasive: "Write compelling copy that drives action...",
      educational: "Explain complex topics in simple, clear terms..."
    };
    
    return {
      system: `You are a ${strategy.role}. ${templates[strategy.style]}`,
      user: `Create ${strategy.format} about ${topic}. Requirements: ${strategy.requirements}`
    };
  }
};
```

### Part 3: Quality Assessment Loop (20 min)

#### Implement Automatic Quality Checks
```javascript
// Quality Assessment System
class QualityAssessor {
  constructor() {
    this.criteria = {
      relevance: { weight: 0.3, threshold: 0.7 },
      coherence: { weight: 0.25, threshold: 0.75 },
      engagement: { weight: 0.2, threshold: 0.6 },
      accuracy: { weight: 0.15, threshold: 0.8 },
      brandFit: { weight: 0.1, threshold: 0.7 }
    };
  }
  
  async assess(content, requirements) {
    const scores = {};
    
    // Assess each criterion
    scores.relevance = await this.checkRelevance(content, requirements.topic);
    scores.coherence = this.checkCoherence(content);
    scores.engagement = this.checkEngagement(content);
    scores.accuracy = await this.checkAccuracy(content);
    scores.brandFit = this.checkBrandFit(content, requirements.brand);
    
    // Calculate weighted score
    const totalScore = Object.entries(scores).reduce((total, [criterion, score]) => {
      return total + (score * this.criteria[criterion].weight);
    }, 0);
    
    // Determine if revision needed
    const needsRevision = Object.entries(scores).some(([criterion, score]) => {
      return score < this.criteria[criterion].threshold;
    });
    
    return {
      totalScore,
      scores,
      needsRevision,
      feedback: this.generateFeedback(scores)
    };
  }
  
  generateFeedback(scores) {
    const feedback = [];
    Object.entries(scores).forEach(([criterion, score]) => {
      if (score < this.criteria[criterion].threshold) {
        feedback.push({
          issue: criterion,
          score: score,
          suggestion: this.getSuggestion(criterion)
        });
      }
    });
    return feedback;
  }
}
```

### Part 4: Self-Improving Workflow (25 min)

#### Add Learning Capabilities
```javascript
// Self-Improving System
class LearningWorkflow {
  constructor() {
    this.history = [];
    this.patterns = new Map();
  }
  
  async processWithLearning(request) {
    // Check if we've seen similar requests
    const similar = this.findSimilarRequests(request);
    
    if (similar.length > 0) {
      // Use successful patterns from history
      const bestApproach = this.selectBestApproach(similar);
      const result = await this.executeWithApproach(request, bestApproach);
      
      // Learn from this execution
      this.recordOutcome(request, bestApproach, result);
      
      return result;
    } else {
      // Try new approach and learn
      const result = await this.experimentalProcess(request);
      this.recordOutcome(request, 'experimental', result);
      
      return result;
    }
  }
  
  findSimilarRequests(request) {
    return this.history.filter(historic => {
      const similarity = this.calculateSimilarity(request, historic.request);
      return similarity > 0.7;
    });
  }
  
  calculateSimilarity(req1, req2) {
    // Simple similarity calculation
    let score = 0;
    if (req1.type === req2.type) score += 0.3;
    if (req1.audience === req2.audience) score += 0.2;
    if (Math.abs(req1.length - req2.length) < 200) score += 0.2;
    if (req1.tone === req2.tone) score += 0.3;
    
    return score;
  }
  
  recordOutcome(request, approach, result) {
    this.history.push({
      request,
      approach,
      result,
      quality: result.qualityScore,
      timestamp: Date.now()
    });
    
    // Update patterns
    const key = `${request.type}-${request.audience}`;
    if (!this.patterns.has(key)) {
      this.patterns.set(key, []);
    }
    this.patterns.get(key).push({
      approach,
      success: result.qualityScore > 0.8
    });
  }
}
```

### Part 5: Error Recovery System (10 min)

#### Implement Intelligent Fallbacks
```javascript
// Error Recovery Manager
const errorRecovery = {
  strategies: [
    { name: 'retry', condition: 'timeout' },
    { name: 'simplify', condition: 'complexity' },
    { name: 'alternative', condition: 'api_error' },
    { name: 'manual', condition: 'quality_fail' }
  ],
  
  async handleError(error, context) {
    console.log(`Error occurred: ${error.message}`);
    
    // Identify error type
    const errorType = this.classifyError(error);
    
    // Select recovery strategy
    const strategy = this.strategies.find(s => s.condition === errorType);
    
    if (strategy) {
      return await this.executeStrategy(strategy.name, context);
    } else {
      // Unknown error - escalate
      return this.escalate(error, context);
    }
  },
  
  async executeStrategy(strategyName, context) {
    switch(strategyName) {
      case 'retry':
        await this.wait(5000);
        return await this.retryWithBackoff(context);
        
      case 'simplify':
        context.requirements = this.simplifyRequirements(context.requirements);
        return await this.reprocess(context);
        
      case 'alternative':
        return await this.useAlternativeService(context);
        
      case 'manual':
        return await this.requestHumanIntervention(context);
        
      default:
        throw new Error(`Unknown strategy: ${strategyName}`);
    }
  }
};
```

## 💡 Implementation Tips

1. **Workflow Branching**
   - Use IF nodes for conditional logic
   - Implement parallel branches for efficiency
   - Add merge nodes to combine results

2. **State Management**
   - Use workflow static data for persistence
   - Track execution history
   - Implement checkpoints

3. **Performance**
   - Cache AI responses when possible
   - Batch similar requests
   - Use appropriate models for tasks

## ✅ Success Criteria

- [ ] Intelligent routing based on content type
- [ ] Adaptive prompt generation working
- [ ] Quality assessment implemented
- [ ] Learning system tracking patterns
- [ ] Error recovery handling all cases

## 🚀 Bonus Challenge

Create a "Meta-Optimizer" that:
1. Analyzes workflow performance over time
2. Suggests workflow improvements
3. Automatically adjusts parameters
4. Predicts optimal settings for new requests
5. Generates performance reports

## 📊 Expected Output

```json
{
  "request": {
    "topic": "AI in Healthcare",
    "type": "blog",
    "urgency": "normal",
    "audience": "professionals"
  },
  "routing": {
    "complexity": "medium",
    "selectedWorkflow": "standard-workflow",
    "assignedAgents": ["research", "writing", "quality"]
  },
  "generation": {
    "model": "claude-3-haiku",
    "tokensUsed": 1847,
    "generationTime": 4.2
  },
  "quality": {
    "totalScore": 0.87,
    "scores": {
      "relevance": 0.92,
      "coherence": 0.88,
      "engagement": 0.85,
      "accuracy": 0.90,
      "brandFit": 0.80
    },
    "approved": true
  },
  "learning": {
    "similarRequests": 3,
    "patternMatched": true,
    "improvement": "+12% vs baseline"
  }
}
```

## 🔗 Resources

- [N8N Conditional Logic](https://docs.n8n.io/if)
- [Workflow Variables](https://docs.n8n.io/variables)
- [Error Handling Guide](../../resources/error-handling.md)

## Next Exercise
[Exercise 6.1: Media Generation with fal.ai →](../06-api-integration/fal-ai-media-generation.md)