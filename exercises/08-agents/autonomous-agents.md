# Exercise 8.1: Building Autonomous AI Agents

## 🎯 Learning Goals
- Design autonomous agent architectures
- Implement tool use and function calling
- Create agent memory systems
- Build supervisor patterns for agent control

## 📋 Prerequisites
- Understanding of agent concepts
- Claude AI API experience
- Completed multi-agent basics

## 🔨 Task Description

Build truly autonomous agents that can make decisions, use tools, and collaborate without constant human oversight.

### Part 1: Agent Architecture Design (20 min)

#### Autonomous Agent Components

```javascript
class AutonomousAgent {
  constructor(config) {
    this.name = config.name;
    this.role = config.role;
    this.capabilities = config.capabilities;
    
    // Core components
    this.memory = new AgentMemory();
    this.tools = new ToolRegistry();
    this.decisionEngine = new DecisionEngine();
    this.supervisor = null; // Set by orchestrator
  }
  
  async think(context) {
    // Reasoning process
    const thoughts = await this.decisionEngine.analyze(context);
    this.memory.store('thoughts', thoughts);
    return thoughts;
  }
  
  async act(decision) {
    // Execute action based on decision
    const tool = this.tools.select(decision.action);
    const result = await tool.execute(decision.parameters);
    this.memory.store('actions', { decision, result });
    return result;
  }
  
  async reflect(result) {
    // Learn from outcomes
    const reflection = await this.decisionEngine.evaluate(result);
    this.memory.update('experience', reflection);
    return reflection;
  }
}
```

#### ReAct Pattern Implementation

```javascript
// Reasoning + Acting Pattern
const ReActAgent = {
  systemPrompt: `You are an autonomous agent that follows the ReAct pattern.
    For each task:
    1. Thought: Analyze what needs to be done
    2. Action: Choose and execute an action
    3. Observation: Observe the result
    4. Repeat until task is complete
    
    Available actions:
    - search(query): Search for information
    - generate(type, content): Generate content
    - analyze(data): Analyze data
    - decide(options): Make a decision
    - delegate(task, agent): Delegate to another agent`,
  
  async process(task) {
    const maxSteps = 10;
    const trajectory = [];
    
    for (let step = 0; step < maxSteps; step++) {
      // Think
      const thought = await this.think(task, trajectory);
      trajectory.push({ type: 'thought', content: thought });
      
      if (thought.includes('Task complete')) {
        break;
      }
      
      // Act
      const action = await this.selectAction(thought);
      trajectory.push({ type: 'action', content: action });
      
      // Execute
      const observation = await this.execute(action);
      trajectory.push({ type: 'observation', content: observation });
      
      // Update task context
      task.context = observation;
    }
    
    return this.summarize(trajectory);
  }
};
```

### Part 2: Tool Use & Function Calling (25 min)

#### Implement Tool Registry

```javascript
// Tool Registry System
class ToolRegistry {
  constructor() {
    this.tools = new Map();
    this.registerDefaultTools();
  }
  
  registerDefaultTools() {
    // Content Generation Tools
    this.register('generate_text', {
      description: 'Generate text content',
      parameters: {
        type: 'string',
        prompt: 'string',
        length: 'number'
      },
      execute: async (params) => {
        return await this.callClaude(params.prompt, params.length);
      }
    });
    
    // Research Tools
    this.register('web_search', {
      description: 'Search the web for information',
      parameters: {
        query: 'string',
        num_results: 'number'
      },
      execute: async (params) => {
        return await this.searchWeb(params.query, params.num_results);
      }
    });
    
    // Media Generation Tools
    this.register('generate_image', {
      description: 'Generate an image using AI',
      parameters: {
        prompt: 'string',
        style: 'string',
        size: 'string'
      },
      execute: async (params) => {
        return await this.callFalAI(params);
      }
    });
    
    // Analysis Tools
    this.register('analyze_content', {
      description: 'Analyze content for insights',
      parameters: {
        content: 'string',
        analysis_type: 'string'
      },
      execute: async (params) => {
        return await this.analyzeWithAI(params);
      }
    });
    
    // File System Tools
    this.register('read_file', {
      description: 'Read content from a file',
      parameters: {
        path: 'string'
      },
      execute: async (params) => {
        return await this.readFile(params.path);
      }
    });
    
    this.register('write_file', {
      description: 'Write content to a file',
      parameters: {
        path: 'string',
        content: 'string'
      },
      execute: async (params) => {
        return await this.writeFile(params.path, params.content);
      }
    });
  }
  
  getToolDescriptions() {
    const descriptions = [];
    this.tools.forEach((tool, name) => {
      descriptions.push({
        name: name,
        description: tool.description,
        parameters: tool.parameters
      });
    });
    return descriptions;
  }
  
  async executeTool(toolName, parameters) {
    const tool = this.tools.get(toolName);
    if (!tool) {
      throw new Error(`Tool ${toolName} not found`);
    }
    
    // Validate parameters
    this.validateParameters(tool.parameters, parameters);
    
    // Execute with error handling
    try {
      const result = await tool.execute(parameters);
      return {
        success: true,
        result: result
      };
    } catch (error) {
      return {
        success: false,
        error: error.message
      };
    }
  }
}

// Agent with Tool Use
const ToolUsingAgent = {
  async processWithTools(task) {
    const prompt = `
      Task: ${task.description}
      
      Available tools:
      ${JSON.stringify(this.tools.getToolDescriptions(), null, 2)}
      
      Plan your approach and specify which tools to use.
      Format: 
      {
        "thought": "your reasoning",
        "tool": "tool_name",
        "parameters": {...}
      }
    `;
    
    const response = await this.think(prompt);
    const plan = JSON.parse(response);
    
    // Execute tool
    const result = await this.tools.executeTool(
      plan.tool,
      plan.parameters
    );
    
    // Process result
    return await this.processToolResult(result, task);
  }
};
```

