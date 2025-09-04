# Exercise 15: Code Node with Python & JavaScript

## 🎯 Learning Goals
- Master the Code node in n8n
- Write custom JavaScript and Python code
- Process complex data transformations
- Use built-in methods and variables
- Handle different execution modes

## 📋 Prerequisites
- N8N workflow basics
- Basic programming knowledge (JavaScript or Python)
- Understanding of data structures

## 🔨 Task Description

Learn to use the Code node to implement custom logic, data processing, and complex transformations in your ContentFlow AI workflows.

### Part 1: Code Node Basics (15 min)

#### Understanding Execution Modes

```javascript
// Mode 1: Run Once for All Items
// Processes all input items at once
return items.map(item => {
  return {
    json: {
      ...item.json,
      processed: true,
      timestamp: new Date().toISOString()
    }
  };
});

// Mode 2: Run Once for Each Item  
// Processes each item individually
return {
  json: {
    ...item.json,
    processedIndividually: true,
    index: $itemIndex
  }
};
```

#### Python vs JavaScript Comparison

```python
# Python in Code Node (using Pyodide)
import json
from datetime import datetime

# Access input data
items = _input.all()

# Process data
processed_items = []
for item in items:
    item_data = item['json']
    item_data['processed_python'] = True
    item_data['timestamp'] = datetime.now().isoformat()
    processed_items.append({'json': item_data})

return processed_items
```

```javascript
// JavaScript equivalent
const processedItems = items.map(item => ({
  json: {
    ...item.json,
    processed_js: true,
    timestamp: new Date().toISOString()
  }
}));

return processedItems;
```

### Part 2: Data Processing with JavaScript (25 min)

#### Content Quality Score Calculator

```javascript
// Calculate quality score for content
const calculateContentQuality = (content) => {
  let score = 0;
  const weights = {
    length: 0.2,
    readability: 0.3,
    keywords: 0.2,
    structure: 0.3
  };
  
  // Length score (optimal: 800-1200 words)
  const wordCount = content.split(' ').length;
  if (wordCount >= 800 && wordCount <= 1200) {
    score += weights.length * 100;
  } else if (wordCount >= 500 && wordCount < 800) {
    score += weights.length * 70;
  } else if (wordCount > 1200 && wordCount <= 2000) {
    score += weights.length * 70;
  } else {
    score += weights.length * 40;
  }
  
  // Readability score (sentence length)
  const sentences = content.split(/[.!?]+/);
  const avgWordsPerSentence = wordCount / sentences.length;
  if (avgWordsPerSentence <= 20) {
    score += weights.readability * 100;
  } else if (avgWordsPerSentence <= 25) {
    score += weights.readability * 70;
  } else {
    score += weights.readability * 40;
  }
  
  // Keyword density
  const keywords = ['AI', 'automation', 'workflow', 'content'];
  const keywordCount = keywords.reduce((count, keyword) => {
    const regex = new RegExp(keyword, 'gi');
    const matches = content.match(regex);
    return count + (matches ? matches.length : 0);
  }, 0);
  
  const keywordDensity = (keywordCount / wordCount) * 100;
  if (keywordDensity >= 1 && keywordDensity <= 3) {
    score += weights.keywords * 100;
  } else if (keywordDensity > 3 && keywordDensity <= 5) {
    score += weights.keywords * 70;
  } else {
    score += weights.keywords * 40;
  }
  
  // Structure score (headings, paragraphs)
  const hasHeadings = /^#{1,6}\s/m.test(content);
  const paragraphs = content.split('\n\n').length;
  
  if (hasHeadings && paragraphs >= 5) {
    score += weights.structure * 100;
  } else if (hasHeadings || paragraphs >= 5) {
    score += weights.structure * 70;
  } else {
    score += weights.structure * 40;
  }
  
  return Math.round(score);
};

// Process all content items
const results = [];

for (const item of $input.all()) {
  const content = item.json.content;
  const qualityScore = calculateContentQuality(content);
  
  results.push({
    json: {
      ...item.json,
      qualityScore: qualityScore,
      qualityGrade: qualityScore >= 80 ? 'Excellent' :
                    qualityScore >= 60 ? 'Good' :
                    qualityScore >= 40 ? 'Fair' : 'Poor',
      metrics: {
        wordCount: content.split(' ').length,
        sentences: content.split(/[.!?]+/).length,
        paragraphs: content.split('\n\n').length
      }
    }
  });
}

return results;
```

#### Data Transformation Pipeline

