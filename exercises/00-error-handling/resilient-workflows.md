# Exercise 0: Error Handling & Resilient Workflows

## 🎯 Learning Goals
- Implement comprehensive error handling strategies
- Create error workflows with Error Trigger
- Handle different types of failures gracefully
- Build recovery and notification systems
- Debug and investigate failed executions

## 📋 Prerequisites
- Basic understanding of n8n workflows
- Access to n8n instance
- Notification channels (Slack/Email) configured

## 🔨 Task Description

Build a robust error handling system for ContentFlow AI that ensures reliability, provides instant alerts, and implements automatic recovery strategies.

### Part 1: Basic Error Workflow (20 min)

#### Step 1: Create Error Handler Workflow

1. Create a new workflow called "Error Handler"
2. Add Error Trigger as the first node
3. Configure notification flow:

```json
// Error Trigger Output Structure
{
  "execution": {
    "id": "231",
    "url": "https://n8n.example.com/execution/231",
    "retryOf": "34",
    "error": {
      "message": "Example Error Message",
      "stack": "Stacktrace"
    },
    "lastNodeExecuted": "Node With Error",
    "mode": "manual"
  },
  "workflow": {
    "id": "1",
    "name": "Example Workflow"
  }
}
```

#### Step 2: Build Notification System

```yaml
Error Handler Workflow:
  1. Error Trigger
  2. Switch Node (Route by error type)
     - Critical: Immediate Slack + Email
     - Warning: Slack notification
     - Info: Log to database
  3. Format Error Message
  4. Send Notifications
  5. Log to Error Database
```

#### Step 3: Attach to Main Workflows

```yaml
Main Workflow Settings:
  - Go to Workflow Settings
  - Select "Error Workflow"
  - Choose "Error Handler"
  - Save settings
```

### Part 2: Advanced Error Patterns (25 min)

#### Pattern 1: Smart Retry Logic

```javascript
// Code node in Error Workflow
const errorData = $input.first().json;
const retryCount = errorData.execution.retryOf ? 
  parseInt(errorData.execution.retryOf.split('-').pop()) + 1 : 1;

// Determine retry strategy
const retryStrategy = {
  shouldRetry: false,
  delay: 0,
  reason: ''
};

// Check error type and decide on retry
if (errorData.execution.error.message.includes('timeout')) {
  if (retryCount <= 3) {
    retryStrategy.shouldRetry = true;
    retryStrategy.delay = Math.pow(2, retryCount) * 1000; // Exponential backoff
    retryStrategy.reason = 'Timeout error - will retry';
  }
} else if (errorData.execution.error.message.includes('rate limit')) {
  retryStrategy.shouldRetry = true;
  retryStrategy.delay = 60000; // Wait 1 minute
  retryStrategy.reason = 'Rate limit hit - waiting before retry';
} else if (errorData.execution.error.message.includes('connection')) {
  if (retryCount <= 2) {
    retryStrategy.shouldRetry = true;
    retryStrategy.delay = 5000;
    retryStrategy.reason = 'Connection issue - quick retry';
  }
}

return {
  json: {
    ...errorData,
    retryCount,
    retryStrategy
  }
};
```

#### Pattern 2: Contextual Error Handling

```yaml
Content Processing Error Workflow:
  1. Error Trigger
  2. Get Workflow Context (HTTP Request to n8n API)
  3. Identify Failed Stage:
     - Content Generation Failed → Use fallback AI
     - Image Generation Failed → Use stock images
     - Publishing Failed → Queue for manual review
  4. Execute Recovery Action
  5. Update Status Dashboard
```

#### Pattern 3: Error Categorization

```javascript
// Categorize errors for better handling
function categorizeError(error) {
  const categories = {
    'API_ERROR': ['timeout', 'connection', 'ENOTFOUND'],
    'AUTH_ERROR': ['401', 'unauthorized', 'token'],
    'RATE_LIMIT': ['429', 'rate limit', 'quota'],
    'DATA_ERROR': ['validation', 'missing field', 'type error'],
    'SYSTEM_ERROR': ['memory', 'disk space', 'EACCES'],
    'BUSINESS_ERROR': ['duplicate', 'not found', 'invalid']
  };
  
  const errorMessage = error.message.toLowerCase();
  
  for (const [category, keywords] of Object.entries(categories)) {
    if (keywords.some(keyword => errorMessage.includes(keyword))) {
      return {
        category,
        severity: getSeverity(category),
        action: getAction(category)
      };
    }
  }
  
  return {
    category: 'UNKNOWN',
    severity: 'HIGH',
    action: 'ALERT'
  };
}

function getSeverity(category) {
  const severityMap = {
    'API_ERROR': 'MEDIUM',
    'AUTH_ERROR': 'HIGH',
    'RATE_LIMIT': 'LOW',
    'DATA_ERROR': 'MEDIUM',
    'SYSTEM_ERROR': 'CRITICAL',
    'BUSINESS_ERROR': 'LOW'
  };
  return severityMap[category] || 'MEDIUM';
}

function getAction(category) {
  const actionMap = {
    'API_ERROR': 'RETRY',
    'AUTH_ERROR': 'ALERT',
    'RATE_LIMIT': 'WAIT',
    'DATA_ERROR': 'FIX',
    'SYSTEM_ERROR': 'ESCALATE',
    'BUSINESS_ERROR': 'LOG'
  };
  return actionMap[category] || 'ALERT';
}
```