### Part 3: Agent Memory Systems (20 min)

#### Implement Memory Management

```javascript
// Agent Memory System
class AgentMemory {
  constructor(capacity = 100) {
    this.shortTerm = [];
    this.longTerm = new Map();
    this.episodic = [];
    this.capacity = capacity;
  }
  
  // Short-term memory for current context
  addShortTerm(item) {
    this.shortTerm.push({
      content: item,
      timestamp: Date.now()
    });
    
    // Limit size
    if (this.shortTerm.length > 10) {
      this.shortTerm.shift();
    }
  }
  
  // Long-term memory for learned patterns
  addLongTerm(key, value) {
    this.longTerm.set(key, {
      value: value,
      frequency: (this.longTerm.get(key)?.frequency || 0) + 1,
      lastAccessed: Date.now()
    });
    
    // Prune least used if over capacity
    if (this.longTerm.size > this.capacity) {
      this.pruneLeastUsed();
    }
  }
  
  // Episodic memory for complete interactions
  addEpisode(episode) {
    this.episodic.push({
      ...episode,
      timestamp: Date.now(),
      success: episode.success || false
    });
    
    // Keep only recent episodes
    if (this.episodic.length > 50) {
      this.episodic.shift();
    }
  }
  
  // Retrieve relevant memories
  retrieve(query, type = 'all') {
    const results = {
      shortTerm: [],
      longTerm: [],
      episodic: []
    };
    
    if (type === 'all' || type === 'short') {
      results.shortTerm = this.searchShortTerm(query);
    }
    
    if (type === 'all' || type === 'long') {
      results.longTerm = this.searchLongTerm(query);
    }
    
    if (type === 'all' || type === 'episodic') {
      results.episodic = this.searchEpisodic(query);
    }
    
    return results;
  }
  
  // Learn from experiences
  consolidate() {
    // Move important short-term to long-term
    const importantMemories = this.shortTerm.filter(m => 
      m.importance > 0.7
    );
    
    importantMemories.forEach(memory => {
      this.addLongTerm(
        this.generateKey(memory),
        memory.content
      );
    });
    
    // Extract patterns from episodes
    const patterns = this.extractPatterns();
    patterns.forEach(pattern => {
      this.addLongTerm(`pattern_${pattern.type}`, pattern);
    });
  }
  
  extractPatterns() {
    // Analyze episodic memory for patterns
    const successfulEpisodes = this.episodic.filter(e => e.success);
    const patterns = [];
    
    // Group by task type
    const taskGroups = {};
    successfulEpisodes.forEach(episode => {
      const taskType = episode.taskType || 'general';
      if (!taskGroups[taskType]) {
        taskGroups[taskType] = [];
      }
      taskGroups[taskType].push(episode);
    });
    
    // Extract common patterns
    Object.entries(taskGroups).forEach(([type, episodes]) => {
      if (episodes.length >= 3) {
        patterns.push({
          type: type,
          commonActions: this.findCommonActions(episodes),
          averageTime: this.calculateAverageTime(episodes),
          successRate: episodes.length / this.episodic.filter(
            e => e.taskType === type
          ).length
        });
      }
    });
    
    return patterns;
  }
}

// Memory-Enhanced Agent
const MemoryAgent = {
  async processWithMemory(task) {
    // Retrieve relevant memories
    const memories = this.memory.retrieve(task.description);
    
    // Check for similar past tasks
    const similarTasks = memories.episodic.filter(
      e => this.calculateSimilarity(e.task, task) > 0.7
    );
    
    if (similarTasks.length > 0) {
      // Use past successful approach
      const bestApproach = similarTasks.sort(
        (a, b) => b.successScore - a.successScore
      )[0];
      
      console.log('Using learned approach from memory');
      return await this.applyLearnedApproach(bestApproach, task);
    }
    
    // No similar tasks, reason from scratch
    const result = await this.reasonAndAct(task, memories);
    
    // Store episode
    this.memory.addEpisode({
      task: task,
      approach: result.approach,
      outcome: result.outcome,
      success: result.success,
      successScore: result.qualityScore
    });
    
    // Consolidate memories
    this.memory.consolidate();
    
    return result;
  }
};
```

