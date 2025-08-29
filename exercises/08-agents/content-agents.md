# Exercise 8.1: Building Specialized Content Agents

## 🎯 Learning Goals
- Design autonomous AI agents for content tasks
- Implement agent communication protocols
- Build supervisor patterns for quality control
- Create multi-agent orchestration

## 📋 Prerequisites
- Understanding of AI prompting
- Completed workflow exercises
- Basic understanding of agent concepts

## 🔨 Task Description

Build a team of specialized content agents that work together to create comprehensive marketing campaigns for ContentFlow AI.

### Part 1: Agent Architecture Design (20 min)

#### Define Agent Roles and Responsibilities

```javascript
// Agent Team Structure
const contentAgentTeam = {
  agents: {
    researcher: {
      role: "Content Research Specialist",
      capabilities: ["trend_analysis", "competitor_research", "keyword_research"],
      tools: ["web_search", "database_query", "analytics_api"],
      autonomy: "high"
    },
    
    strategist: {
      role: "Content Strategy Director",
      capabilities: ["campaign_planning", "content_calendar", "audience_targeting"],
      tools: ["analytics", "planning_tools", "audience_insights"],
      autonomy: "high"
    },
    
    writer: {
      role: "Creative Content Writer",
      capabilities: ["blog_writing", "copywriting", "storytelling"],
      tools: ["ai_writing", "grammar_check", "seo_tools"],
      autonomy: "medium"
    },
    
    designer: {
      role: "Visual Content Creator",
      capabilities: ["image_generation", "video_creation", "infographic_design"],
      tools: ["fal_ai", "canva_api", "stock_photos"],
      autonomy: "medium"
    },
    
    editor: {
      role: "Quality Assurance Editor",
      capabilities: ["fact_checking", "brand_compliance", "optimization"],
      tools: ["review_tools", "brand_guidelines", "quality_metrics"],
      autonomy: "low"
    },
    
    supervisor: {
      role: "Campaign Supervisor",
      capabilities: ["coordination", "decision_making", "conflict_resolution"],
      tools: ["agent_monitor", "workflow_control", "reporting"],
      autonomy: "full"
    }
  }
};
```

### Part 2: Individual Agent Implementation (25 min)

#### Research Agent
```javascript
class ResearchAgent {
  constructor() {
    this.name = "Research Agent";
    this.systemPrompt = `You are a content research specialist. Your job is to:
    1. Analyze market trends
    2. Research competitor content
    3. Identify content gaps
    4. Provide data-driven insights
    
    Always provide sources and confidence levels for your findings.`;
  }
  
  async execute(task) {
    const research = {
      topic: task.topic,
      findings: [],
      sources: [],
      confidence: 0
    };
    
    // Step 1: Trend Analysis
    const trends = await this.analyzeTrends(task.topic);
    research.findings.push({
      type: 'trends',
      data: trends,
      insight: this.extractInsight(trends)
    });
    
    // Step 2: Competitor Analysis
    const competitors = await this.analyzeCompetitors(task.topic);
    research.findings.push({
      type: 'competitors',
      data: competitors,
      gaps: this.identifyGaps(competitors)
    });
    
    // Step 3: Keyword Research
    const keywords = await this.researchKeywords(task.topic);
    research.findings.push({
      type: 'keywords',
      data: keywords,
      recommendations: this.prioritizeKeywords(keywords)
    });
    
    // Calculate confidence
    research.confidence = this.calculateConfidence(research.findings);
    
    return research;
  }
  
  async analyzeTrends(topic) {
    // Use web search and analytics
    const searchQuery = `${topic} trends 2024 statistics`;
    const results = await this.webSearch(searchQuery);
    
    return {
      trending: this.extractTrends(results),
      volume: this.analyzeSearchVolume(topic),
      sentiment: this.analyzeSentiment(results)
    };
  }
  
  think(context) {
    // Agent's reasoning process
    return {
      observation: "Topic has high search volume but low competition",
      hypothesis: "Good opportunity for detailed content",
      recommendation: "Create comprehensive guide with unique angle",
      confidence: 0.85
    };
  }
}
```