```javascript
// Transform content for multiple platforms
const transformForPlatforms = () => {
  const items = $input.all();
  const transformed = [];
  
  for (const item of items) {
    const original = item.json;
    
    // LinkedIn transformation
    const linkedin = {
      platform: 'LinkedIn',
      content: original.content.substring(0, 3000),
      hashtags: extractHashtags(original.keywords, 5),
      format: 'article'
    };
    
    // Twitter/X transformation
    const twitter = {
      platform: 'Twitter',
      content: createThread(original.content, 280),
      hashtags: extractHashtags(original.keywords, 2),
      format: 'thread'
    };
    
    // Email transformation
    const email = {
      platform: 'Email',
      subject: original.title,
      preheader: original.content.substring(0, 100) + '...',
      body: formatForEmail(original.content),
      cta: 'Read More'
    };
    
    transformed.push({
      json: {
        originalId: original.id,
        platforms: {
          linkedin,
          twitter,
          email
        },
        transformedAt: new Date().toISOString()
      }
    });
  }
  
  return transformed;
};

// Helper functions
function extractHashtags(keywords, limit) {
  return keywords
    .slice(0, limit)
    .map(k => '#' + k.replace(/\s+/g, ''));
}

function createThread(content, charLimit) {
  const sentences = content.split('. ');
  const thread = [];
  let currentTweet = '';
  
  for (const sentence of sentences) {
    if ((currentTweet + sentence).length <= charLimit - 10) {
      currentTweet += sentence + '. ';
    } else {
      if (currentTweet) thread.push(currentTweet.trim());
      currentTweet = sentence + '. ';
    }
  }
  
  if (currentTweet) thread.push(currentTweet.trim());
  
  return thread.map((tweet, i) => 
    `${i + 1}/${thread.length} ${tweet}`
  );
}

function formatForEmail(content) {
  return content
    .replace(/^#+ (.+)$/gm, '<h2>$1</h2>')
    .replace(/\n\n/g, '</p><p>')
    .replace(/^/, '<p>')
    .replace(/$/, '</p>');
}

return transformForPlatforms();
```

### Part 3: Data Processing with Python (25 min)

#### Python Content Analysis

```python
# Python Code Node for content analysis
import json
import re
from datetime import datetime
from collections import Counter

# Get input items
items = _input.all()
results = []

def analyze_content(text):
    """Analyze content and return metrics"""
    # Word frequency analysis
    words = re.findall(r'\w+', text.lower())
    word_freq = Counter(words)
    
    # Remove common words
    common_words = {'the', 'a', 'an', 'and', 'or', 'but', 'in', 'on', 'at', 'to', 'for'}
    for word in common_words:
        word_freq.pop(word, None)
    
    # Sentiment indicators (simple version)
    positive_words = {'good', 'great', 'excellent', 'amazing', 'wonderful', 'fantastic', 'love'}
    negative_words = {'bad', 'poor', 'terrible', 'awful', 'hate', 'difficult', 'problem'}
    
    positive_count = sum(1 for word in words if word in positive_words)
    negative_count = sum(1 for word in words if word in negative_words)
    
    sentiment = 'neutral'
    if positive_count > negative_count * 2:
        sentiment = 'positive'
    elif negative_count > positive_count * 2:
        sentiment = 'negative'
    
    return {
        'word_count': len(words),
        'unique_words': len(set(words)),
        'top_words': word_freq.most_common(10),
        'sentiment': sentiment,
        'positive_ratio': positive_count / len(words) if words else 0,
        'negative_ratio': negative_count / len(words) if words else 0,
        'lexical_diversity': len(set(words)) / len(words) if words else 0
    }

# Process each item
for item in items:
    content = item['json'].get('content', '')
    title = item['json'].get('title', '')
    
    # Analyze content
    analysis = analyze_content(content)
    
    # Extract key phrases (simple n-gram approach)
    bigrams = []
    words = re.findall(r'\w+', content.lower())
    for i in range(len(words) - 1):
        bigrams.append(f"{words[i]} {words[i+1]}")
    
    bigram_freq = Counter(bigrams)
    key_phrases = bigram_freq.most_common(5)
    
    # Build result
    result = {
        'json': {
            **item['json'],
            'analysis': {
                **analysis,
                'key_phrases': key_phrases,
                'reading_time': analysis['word_count'] // 200,  # minutes
                'complexity_score': min(100, analysis['lexical_diversity'] * 100),
                'analyzed_at': datetime.now().isoformat()
            }
        }
    }
    
    results.append(result)

return results
```

