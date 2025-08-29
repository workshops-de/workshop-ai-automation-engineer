# Exercise 2.1: Building Your Content Prompt Library

## 🎯 Learning Goals
- Master prompt engineering techniques for content creation
- Build reusable prompt templates
- Implement output formatting with JSON schemas
- Create brand voice consistency

## 📋 Prerequisites
- Completed Exercise 1.1
- Understanding of basic prompt structure
- Claude AI API access

## 🔨 Task Description

Build a comprehensive prompt library for ContentFlow AI that ensures consistent, high-quality content generation across all client projects.

### Part 1: System Prompts for Different Agents (20 min)

Create specialized system prompts for each content agent:

#### 1. Blog Writing Agent
```python
system_prompt = """
You are a professional blog writer for CreativeHub Agency.
You specialize in creating engaging, SEO-optimized content.
Always structure content with clear headings and maintain
a conversational yet authoritative tone.
Output all content in markdown format.
"""
```

**Your Task:** Enhance this prompt with:
- Specific writing guidelines
- SEO requirements
- Content structure rules
- Brand voice parameters

#### 2. Social Media Agent
Create a system prompt that handles:
- Platform-specific requirements (LinkedIn, Twitter, Instagram)
- Character limits
- Hashtag generation
- Emoji usage guidelines

#### 3. Email Marketing Agent
Design a prompt for:
- Subject line optimization
- Preview text creation
- Body content structure
- Call-to-action formatting

### Part 2: Few-Shot Learning Templates (25 min)

Implement few-shot examples for consistency:

#### Blog Post Template
```
Example Input: "AI in Healthcare"
Example Output:
# Revolutionizing Healthcare: How AI is Transforming Patient Care

In the rapidly evolving landscape of healthcare technology...

## Key Benefits
- Improved diagnostic accuracy
- Reduced wait times
- Personalized treatment plans

## Real-World Applications
[Continue pattern...]
```

Create 3 few-shot examples for:
1. Technology topics
2. Business/Finance topics
3. Lifestyle/Wellness topics

### Part 3: JSON Output Schemas (20 min)

Design structured output formats:

#### Blog Post Schema
```json
{
  "title": "string, max 60 chars",
  "meta_description": "string, max 155 chars",
  "introduction": "string, 150-200 words",
  "main_sections": [
    {
      "heading": "string",
      "content": "string",
      "key_points": ["array of strings"]
    }
  ],
  "conclusion": "string, 100-150 words",
  "cta": "string, call-to-action",
  "tags": ["array of strings"],
  "reading_time": "number in minutes"
}
```

Create similar schemas for:
- Social media posts
- Email campaigns
- Product descriptions

### Part 4: Brand Voice Variations (15 min)

Create prompt modifiers for different brand voices:

| Brand Type | Voice Characteristics | Example Modifier |
|------------|----------------------|------------------|
| Corporate | Professional, formal | "Use industry terminology, passive voice where appropriate" |
| Startup | Casual, innovative | "Be conversational, use 'we' and 'you', include modern references" |
| Luxury | Sophisticated, exclusive | "Emphasize quality, use sensory language, create aspiration" |
| Tech | Technical yet accessible | "Explain complex concepts simply, use analogies" |

### Part 5: Chain-of-Thought Prompting (20 min)

Implement step-by-step reasoning for complex content:

```
Create a blog post about [TOPIC].

Step 1: Identify the target audience and their pain points
Step 2: List 3 key messages to convey
Step 3: Choose an engaging angle or hook
Step 4: Outline the structure with headings
Step 5: Write the content following the outline
Step 6: Add a compelling call-to-action
```

Test with 3 different topics and compare quality vs. direct prompting.

## 💡 Implementation Tips

```python
# Template Management Structure
class PromptTemplate:
    def __init__(self, name, system_prompt, user_template, examples=None):
        self.name = name
        self.system_prompt = system_prompt
        self.user_template = user_template
        self.examples = examples or []
    
    def generate(self, **kwargs):
        # Fill template with variables
        return self.user_template.format(**kwargs)

# Usage
blog_template = PromptTemplate(
    name="blog_post",
    system_prompt="You are a blog writer...",
    user_template="Write a blog about {topic} for {audience}",
    examples=[...]
)
```

## ✅ Success Criteria

- [ ] Created 5+ system prompts for different agents
- [ ] Implemented 3+ few-shot learning examples
- [ ] Designed JSON schemas for structured output
- [ ] Tested 4+ brand voice variations
- [ ] Demonstrated chain-of-thought improvements

## 🚀 Bonus Challenge

Create a "prompt optimizer" that:
1. Takes a basic prompt
2. Automatically enhances it with:
   - Role definition
   - Context setting
   - Output formatting
   - Constraints
3. A/B tests variations
4. Tracks performance metrics

## 📊 Testing Framework

Create a testing matrix:

| Prompt Type | Test Cases | Success Rate | Avg Tokens | Notes |
|-------------|------------|--------------|------------|-------|
| Blog Intro | 10 topics | _/10 | | |
| Social Media | 10 products | _/10 | | |
| Email Subject | 10 campaigns | _/10 | | |

## 🔗 Resources

- [Anthropic Prompt Engineering Guide](https://docs.anthropic.com/claude/docs/prompt-engineering)
- [JSON Schema Documentation](https://json-schema.org/)
- [Few-Shot Learning Examples](../../resources/few-shot-examples.md)

## 📝 Solution

<details>
<summary>Click to reveal optimal prompt library structure</summary>

### Master Template Structure

```python
MASTER_SYSTEM_PROMPT = """
You are {role} at CreativeHub Agency.
Your expertise: {expertise}
Writing style: {style}
Target audience: {audience}

Guidelines:
{guidelines}

Always output in {format} format.
"""

CONTENT_GENERATION_TEMPLATE = """
Task: {task_type}
Topic: {topic}
Requirements:
- Length: {length}
- Keywords: {keywords}
- Tone: {tone}

{additional_context}

Format your response as:
{output_structure}
"""

# Example Usage
def generate_blog_post(topic, audience, keywords):
    system = MASTER_SYSTEM_PROMPT.format(
        role="Senior Content Strategist",
        expertise="SEO-optimized blog writing",
        style="engaging and informative",
        audience=audience,
        guidelines="Use H2 headings, include examples, add statistics",
        format="markdown"
    )
    
    user = CONTENT_GENERATION_TEMPLATE.format(
        task_type="Write a comprehensive blog post",
        topic=topic,
        length="800-1000 words",
        keywords=", ".join(keywords),
        tone="professional yet conversational",
        additional_context="Include real-world examples",
        output_structure="Title, Introduction, 3 Main Sections, Conclusion, CTA"
    )
    
    return system, user
```

</details>

## Next Exercise
[Exercise 3.1: Process Mapping for ContentFlow →](../03-problem-solving/process-mapping.md)