#### Writer Agent with Memory
```javascript
class WriterAgent {
  constructor() {
    this.name = "Writer Agent";
    this.memory = new AgentMemory();
    this.style = {
      tone: "professional yet engaging",
      voice: "active",
      perspective: "second person"
    };
  }
  
  async write(brief, research) {
    // Remember previous work
    const context = this.memory.recall(brief.topic);
    
    // Build writing prompt
    const prompt = this.buildPrompt(brief, research, context);
    
    // Generate content
    const content = await this.generate(prompt);
    
    // Store in memory
    this.memory.store(brief.topic, {
      brief: brief,
      content: content,
      timestamp: Date.now()
    });
    
    return content;
  }
  
  buildPrompt(brief, research, context) {
    return `
    Role: You are writing for ${brief.audience}
    Topic: ${brief.topic}
    
    Research findings:
    ${JSON.stringify(research.findings, null, 2)}
    
    Previous related content:
    ${context ? context.summary : 'None'}
    
    Requirements:
    - Length: ${brief.wordCount} words
    - Include keywords: ${brief.keywords.join(', ')}
    - Call-to-action: ${brief.cta}
    
    Style guide:
    ${JSON.stringify(this.style)}
    
    Write the content now:
    `;
  }
  
  async revise(content, feedback) {
    // Self-revision capability
    const revisionPrompt = `
    Original content: ${content}
    
    Feedback: ${JSON.stringify(feedback)}
    
    Revise the content addressing all feedback points.
    Maintain the original structure but improve based on feedback.
    `;
    
    return await this.generate(revisionPrompt);
  }
}

// Agent Memory System
class AgentMemory {
  constructor(maxItems = 100) {
    this.shortTerm = new Map();
    this.longTerm = new Map();
    this.maxItems = maxItems;
  }
  
  store(key, value) {
    // Store in short-term memory
    this.shortTerm.set(key, {
      data: value,
      accessCount: 0,
      timestamp: Date.now()
    });
    
    // Promote to long-term if frequently accessed
    if (this.shortTerm.size > this.maxItems) {
      this.consolidate();
    }
  }
  
  recall(key) {
    // Check short-term first
    if (this.shortTerm.has(key)) {
      const item = this.shortTerm.get(key);
      item.accessCount++;
      return item.data;
    }
    
    // Check long-term
    if (this.longTerm.has(key)) {
      return this.longTerm.get(key);
    }
    
    // Try fuzzy matching
    return this.fuzzyRecall(key);
  }
  
  consolidate() {
    // Move important memories to long-term
    const sorted = Array.from(this.shortTerm.entries())
      .sort((a, b) => b[1].accessCount - a[1].accessCount);
    
    // Keep top 50% in short-term
    const keep = sorted.slice(0, this.maxItems / 2);
    const archive = sorted.slice(this.maxItems / 2);
    
    // Archive to long-term
    archive.forEach(([key, value]) => {
      this.longTerm.set(key, value.data);
      this.shortTerm.delete(key);
    });
  }
}
```

### Part 3: Agent Communication Protocol (20 min)

#### Implement Inter-Agent Messaging
```javascript
class AgentCommunicationBus {
  constructor() {
    this.agents = new Map();
    this.messages = [];
    this.subscriptions = new Map();
  }
  
  registerAgent(agent) {
    this.agents.set(agent.name, agent);
    this.subscriptions.set(agent.name, []);
  }
  
  async sendMessage(from, to, message) {
    const msg = {
      id: this.generateId(),
      from: from,
      to: to,
      content: message,
      timestamp: Date.now(),
      status: 'pending'
    };
    
    this.messages.push(msg);
    
    // Direct message
    if (to) {
      return await this.deliverMessage(msg);
    }
    
    // Broadcast
    return await this.broadcast(msg);
  }
  
  async deliverMessage(message) {
    const recipient = this.agents.get(message.to);
    
    if (!recipient) {
      message.status = 'failed';
      return { error: 'Recipient not found' };
    }
    
    try {
      const response = await recipient.receiveMessage(message);
      message.status = 'delivered';
      message.response = response;
      
      return response;
    } catch (error) {
      message.status = 'error';
      message.error = error.message;
      throw error;
    }
  }
  
  subscribe(agent, topics) {
    this.subscriptions.set(agent, topics);
  }
  
  async broadcast(message) {
    const responses = [];
    
    for (const [agentName, agent] of this.agents) {
      if (agentName === message.from) continue;
      
      // Check if agent is interested
      const topics = this.subscriptions.get(agentName);
      if (topics && topics.includes(message.content.type)) {
        const response = await agent.receiveMessage(message);
        responses.push({ agent: agentName, response });
      }
    }
    
    return responses;
  }
}

// Agent with communication capability
class CommunicatingAgent {
  constructor(name, bus) {
    this.name = name;
    this.bus = bus;
    this.inbox = [];
    this.pendingRequests = new Map();
  }
  
  async receiveMessage(message) {
    this.inbox.push(message);
    
    // Process based on message type
    switch (message.content.type) {
      case 'request':
        return await this.handleRequest(message.content);
      
      case 'response':
        return this.handleResponse(message.content);
      
      case 'notification':
        return this.handleNotification(message.content);
      
      default:
        return { acknowledged: true };
    }
  }
  
  async requestFrom(agentName, request) {
    const response = await this.bus.sendMessage(
      this.name,
      agentName,
      {
        type: 'request',
        data: request
      }
    );
    
    return response;
  }
  
  async collaborate(task) {
    // Example: Writer requesting research
    if (this.name === 'Writer' && !task.research) {
      const research = await this.requestFrom('Researcher', {
        action: 'research',
        topic: task.topic
      });
      
      task.research = research;
    }
    
    return this.execute(task);
  }
}
```