### Part 4: Supervisor Pattern (20 min)

#### Implement Agent Supervision

```javascript
// Supervisor Agent
class SupervisorAgent {
  constructor() {
    this.subordinates = new Map();
    this.policies = this.loadPolicies();
    this.metrics = new MetricsCollector();
  }
  
  registerSubordinate(agent) {
    this.subordinates.set(agent.id, {
      agent: agent,
      performance: {
        tasksCompleted: 0,
        successRate: 1.0,
        averageTime: 0,
        errors: []
      },
      status: 'idle'
    });
  }
  
  async delegateTask(task) {
    // Analyze task requirements
    const requirements = this.analyzeTask(task);
    
    // Select best agent
    const agent = this.selectAgent(requirements);
    
    if (!agent) {
      return this.escalate(task, 'No suitable agent available');
    }
    
    // Monitor execution
    return await this.superviseExecution(agent, task);
  }
  
  selectAgent(requirements) {
    let bestAgent = null;
    let bestScore = 0;
    
    this.subordinates.forEach((info, agentId) => {
      if (info.status === 'idle') {
        const score = this.calculateFitScore(
          info.agent.capabilities,
          requirements
        );
        
        // Factor in performance
        score *= info.performance.successRate;
        
        if (score > bestScore) {
          bestScore = score;
          bestAgent = info.agent;
        }
      }
    });
    
    return bestAgent;
  }
  
  async superviseExecution(agent, task) {
    const agentInfo = this.subordinates.get(agent.id);
    agentInfo.status = 'working';
    
    const startTime = Date.now();
    const checkpoints = [];
    
    try {
      // Set up monitoring
      const monitor = setInterval(() => {
        checkpoints.push(this.checkProgress(agent));
      }, 5000); // Check every 5 seconds
      
      // Execute with timeout
      const result = await Promise.race([
        agent.execute(task),
        this.timeout(30000) // 30 second timeout
      ]);
      
      clearInterval(monitor);
      
      // Validate result
      const validation = this.validateResult(result, task);
      
      if (!validation.valid) {
        throw new Error(`Validation failed: ${validation.reason}`);
      }
      
      // Update metrics
      this.updatePerformance(agent.id, {
        success: true,
        time: Date.now() - startTime
      });
      
      agentInfo.status = 'idle';
      return result;
      
    } catch (error) {
      // Handle failure
      agentInfo.status = 'idle';
      agentInfo.performance.errors.push({
        task: task.id,
        error: error.message,
        timestamp: Date.now()
      });
      
      // Decide on recovery strategy
      return await this.handleFailure(agent, task, error);
    }
  }
  
  async handleFailure(agent, task, error) {
    const strategies = [
      'retry_same_agent',
      'try_different_agent',
      'decompose_task',
      'escalate_to_human'
    ];
    
    const strategy = this.selectRecoveryStrategy(task, error);
    
    switch (strategy) {
      case 'retry_same_agent':
        return await this.superviseExecution(agent, task);
        
      case 'try_different_agent':
        const newAgent = this.selectAlternativeAgent(agent.id);
        return await this.superviseExecution(newAgent, task);
        
      case 'decompose_task':
        const subtasks = this.decomposeTask(task);
        const results = await Promise.all(
          subtasks.map(st => this.delegateTask(st))
        );
        return this.combineResults(results);
        
      case 'escalate_to_human':
      default:
        return this.escalate(task, error);
    }
  }
  
  // Quality control
  validateResult(result, task) {
    const validations = [];
    
    // Check completeness
    validations.push({
      check: 'completeness',
      valid: this.isComplete(result, task.requirements)
    });
    
    // Check quality
    validations.push({
      check: 'quality',
      valid: result.qualityScore >= task.minQuality
    });
    
    // Check compliance
    validations.push({
      check: 'compliance',
      valid: this.checkCompliance(result, this.policies)
    });
    
    const allValid = validations.every(v => v.valid);
    
    return {
      valid: allValid,
      validations: validations,
      reason: validations.find(v => !v.valid)?.check
    };
  }
}
```

### Part 5: Multi-Agent Collaboration (15 min)

#### Implement Agent Communication

