# Challenge: Multi-Agent Negotiation System

## 🎯 Challenge Goal
Build a system where multiple AI agents negotiate and reach consensus on content strategy decisions.

## 📋 Requirements
- Multi-agent system understanding
- API integration experience
- Advanced N8N workflows

## 🔨 Challenge Tasks

### Task 1: Agent Negotiation Protocol (45 min)

Create agents that can negotiate with each other:

```javascript
// Negotiation Protocol
class NegotiationProtocol {
  constructor() {
    this.agents = {
      creative: new CreativeAgent(),
      analytical: new AnalyticalAgent(), 
      budget: new BudgetAgent(),
      quality: new QualityAgent()
    };
    
    this.negotiationRounds = 5;
  }
  
  async negotiate(proposal) {
    let currentProposal = proposal;
    let consensus = false;
    let round = 0;
    
    const history = [];
    
    while (!consensus && round < this.negotiationRounds) {
      round++;
      console.log(`Negotiation Round ${round}`);
      
      // Each agent evaluates and modifies proposal
      const evaluations = await this.gatherEvaluations(currentProposal);
      history.push(evaluations);
      
      // Check for consensus
      consensus = this.checkConsensus(evaluations);
      
      if (!consensus) {
        // Agents propose modifications
        currentProposal = await this.synthesizeProposal(evaluations);
      }
    }
    
    return {
      finalProposal: currentProposal,
      consensus: consensus,
      rounds: round,
      history: history
    };
  }
  
  async gatherEvaluations(proposal) {
    const evaluations = {};
    
    for (const [name, agent] of Object.entries(this.agents)) {
      evaluations[name] = await agent.evaluate(proposal);
    }
    
    return evaluations;
  }
  
  checkConsensus(evaluations) {
    const scores = Object.values(evaluations).map(e => e.approval);
    return scores.every(score => score >= 0.7);
  }
  
  async synthesizeProposal(evaluations) {
    // Find middle ground between agent positions
    const modifications = [];
    
    Object.entries(evaluations).forEach(([agent, eval]) => {
      if (eval.approval < 0.7) {
        modifications.push(...eval.suggestions);
      }
    });
    
    // Use mediator agent to synthesize
    return await this.mediatorAgent.synthesize(modifications);
  }
}
```

### Task 2: Specialized Negotiating Agents (45 min)

Implement agents with different priorities:

```javascript
// Creative Agent - Prioritizes innovation
class CreativeAgent {
  async evaluate(proposal) {
    const prompt = `
      As a creative director, evaluate this content proposal:
      ${JSON.stringify(proposal)}
      
      Consider:
      - Innovation and originality
      - Audience engagement potential
      - Creative storytelling opportunities
      
      Return JSON:
      {
        "approval": 0.0-1.0,
        "concerns": [],
        "suggestions": [],
        "strengths": []
      }
    `;
    
    const response = await callClaude(prompt);
    return JSON.parse(response);
  }
  
  negotiationStyle() {
    return {
      flexibility: 0.8,
      priorities: ['innovation', 'engagement', 'storytelling'],
      dealBreakers: ['boring', 'generic', 'uninspired']
    };
  }
}

// Budget Agent - Prioritizes cost efficiency
class BudgetAgent {
  constructor(budget) {
    this.totalBudget = budget;
    this.allocations = {};
  }
  
  async evaluate(proposal) {
    const cost = this.calculateCost(proposal);
    const approval = cost <= this.totalBudget ? 
      (this.totalBudget - cost) / this.totalBudget : 0;
    
    return {
      approval: approval,
      concerns: cost > this.totalBudget ? 
        [`Over budget by $${cost - this.totalBudget}`] : [],
      suggestions: this.getCostReductions(proposal),
      metrics: {
        estimatedCost: cost,
        budgetRemaining: this.totalBudget - cost
      }
    };
  }
  
  calculateCost(proposal) {
    let cost = 0;
    
    // Content creation costs
    cost += proposal.blogPosts * 2.5;  // API costs per post
    cost += proposal.videos * 15;      // Video generation
    cost += proposal.images * 0.5;     // Image generation
    
    // Platform costs
    cost += proposal.platforms.length * 5;
    
    return cost;
  }
  
  getCostReductions(proposal) {
    const suggestions = [];
    
    if (proposal.videos > 2) {
      suggestions.push({
        action: 'reduce_videos',
        savings: (proposal.videos - 2) * 15
      });
    }
    
    if (proposal.aiModel === 'claude-opus') {
      suggestions.push({
        action: 'use_cheaper_model',
        savings: proposal.blogPosts * 1.5
      });
    }
    
    return suggestions;
  }
}

// Quality Agent - Ensures standards
class QualityAgent {
  async evaluate(proposal) {
    const qualityChecks = {
      hasQualityControl: proposal.qc !== undefined,
      hasReviewProcess: proposal.review !== undefined,
      hasBrandGuidelines: proposal.brandVoice !== undefined,
      hasSuccessMetrics: proposal.kpis !== undefined
    };
    
    const score = Object.values(qualityChecks).filter(v => v).length / 
                  Object.keys(qualityChecks).length;
    
    return {
      approval: score,
      concerns: Object.entries(qualityChecks)
        .filter(([k, v]) => !v)
        .map(([k]) => `Missing: ${k}`),
      suggestions: this.getQualityImprovements(proposal),
      qualityScore: score
    };
  }
}
```

### Task 3: Consensus Mechanisms (30 min)

Implement different consensus strategies:

```javascript
// Consensus Strategies
class ConsensusStrategies {
  // Weighted voting based on expertise
  weightedConsensus(evaluations, weights) {
    let totalScore = 0;
    let totalWeight = 0;
    
    Object.entries(evaluations).forEach(([agent, eval]) => {
      const weight = weights[agent] || 1;
      totalScore += eval.approval * weight;
      totalWeight += weight;
    });
    
    return totalScore / totalWeight;
  }
  
  // Veto power for critical agents
  vetoConsensus(evaluations, vetoAgents) {
    for (const agent of vetoAgents) {
      if (evaluations[agent].approval < 0.5) {
        return {
          consensus: false,
          vetoedBy: agent,
          reason: evaluations[agent].concerns[0]
        };
      }
    }
    
    return {
      consensus: true,
      averageApproval: this.calculateAverage(evaluations)
    };
  }
  
  // Ranked choice voting
  rankedChoice(proposals, evaluations) {
    const rankings = {};
    
    // Each agent ranks proposals
    Object.entries(evaluations).forEach(([agent, prefs]) => {
      rankings[agent] = prefs.sort((a, b) => 
        b.score - a.score
      ).map(p => p.id);
    });
    
    // Calculate winner using instant runoff
    return this.instantRunoff(rankings);
  }
  
  // Compromise finding
  async findCompromise(evaluations) {
    const conflicts = this.identifyConflicts(evaluations);
    
    for (const conflict of conflicts) {
      const compromise = await this.mediateConflict(
        conflict.agent1,
        conflict.agent2,
        conflict.issue
      );
      
      if (compromise) {
        return compromise;
      }
    }
    
    return null;
  }
}
```

### Task 4: Learning & Adaptation (30 min)

Make agents learn from negotiations:

```javascript
// Learning Negotiation System
class LearningNegotiator {
  constructor() {
    this.negotiationHistory = [];
    this.successPatterns = {};
    this.failurePatterns = {};
  }
  
  async learnFromNegotiation(negotiation) {
    this.negotiationHistory.push(negotiation);
    
    if (negotiation.consensus) {
      this.analyzeSuccess(negotiation);
    } else {
      this.analyzeFailure(negotiation);
    }
    
    // Update agent strategies
    this.updateStrategies();
  }
  
  analyzeSuccess(negotiation) {
    // Extract successful patterns
    const patterns = {
      proposalFeatures: this.extractFeatures(negotiation.finalProposal),
      agentBehaviors: this.extractBehaviors(negotiation.history),
      compromises: this.extractCompromises(negotiation.history)
    };
    
    // Store successful patterns
    Object.entries(patterns).forEach(([key, value]) => {
      if (!this.successPatterns[key]) {
        this.successPatterns[key] = [];
      }
      this.successPatterns[key].push(value);
    });
  }
  
  updateStrategies() {
    // Adjust agent negotiation strategies based on learning
    Object.entries(this.agents).forEach(([name, agent]) => {
      const agentPatterns = this.getAgentPatterns(name);
      
      // Update flexibility based on success
      if (agentPatterns.compromiseSuccess > 0.7) {
        agent.flexibility *= 1.1;
      } else if (agentPatterns.compromiseSuccess < 0.3) {
        agent.flexibility *= 0.9;
      }
      
      // Update priorities based on outcomes
      agent.priorities = this.optimizePriorities(
        agent.priorities,
        agentPatterns.successfulPriorities
      );
    });
  }
  
  predictNegotiationOutcome(proposal, agents) {
    // Use historical data to predict success
    const similarity = this.findSimilarNegotiations(proposal);
    
    if (similarity.length > 0) {
      const successRate = similarity.filter(n => n.consensus).length / 
                         similarity.length;
      
      return {
        predictedSuccess: successRate,
        estimatedRounds: this.averageRounds(similarity),
        likelyConflicts: this.predictConflicts(proposal, agents)
      };
    }
    
    return {
      predictedSuccess: 0.5,
      estimatedRounds: 3,
      likelyConflicts: []
    };
  }
}
```

## 💡 Advanced Features

1. **Deadlock Resolution**: Automatic mediator intervention
2. **Coalition Formation**: Agents forming alliances
3. **Strategic Voting**: Agents voting strategically
4. **Trust Scores**: Agents building trust over time
5. **External Arbitration**: Human intervention when needed

## ✅ Success Criteria

- [ ] Agents reach consensus in 80% of cases
- [ ] Average negotiation rounds < 4
- [ ] Learning improves success rate over time
- [ ] Different consensus mechanisms work
- [ ] Compromises are balanced

## 📊 Expected Output

```json
{
  "negotiation_id": "neg-456",
  "initial_proposal": {...},
  "final_proposal": {
    "content_strategy": "balanced-approach",
    "budget": 4500,
    "timeline": "3-weeks",
    "quality_score": 8.5
  },
  "consensus": {
    "achieved": true,
    "method": "weighted-voting",
    "rounds": 3,
    "final_scores": {
      "creative": 0.82,
      "analytical": 0.75,
      "budget": 0.71,
      "quality": 0.88
    }
  },
  "compromises": [
    "Reduced video count from 5 to 3",
    "Increased review time by 2 days",
    "Switched to hybrid AI model approach"
  ],
  "learning": {
    "patterns_identified": 4,
    "strategy_updates": 2,
    "success_prediction": 0.84
  }
}
```

## 🏆 Challenge Completion

Master this challenge to prove:
- Advanced multi-agent orchestration
- Complex negotiation modeling
- Machine learning integration
- System design skills

**Top implementations will be featured in the workshop!**