### Part 4: Supervisor Pattern (20 min)

#### Implement Supervisor Agent
```javascript
class SupervisorAgent {
  constructor(agents) {
    this.name = "Supervisor";
    this.agents = agents;
    this.workQueue = [];
    this.results = new Map();
  }
  
  async orchestrate(campaign) {
    console.log(`📋 Starting campaign: ${campaign.name}`);
    
    // Plan execution
    const plan = this.createExecutionPlan(campaign);
    
    // Execute plan
    for (const phase of plan.phases) {
      console.log(`\n🔄 Phase: ${phase.name}`);
      
      const phaseResults = await this.executePhase(phase);
      this.results.set(phase.name, phaseResults);
      
      // Quality check
      const quality = await this.assessQuality(phaseResults);
      
      if (quality.score < quality.threshold) {
        console.log(`⚠️ Quality below threshold. Initiating revision.`);
        await this.initiateRevision(phase, phaseResults, quality.feedback);
      }
    }
    
    // Final review
    return await this.finalReview();
  }
  
  createExecutionPlan(campaign) {
    return {
      campaign: campaign.name,
      phases: [
        {
          name: "Research",
          agents: ["Researcher"],
          parallel: false,
          timeout: 600000,
          dependencies: []
        },
        {
          name: "Strategy",
          agents: ["Strategist"],
          parallel: false,
          timeout: 300000,
          dependencies: ["Research"]
        },
        {
          name: "Content Creation",
          agents: ["Writer", "Designer"],
          parallel: true,
          timeout: 1800000,
          dependencies: ["Strategy"]
        },
        {
          name: "Review",
          agents: ["Editor"],
          parallel: false,
          timeout: 600000,
          dependencies: ["Content Creation"]
        },
        {
          name: "Publishing",
          agents: ["Publisher"],
          parallel: false,
          timeout: 300000,
          dependencies: ["Review"]
        }
      ]
    };
  }
  
  async executePhase(phase) {
    const results = {};
    
    // Get dependencies
    const dependencies = this.gatherDependencies(phase.dependencies);
    
    if (phase.parallel) {
      // Execute agents in parallel
      const promises = phase.agents.map(agentName => 
        this.executeAgent(agentName, dependencies)
      );
      
      const agentResults = await Promise.allSettled(promises);
      
      phase.agents.forEach((agentName, index) => {
        results[agentName] = agentResults[index];
      });
    } else {
      // Execute agents sequentially
      for (const agentName of phase.agents) {
        results[agentName] = await this.executeAgent(agentName, dependencies);
      }
    }
    
    return results;
  }
  
  async executeAgent(agentName, context) {
    const agent = this.agents[agentName];
    
    if (!agent) {
      throw new Error(`Agent ${agentName} not found`);
    }
    
    console.log(`  👤 ${agentName} working...`);
    
    try {
      const result = await agent.execute(context);
      console.log(`  ✅ ${agentName} completed`);
      return { success: true, result };
    } catch (error) {
      console.log(`  ❌ ${agentName} failed: ${error.message}`);
      return { success: false, error: error.message };
    }
  }
  
  async assessQuality(results) {
    // Quality assessment logic
    const metrics = {
      completeness: this.checkCompleteness(results),
      accuracy: this.checkAccuracy(results),
      consistency: this.checkConsistency(results),
      brandCompliance: this.checkBrandCompliance(results)
    };
    
    const score = Object.values(metrics).reduce((a, b) => a + b, 0) / 
                  Object.keys(metrics).length;
    
    return {
      score: score,
      threshold: 7.5,
      metrics: metrics,
      feedback: this.generateFeedback(metrics)
    };
  }
  
  async initiateRevision(phase, results, feedback) {
    console.log(`  🔧 Revision requested for ${phase.name}`);
    
    // Send feedback to agents
    for (const agentName of phase.agents) {
      const agent = this.agents[agentName];
      
      if (agent.revise) {
        const revised = await agent.revise(results[agentName], feedback);
        results[agentName] = revised;
      }
    }
    
    // Re-assess quality
    const newQuality = await this.assessQuality(results);
    console.log(`  📊 Revised quality score: ${newQuality.score}`);
    
    return results;
  }
}
```