#### Python Data Aggregation

```python
# Aggregate content metrics across multiple items
import statistics
from datetime import datetime, timedelta

items = _input.all()

# Collect all metrics
all_scores = []
all_word_counts = []
all_sentiments = []
categories = {}

for item in items:
    json_data = item['json']
    
    # Collect metrics
    if 'qualityScore' in json_data:
        all_scores.append(json_data['qualityScore'])
    
    if 'wordCount' in json_data:
        all_word_counts.append(json_data['wordCount'])
    
    if 'sentiment' in json_data:
        all_sentiments.append(json_data['sentiment'])
    
    # Count categories
    category = json_data.get('category', 'uncategorized')
    categories[category] = categories.get(category, 0) + 1

# Calculate aggregate statistics
aggregate_stats = {
    'total_items': len(items),
    'quality_metrics': {
        'average_score': statistics.mean(all_scores) if all_scores else 0,
        'median_score': statistics.median(all_scores) if all_scores else 0,
        'min_score': min(all_scores) if all_scores else 0,
        'max_score': max(all_scores) if all_scores else 0,
        'std_dev': statistics.stdev(all_scores) if len(all_scores) > 1 else 0
    },
    'content_metrics': {
        'total_words': sum(all_word_counts),
        'average_length': statistics.mean(all_word_counts) if all_word_counts else 0,
        'shortest': min(all_word_counts) if all_word_counts else 0,
        'longest': max(all_word_counts) if all_word_counts else 0
    },
    'sentiment_distribution': {
        'positive': all_sentiments.count('positive'),
        'neutral': all_sentiments.count('neutral'),
        'negative': all_sentiments.count('negative')
    },
    'category_distribution': categories,
    'generated_at': datetime.now().isoformat()
}

# Identify items needing attention
items_needing_attention = []
for item in items:
    json_data = item['json']
    if json_data.get('qualityScore', 100) < 60:
        items_needing_attention.append({
            'id': json_data.get('id'),
            'title': json_data.get('title'),
            'score': json_data.get('qualityScore'),
            'issue': 'Low quality score'
        })

# Return aggregated results
return [{
    'json': {
        'aggregate_stats': aggregate_stats,
        'items_needing_attention': items_needing_attention,
        'recommendations': generate_recommendations(aggregate_stats)
    }
}]

def generate_recommendations(stats):
    """Generate recommendations based on statistics"""
    recommendations = []
    
    if stats['quality_metrics']['average_score'] < 70:
        recommendations.append('Consider improving content quality guidelines')
    
    if stats['content_metrics']['average_length'] < 500:
        recommendations.append('Content may be too short for SEO optimization')
    
    if stats['sentiment_distribution']['negative'] > stats['sentiment_distribution']['positive']:
        recommendations.append('Review content tone - high negative sentiment detected')
    
    return recommendations
```

### Part 4: Built-in Methods & Variables (20 min)

#### JavaScript Built-in Methods

```javascript
// Accessing n8n built-in methods and variables

// 1. $input - Access input data
const inputItems = $input.all(); // All items
const firstItem = $input.first(); // First item
const lastItem = $input.last(); // Last item
const specificItem = $input.item(2); // Item at index 2

// 2. $node - Access other nodes
const previousNodeData = $node["HTTP Request"].json;
const webhookData = $node["Webhook"].json;

// 3. $workflow - Workflow information
const workflowId = $workflow.id;
const workflowName = $workflow.name;

// 4. $execution - Execution information
const executionId = $execution.id;
const executionMode = $execution.mode;

// 5. $env - Environment variables (self-hosted only)
// const apiKey = $env.MY_API_KEY;

// 6. Working with binary data
const binaryData = items[0].binary;
if (binaryData && binaryData.file) {
  const fileName = binaryData.file.fileName;
  const fileSize = binaryData.file.fileSize;
  const mimeType = binaryData.file.mimeType;
}

// 7. Date/time helpers
const now = $now; // Current timestamp
const today = $today; // Today's date

// Practical example: Process with context
const processedItems = [];

for (const item of $input.all()) {
  processedItems.push({
    json: {
      ...item.json,
      workflow: {
        id: $workflow.id,
        name: $workflow.name,
        execution: $execution.id
      },
      processed: {
        at: new Date().toISOString(),
        by: 'Code Node',
        itemIndex: $itemIndex
      }
    }
  });
}

return processedItems;
```

#### Python Built-in Variables