### Part 3: Stop And Error Node Usage (15 min)

#### Implementing Business Logic Validation

```yaml
Content Validation Workflow:
  1. Receive Content
  2. Validate Content (Code Node):
     - Check word count
     - Verify required fields
     - Validate format
  3. IF validation fails:
     - Stop And Error Node:
       errorMessage: "Content validation failed"
       errorDetails: {validation_errors}
  4. Continue processing...
```

#### Custom Error Triggers

```javascript
// Code node before Stop And Error
const content = $input.first().json;
const errors = [];

// Business rule validations
if (content.wordCount < 500) {
  errors.push('Content too short (minimum 500 words)');
}

if (!content.title || content.title.length < 10) {
  errors.push('Title missing or too short');
}

if (!content.category || !['tech', 'business', 'lifestyle'].includes(content.category)) {
  errors.push('Invalid or missing category');
}

if (content.keywords && content.keywords.length < 3) {
  errors.push('At least 3 keywords required');
}

// Trigger error if validations fail
if (errors.length > 0) {
  return {
    json: {
      hasErrors: true,
      errorMessage: `Validation failed: ${errors.join(', ')}`,
      errors: errors,
      originalData: content
    }
  };
}

return {
  json: {
    hasErrors: false,
    ...content
  }
};
```

### Part 4: Error Recovery Strategies (20 min)

#### Fallback Chains

```yaml
AI Content Generation with Fallbacks:
  1. Try Primary AI (Claude)
  2. On Error → Try Secondary AI (GPT)
  3. On Error → Try Simple Template
  4. On Error → Queue for Manual Creation
  
Each stage has its own error handler that attempts the next option
```

#### Circuit Breaker Pattern

```javascript
// Implement circuit breaker for external services
class CircuitBreaker {
  constructor(threshold = 5, timeout = 60000) {
    this.failureCount = 0;
    this.threshold = threshold;
    this.timeout = timeout;
    this.state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN
    this.nextAttempt = 0;
  }
  
  async execute(fn) {
    if (this.state === 'OPEN') {
      if (Date.now() < this.nextAttempt) {
        throw new Error('Circuit breaker is OPEN');
      }
      this.state = 'HALF_OPEN';
    }
    
    try {
      const result = await fn();
      if (this.state === 'HALF_OPEN') {
        this.state = 'CLOSED';
        this.failureCount = 0;
      }
      return result;
    } catch (error) {
      this.failureCount++;
      
      if (this.failureCount >= this.threshold) {
        this.state = 'OPEN';
        this.nextAttempt = Date.now() + this.timeout;
      }
      
      throw error;
    }
  }
}

// Usage in workflow
const breaker = new CircuitBreaker(5, 60000);

try {
  const result = await breaker.execute(async () => {
    // Call external API
    return await makeAPICall();
  });
  return { json: result };
} catch (error) {
  return { 
    json: { 
      error: error.message, 
      fallback: true 
    } 
  };
}
```

### Part 5: Error Monitoring Dashboard (20 min)

#### Create Error Analytics Workflow

```yaml
Error Analytics Workflow:
  1. Schedule Trigger (Every hour)
  2. Query Error Database
  3. Calculate Metrics:
     - Error rate by workflow
     - Most common error types
     - Recovery success rate
     - Average resolution time
  4. Generate Report
  5. Update Dashboard
  6. Alert if thresholds exceeded
```

#### Error Metrics Collection