```javascript
// Agent Communication Protocol
class AgentCommunication {
  constructor() {
    this.messageQueue = new Map();
    this.subscriptions = new Map();
  }
  
  // Send message between agents
  async sendMessage(fromAgent, toAgent, message) {
    const formattedMessage = {
      id: this.generateMessageId(),
      from: fromAgent.id,
      to: toAgent.id,
      type: message.type,
      content: message.content,
      timestamp: Date.now(),
      requiresResponse: message.requiresResponse || false
    };
    
    // Add to recipient's queue
    if (!this.messageQueue.has(toAgent.id)) {
      this.messageQueue.set(toAgent.id, []);
    }
    this.messageQueue.get(toAgent.id).push(formattedMessage);
    
    // Notify recipient
    if (message.requiresResponse) {
      return await this.waitForResponse(formattedMessage.id);
    }
    
    return { sent: true, messageId: formattedMessage.id };
  }
  
  // Broadcast to multiple agents
  broadcast(fromAgent, message, filter = null) {
    const recipients = filter ? 
      this.filterAgents(filter) : 
      this.getAllAgents();
    
    recipients.forEach(agent => {
      if (agent.id !== fromAgent.id) {
        this.sendMessage(fromAgent, agent, message);
      }
    });
  }
  
  // Subscribe to message types
  subscribe(agent, messageType, handler) {
    if (!this.subscriptions.has(messageType)) {
      this.subscriptions.set(messageType, []);
    }
    
    this.subscriptions.get(messageType).push({
      agent: agent,
      handler: handler
    });
  }
}

// Collaborative Task Execution
const CollaborativeAgent = {
  async executeCollaboratively(task) {
    // Announce task to other agents
    await this.communication.broadcast(this, {
      type: 'task_announcement',
      content: {
        task: task,
        needsHelp: this.assessComplexity(task) > 0.7
      }
    });
    
    // Wait for volunteers
    const volunteers = await this.waitForVolunteers();
    
    if (volunteers.length > 0) {
      // Coordinate with volunteers
      return await this.coordinateExecution(task, volunteers);
    }
    
    // Execute solo
    return await this.executeSolo(task);
  },
  
  async coordinateExecution(task, volunteers) {
    // Divide task
    const subtasks = this.divideTask(task, volunteers.length + 1);
    
    // Assign subtasks
    const assignments = volunteers.map((volunteer, index) => ({
      agent: volunteer,
      subtask: subtasks[index + 1]
    }));
    
    // Execute own subtask
    const myResult = await this.execute(subtasks[0]);
    
    // Collect results from others
    const otherResults = await Promise.all(
      assignments.map(async (assignment) => {
        const response = await this.communication.sendMessage(
          this,
          assignment.agent,
          {
            type: 'subtask_assignment',
            content: assignment.subtask,
            requiresResponse: true
          }
        );
        return response;
      })
    );
    
    // Combine results
    return this.combineResults([myResult, ...otherResults]);
  }
};
```

## 💡 Best Practices

1. **Agent Boundaries**: Clear responsibilities
2. **Failure Handling**: Graceful degradation
3. **Resource Management**: Monitor API usage
4. **Audit Trail**: Log all decisions
5. **Human Oversight**: Critical decision points

## ✅ Success Criteria

- [ ] Autonomous decision-making working
- [ ] Tool use implemented correctly
- [ ] Memory system functional
- [ ] Supervisor controlling agents
- [ ] Agents collaborating successfully

## 🚀 Bonus Challenge

Create a "Self-Improving Agent System" that:
1. Learns from each interaction
2. Optimizes its own prompts
3. Discovers new tool combinations
4. Teaches other agents
5. Evolves strategies over time

## 📊 Expected Output

```json
{
  "task": "Create comprehensive marketing campaign",
  "execution": {
    "lead_agent": "CampaignManager",
    "collaborators": ["Writer", "Designer", "Analyst"],
    "duration": 120.5,
    "decisions_made": 23,
    "tools_used": 12
  },
  "trajectory": [
    {"step": 1, "thought": "Need to research target audience", "action": "search"},
    {"step": 2, "observation": "Found demographic data", "action": "analyze"},
    {"step": 3, "thought": "Create content strategy", "action": "delegate"}
  ],
  "memory_updates": {
    "patterns_learned": 3,
    "long_term_stored": 5
  },
  "result": {
    "campaign_components": 7,
    "quality_score": 9.2,
    "autonomous_decisions": 18
  }
}
```

## 🔗 Resources

- [Agent Architectures](../../resources/agent-architectures.md)
- [Tool Use Patterns](../../resources/tool-patterns.md)
- [Memory Systems](../../resources/memory-systems.md)

## Next Exercise
[Exercise 9.1: Complete Content Generation System →](../09-content-generation/complete-system.md)
