# Exercise 1.1: First AI Interaction

## 🎯 Learning Goals
- Understand how Claude AI responds to different prompts
- Measure response quality and consistency
- Calculate token usage and API costs
- Identify AI capabilities and limitations

## 📋 Prerequisites
- Claude AI API key configured
- Basic understanding of HTTP requests
- N8N installed and running

## 🔨 Task Description

In this exercise, you'll test Claude AI with various content generation tasks to understand its capabilities for the ContentFlow AI system.

### Part 1: Test Different Content Types (20 min)

Create prompts to generate the following content types and evaluate the results:

1. **Blog Post Introduction**
   ```
   Write a 150-word introduction for a blog post about "The Future of Remote Work"
   Target audience: HR professionals
   Tone: Professional yet engaging
   ```

2. **Social Media Post**
   ```
   Create a LinkedIn post about a new product launch
   - Maximum 200 characters
   - Include 3 relevant hashtags
   - Call-to-action at the end
   ```

3. **Product Description**
   ```
   Write a product description for eco-friendly water bottles
   - Highlight 3 key features
   - Include emotional appeal
   - SEO-friendly language
   ```

4. **Email Subject Lines**
   ```
   Generate 5 email subject lines for a summer sale campaign
   - Maximum 50 characters each
   - Create urgency
   - A/B test variations
   ```

### Part 2: Quality Assessment (15 min)

For each generated content, evaluate:

| Criteria | Score (1-5) | Notes |
|----------|-------------|-------|
| Relevance | | Does it match the request? |
| Quality | | Is it well-written? |
| Creativity | | Is it unique/engaging? |
| Usability | | Can it be used as-is? |
| Brand Fit | | Does tone match requirements? |

### Part 3: Token Analysis (10 min)

Track for each request:
- Input tokens used
- Output tokens generated
- Total cost (based on Claude pricing)
- Response time

Calculate:
- Average tokens per content type
- Cost per 1000 pieces of content
- Most cost-effective content type

### Part 4: Edge Cases Testing (15 min)

Test AI limitations with these scenarios:

1. **Multilingual Content**
   - Request: "Write this in Spanish, French, and German"
   - Evaluate translation quality

2. **Technical Accuracy**
   - Request: "Explain quantum computing in simple terms"
   - Check for factual errors

3. **Creative Constraints**
   - Request: "Write a haiku about cloud computing"
   - Test format adherence

4. **Long-form Content**
   - Request: "Write a 2000-word article"
   - Monitor coherence over length

## 💡 Tips

- Save successful prompts for your prompt library
- Note patterns in what works well
- Document failure cases for edge case handling
- Consider prompt variations for A/B testing

## ✅ Success Criteria

- [ ] Generated at least 4 different content types
- [ ] Documented quality scores for each
- [ ] Calculated token usage and costs
- [ ] Identified 3+ AI limitations
- [ ] Created initial prompt library

## 🚀 Bonus Challenge

Create a "prompt template" that consistently generates high-quality blog introductions regardless of topic. Test with 5 different topics and measure consistency.

## 📊 Expected Output

By the end of this exercise, you should have:
1. A spreadsheet with quality assessments
2. Cost calculation for 1000 pieces of content
3. List of AI strengths and limitations
4. Initial prompt library with 5+ templates

## 🔗 Resources

- [Claude API Pricing](https://www.anthropic.com/pricing)
- [Token Counter Tool](https://platform.openai.com/tokenizer)
- [Prompt Engineering Guide](../../resources/prompt-engineering.md)

## 📝 Solution

<details>
<summary>Click to reveal solution approach</summary>

### Optimal Prompt Structure

```
System: You are a professional content writer for CreativeHub Agency.

Task: [Specific content type]
Context: [Background information]
Requirements:
- Length: [Word/character count]
- Tone: [Voice and style]
- Format: [Structure needed]
- Keywords: [If applicable]

Output: [Specific format instructions]
```

### Cost Optimization Tips
1. Use clear, concise prompts to reduce input tokens
2. Set max_tokens appropriately for each content type
3. Cache common responses where possible
4. Batch similar requests

</details>

## Next Exercise
[Exercise 2.1: Building Your Prompt Library →](../02-prompt-engineering/prompt-library.md)