```javascript
// Aggregate error metrics
const errors = $input.all();
const metrics = {
  totalErrors: errors.length,
  byWorkflow: {},
  byType: {},
  bySeverity: {},
  timeline: [],
  recoveryRate: 0,
  mttr: 0 // Mean Time To Recovery
};

// Process each error
errors.forEach(error => {
  const workflow = error.json.workflow.name;
  const type = error.json.category;
  const severity = error.json.severity;
  
  metrics.byWorkflow[workflow] = (metrics.byWorkflow[workflow] || 0) + 1;
  metrics.byType[type] = (metrics.byType[type] || 0) + 1;
  metrics.bySeverity[severity] = (metrics.bySeverity[severity] || 0) + 1;
});

// Calculate recovery rate
const recovered = errors.filter(e => e.json.recovered).length;
metrics.recoveryRate = (recovered / errors.length) * 100;

// Sort and prepare top issues
metrics.topIssues = Object.entries(metrics.byWorkflow)
  .sort(([,a], [,b]) => b - a)
  .slice(0, 5)
  .map(([workflow, count]) => ({ workflow, count }));

return { json: metrics };
```

### Part 6: Debugging Failed Executions (15 min)

#### Investigation Workflow

```yaml
Failed Execution Investigator:
  1. Manual Trigger with execution ID
  2. Get Execution Data (n8n API)
  3. Analyze:
     - Last successful node
     - Input/output data
     - Error details
     - Execution timeline
  4. Generate Debug Report
  5. Suggest fixes
```

#### Debug Helper Implementation

```javascript
// Debug helper for failed executions
function analyzeFailedExecution(executionData) {
  const analysis = {
    executionId: executionData.id,
    workflow: executionData.workflowData.name,
    failedNode: executionData.data.lastNodeExecuted,
    error: executionData.data.executionError,
    diagnostics: [],
    suggestions: []
  };
  
  // Analyze error patterns
  const errorMessage = analysis.error?.message || '';
  
  if (errorMessage.includes('undefined')) {
    analysis.diagnostics.push('Possible missing data field');
    analysis.suggestions.push('Check data mapping in previous nodes');
  }
  
  if (errorMessage.includes('timeout')) {
    analysis.diagnostics.push('Operation took too long');
    analysis.suggestions.push('Increase timeout or optimize query');
  }
  
  if (errorMessage.includes('memory')) {
    analysis.diagnostics.push('Memory limit exceeded');
    analysis.suggestions.push('Process data in smaller batches');
  }
  
  // Check data flow
  const nodeData = executionData.data.resultData.runData;
  const failedNodeInput = nodeData[analysis.failedNode]?.[0]?.data?.main?.[0];
  
  if (!failedNodeInput || failedNodeInput.length === 0) {
    analysis.diagnostics.push('No input data for failed node');
    analysis.suggestions.push('Check if previous node returns data');
  }
  
  return analysis;
}
```

## 💡 Best Practices

1. **Always set error workflows** for production workflows
2. **Categorize errors** by severity and type
3. **Implement retry logic** with exponential backoff
4. **Use circuit breakers** for external services
5. **Monitor error trends** to identify systemic issues
6. **Document error codes** and recovery procedures
7. **Test error scenarios** during development

## ✅ Success Criteria

- [ ] Created comprehensive error handler workflow
- [ ] Implemented smart retry logic
- [ ] Built notification system for different severities
- [ ] Set up error monitoring dashboard
- [ ] Tested various failure scenarios
- [ ] Documented recovery procedures

## 🚀 Bonus Challenge

Create an "Self-Healing Workflow System" that:
1. Detects common failure patterns
2. Automatically applies fixes
3. Learns from past errors
4. Suggests workflow improvements
5. Maintains 99.9% uptime

## 📊 Expected Output

```json
{
  "error_handling": {
    "workflows_protected": 15,
    "errors_caught": 47,
    "auto_recovered": 38,
    "alerts_sent": 9,
    "recovery_rate": "80.85%",
    "mttr_minutes": 3.5,
    "prevented_failures": 124
  },
  "top_errors": [
    {"type": "API_TIMEOUT", "count": 12, "recovered": 11},
    {"type": "RATE_LIMIT", "count": 8, "recovered": 8},
    {"type": "DATA_VALIDATION", "count": 5, "recovered": 3}
  ],
  "system_health": "OPERATIONAL",
  "last_critical_error": "2 days ago"
}
```

## 🔗 Resources

- [n8n Error Handling Documentation](https://docs.n8n.io/flow-logic/error-handling/)
- [Error Trigger Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.errortrigger/)
- [Stop And Error Node](https://docs.n8n.io/integrations/builtin/core-nodes/n8n-nodes-base.stopanderror/)

## Next Steps
With robust error handling in place, your ContentFlow AI system can handle any failure gracefully!
