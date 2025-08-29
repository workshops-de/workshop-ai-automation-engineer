# Challenge: Advanced Content Classifier

## 🎯 Challenge Goal
Extend the basic email classifier to handle multiple languages, confidence scoring, and intelligent fallback mechanisms.

## 📋 Requirements
- Basic email classifier completed
- Understanding of prompt engineering
- N8N workflow experience

## 🔨 Challenge Tasks

### Task 1: Multi-Language Support (30 min)

Build a classifier that works in multiple languages:

```javascript
// Language Detection
const detectLanguage = async (text) => {
  const prompt = `
    Detect the language of this text and return ISO code:
    "${text.substring(0, 100)}"
    
    Return only the ISO code (e.g., en, de, fr, es)
  `;
  
  const language = await callClaude(prompt);
  return language.trim().toLowerCase();
};

// Language-Specific Classification
const classifyByLanguage = async (email, language) => {
  const prompts = {
    en: "Classify this email into: Support, Sales, Feedback, or Other",
    de: "Klassifizieren Sie diese E-Mail in: Support, Vertrieb, Feedback oder Sonstiges",
    fr: "Classez cet e-mail dans: Support, Ventes, Commentaires ou Autre",
    es: "Clasifique este correo en: Soporte, Ventas, Comentarios u Otro"
  };
  
  const systemPrompt = prompts[language] || prompts.en;
  
  return await classifyEmail(email, systemPrompt);
};
```

### Task 2: Confidence Scoring (30 min)

Implement confidence scoring for classifications:

```javascript
// Confidence Score Implementation
const classifyWithConfidence = async (email) => {
  const prompt = `
    Classify this email and provide confidence score.
    
    Email: "${email}"
    
    Return JSON:
    {
      "category": "Support|Sales|Feedback|Other",
      "confidence": 0.0-1.0,
      "reasoning": "brief explanation",
      "alternative": "second most likely category"
    }
  `;
  
  const response = await callClaude(prompt);
  const result = JSON.parse(response);
  
  // Determine action based on confidence
  if (result.confidence < 0.6) {
    return {
      ...result,
      action: 'human_review',
      reason: 'Low confidence score'
    };
  }
  
  return result;
};
```

### Task 3: Fallback Mechanisms (30 min)

Create intelligent fallback strategies:

```javascript
// Fallback Strategy System
class FallbackHandler {
  constructor() {
    this.strategies = [
      this.tryPrimaryClassifier,
      this.trySecondaryClassifier,
      this.tryKeywordMatching,
      this.tryHumanEscalation
    ];
  }
  
  async classify(email) {
    let lastError;
    
    for (const strategy of this.strategies) {
      try {
        const result = await strategy.call(this, email);
        if (result.success) {
          return result;
        }
      } catch (error) {
        lastError = error;
        console.log(`Strategy failed: ${strategy.name}`);
      }
    }
    
    return this.defaultClassification(email, lastError);
  }
  
  async tryPrimaryClassifier(email) {
    // Claude AI classification
    const result = await classifyWithClaude(email);
    return {
      success: result.confidence > 0.7,
      ...result
    };
  }
  
  async tryKeywordMatching(email) {
    // Fallback to keyword matching
    const keywords = {
      support: ['help', 'issue', 'problem', 'bug', 'error'],
      sales: ['buy', 'purchase', 'price', 'quote', 'demo'],
      feedback: ['feedback', 'suggestion', 'improve', 'love', 'hate']
    };
    
    const emailLower = email.toLowerCase();
    
    for (const [category, words] of Object.entries(keywords)) {
      const matches = words.filter(word => emailLower.includes(word));
      if (matches.length >= 2) {
        return {
          success: true,
          category: category,
          confidence: 0.5,
          method: 'keyword_matching'
        };
      }
    }
    
    return { success: false };
  }
  
  async tryHumanEscalation(email) {
    // Send to human review queue
    await this.sendToHumanQueue(email);
    
    return {
      success: true,
      category: 'pending_review',
      confidence: 0,
      method: 'human_escalation',
      queue_id: generateQueueId()
    };
  }
}
```

### Task 4: Advanced Features (30 min)

Add these advanced capabilities:

#### 1. Sentiment Analysis
```javascript
const analyzeSentiment = async (email) => {
  const sentiments = await detectSentiment(email);
  
  return {
    overall: sentiments.overall,
    anger: sentiments.anger,
    joy: sentiments.joy,
    urgency: sentiments.urgency
  };
};
```

#### 2. Priority Scoring
```javascript
const calculatePriority = (classification, sentiment) => {
  let score = 5; // Default medium
  
  // Adjust based on category
  if (classification.category === 'Support') score += 2;
  if (classification.category === 'Sales') score += 1;
  
  // Adjust based on sentiment
  if (sentiment.anger > 0.7) score += 3;
  if (sentiment.urgency > 0.8) score += 2;
  
  // Adjust based on confidence
  if (classification.confidence < 0.5) score += 1;
  
  return Math.min(10, Math.max(1, score));
};
```

#### 3. Auto-Response Generation
```javascript
const generateAutoResponse = async (email, classification) => {
  if (classification.confidence < 0.8) {
    return null; // Don't auto-respond if uncertain
  }
  
  const templates = {
    support: "Thank you for reaching out. We've received your support request...",
    sales: "Thank you for your interest in our products...",
    feedback: "We appreciate your feedback..."
  };
  
  const response = await personalizeTemplate(
    templates[classification.category],
    email
  );
  
  return response;
};
```

## 💡 Bonus Challenges

1. **Learning System**: Store classifications and improve over time
2. **Multi-Modal**: Handle emails with attachments
3. **Thread Analysis**: Understand email conversation context
4. **Custom Categories**: Allow dynamic category creation
5. **A/B Testing**: Test different classifiers simultaneously

## ✅ Success Criteria

- [ ] Supports 5+ languages
- [ ] Confidence scores accurate
- [ ] Fallback never fails completely
- [ ] Priority scoring works correctly
- [ ] Auto-responses appropriate

## 📊 Expected Output

```json
{
  "email_id": "msg-123",
  "language": "en",
  "classification": {
    "category": "Support",
    "confidence": 0.92,
    "alternative": "Sales",
    "method": "ai_classifier"
  },
  "sentiment": {
    "overall": "negative",
    "anger": 0.6,
    "urgency": 0.8
  },
  "priority": 8,
  "auto_response": {
    "generated": true,
    "template": "support_high_priority",
    "personalized": true
  },
  "processing": {
    "time_ms": 450,
    "fallbacks_used": 0,
    "tokens_consumed": 325
  }
}
```

## 🏆 Completion Rewards

Successfully completing this challenge demonstrates:
- Advanced prompt engineering skills
- Error handling expertise
- Multi-language capabilities
- Production-ready thinking

**Share your solution and compete for the best implementation!**