### Part 5: Multi-Agent Collaboration Example (15 min)

#### Complete Campaign Creation Flow
```javascript
// Initialize agent team
async function initializeContentTeam() {
  const bus = new AgentCommunicationBus();
  
  const agents = {
    Researcher: new ResearchAgent(),
    Strategist: new StrategyAgent(),
    Writer: new WriterAgent(),
    Designer: new DesignerAgent(),
    Editor: new EditorAgent()
  };
  
  // Register agents with communication bus
  Object.values(agents).forEach(agent => {
    bus.registerAgent(agent);
    agent.bus = bus;
  });
  
  // Create supervisor
  const supervisor = new SupervisorAgent(agents);
  
  return { agents, supervisor, bus };
}

// Execute campaign
async function createCampaign(brief) {
  const { agents, supervisor, bus } = await initializeContentTeam();
  
  console.log("🚀 ContentFlow AI - Multi-Agent Campaign Creation");
  console.log("=" .repeat(50));
  
  // Start campaign
  const campaign = {
    name: brief.campaignName,
    client: brief.client,
    objectives: brief.objectives,
    deliverables: brief.deliverables,
    deadline: brief.deadline
  };
  
  // Supervisor orchestrates
  const results = await supervisor.orchestrate(campaign);
  
  // Generate report
  const report = generateCampaignReport(results);
  
  return report;
}

// Example usage
const campaignBrief = {
  campaignName: "Summer Product Launch",
  client: "TechStartup Inc",
  objectives: ["Increase awareness", "Drive sales", "Build community"],
  deliverables: {
    blog: 3,
    social: 15,
    video: 2,
    email: 5
  },
  deadline: "2024-06-01"
};

const campaign = await createCampaign(campaignBrief);
```

## ✅ Success Criteria

- [ ] 5+ specialized agents implemented
- [ ] Agent communication working
- [ ] Supervisor orchestration functional
- [ ] Memory system operational
- [ ] Quality control integrated
- [ ] Multi-agent collaboration demonstrated

## 🚀 Bonus Challenge

Create an "Adaptive Agent System" that:
1. Learns from successful campaigns
2. Adjusts agent behavior based on performance
3. Predicts optimal agent combinations
4. Self-organizes for efficiency
5. Handles agent failures gracefully

## 📊 Expected Output

```json
{
  "campaign": "Summer Product Launch",
  "agents": {
    "Researcher": { "tasks": 5, "success": 5, "time": "8m" },
    "Strategist": { "tasks": 3, "success": 3, "time": "5m" },
    "Writer": { "tasks": 8, "success": 7, "time": "25m" },
    "Designer": { "tasks": 12, "success": 12, "time": "30m" },
    "Editor": { "tasks": 8, "success": 8, "time": "10m" }
  },
  "collaboration": {
    "messages": 47,
    "revisions": 2,
    "conflicts": 0
  },
  "quality": {
    "initialScore": 6.8,
    "finalScore": 9.2,
    "revisionCycles": 2
  },
  "deliverables": {
    "blog": { "completed": 3, "quality": 9.1 },
    "social": { "completed": 15, "quality": 8.9 },
    "video": { "completed": 2, "quality": 9.5 },
    "email": { "completed": 5, "quality": 8.7 }
  },
  "totalTime": "78 minutes"
}
```

## Next Exercise
[Exercise 9.1: Complete Content Generation System →](../09-content-generation/complete-system.md)