```python
# Python built-in variables use underscore prefix

# 1. _input - Access input data
all_items = _input.all()  # All items
first_item = _input.first()  # First item
last_item = _input.last()  # Last item

# 2. _node - Access other nodes (note: different syntax)
# previous_data = _node["NodeName"]["json"]

# 3. _workflow - Workflow information
workflow_id = _workflow.id
workflow_name = _workflow.name

# 4. _execution - Execution information
execution_id = _execution.id
execution_mode = _execution.mode

# 5. Working with dates
from datetime import datetime
now = datetime.now()

# Process with metadata
results = []
for index, item in enumerate(all_items):
    enriched_item = {
        'json': {
            **item['json'],
            'metadata': {
                'workflow_id': workflow_id,
                'execution_id': execution_id,
                'processed_at': now.isoformat(),
                'item_position': index + 1,
                'total_items': len(all_items)
            }
        }
    }
    results.append(enriched_item)

return results
```

### Part 5: Advanced Patterns (15 min)

#### Error Handling and Validation

```javascript
// Robust error handling in Code node
const processWithErrorHandling = () => {
  const results = [];
  const errors = [];
  
  for (const item of $input.all()) {
    try {
      // Validate input
      if (!item.json.content) {
        throw new Error('Content is required');
      }
      
      if (typeof item.json.content !== 'string') {
        throw new Error('Content must be a string');
      }
      
      // Process item
      const processed = processContent(item.json.content);
      
      results.push({
        json: {
          ...item.json,
          processed: processed,
          status: 'success'
        }
      });
      
    } catch (error) {
      // Log error but continue processing
      console.error(`Error processing item: ${error.message}`);
      
      errors.push({
        item: item.json.id || 'unknown',
        error: error.message
      });
      
      // Add failed item with error status
      results.push({
        json: {
          ...item.json,
          status: 'failed',
          error: error.message
        }
      });
    }
  }
  
  // Add summary at the end
  results.push({
    json: {
      type: 'summary',
      totalProcessed: results.length - 1,
      successful: results.filter(r => r.json.status === 'success').length,
      failed: errors.length,
      errors: errors
    }
  });
  
  return results;
};

function processContent(content) {
  // Your processing logic here
  return {
    processed: true,
    length: content.length,
    timestamp: new Date().toISOString()
  };
}

return processWithErrorHandling();
```

#### Async Operations (JavaScript only)

```javascript
// Using promises and async/await in Code node
const processAsync = async () => {
  const items = $input.all();
  const results = [];
  
  // Process items with delay (simulating API calls)
  for (const item of items) {
    const processed = await processWithDelay(item.json);
    results.push({
      json: processed
    });
  }
  
  return results;
};

async function processWithDelay(data) {
  // Simulate async operation
  await new Promise(resolve => setTimeout(resolve, 100));
  
  return {
    ...data,
    processedAsync: true,
    timestamp: new Date().toISOString()
  };
}

// Return a promise
return processAsync();
```

## 💡 Best Practices

1. **Choose the Right Mode**: Use "Run Once for All Items" for batch processing
2. **Memory Management**: Be aware of memory limits with large datasets
3. **Error Handling**: Always include try-catch blocks
4. **Debugging**: Use console.log() for JavaScript debugging
5. **Performance**: Python is slower due to WebAssembly compilation

## ✅ Success Criteria

- [ ] Created data processing pipeline
- [ ] Implemented quality scoring system
- [ ] Transformed content for multiple platforms
- [ ] Used built-in methods effectively
- [ ] Handled errors gracefully

## 🚀 Bonus Challenge

Create a "Smart Data Processor" that:
1. Auto-detects data format (JSON, CSV, XML)
2. Applies appropriate transformations
3. Validates against schemas
4. Generates data quality reports
5. Optimizes for performance

## 📊 Expected Output

```json
{
  "processed_items": 25,
  "execution_mode": "all_items",
  "language": "javascript",
  "quality_scores": {
    "excellent": 5,
    "good": 12,
    "fair": 6,
    "poor": 2
  },
  "transformations": {
    "linkedin": 25,
    "twitter": 25,
    "email": 25
  },
  "performance": {
    "execution_time": "450ms",
    "memory_used": "12MB",
    "items_per_second": 55
  },
  "errors": []
}
```

## 🔗 Resources

- [n8n Code Node Documentation](https://docs.n8n.io/code/code-node/)
- [Built-in Methods Reference](https://docs.n8n.io/code/builtin/)
- [Python in n8n Guide](../../resources/python-guide.md)

## Next Steps
With Code node mastery, you can handle any data transformation challenge in your automation workflows!
