# Exercise 10.1: MCP Server Integration for ContentFlow AI

## 🎯 Learning Goals
- Understand Model Context Protocol (MCP)
- Install and configure official MCP servers
- Extend AI capabilities with tools
- Implement file system and database access

## 📋 Prerequisites
- Node.js v18+ installed
- Understanding of AI tool use
- Completed previous workflow exercises

## 🔨 Task Description

Integrate MCP servers to give your ContentFlow AI agents access to file systems, databases, and web search capabilities.

### Part 1: MCP Server Setup (15 min)

#### Understanding MCP Architecture
```
Claude AI ←→ MCP Client ←→ MCP Server ←→ Resources
                ↓              ↓
            Protocol      Tools/Functions
```

#### Install Official MCP Servers

```bash
# File System Server
npm install -g @modelcontextprotocol/server-filesystem

# PostgreSQL Server
npm install -g @modelcontextprotocol/server-postgres

# Web Search Server (if available)
npm install -g @modelcontextprotocol/server-websearch

# GitHub Server
npm install -g @modelcontextprotocol/server-github
```

#### Configure MCP Settings
```json
// mcp-config.json
{
  "servers": {
    "filesystem": {
      "command": "mcp-server-filesystem",
      "args": ["--root", "./content"],
      "env": {
        "MCP_READ_ONLY": "false"
      }
    },
    "database": {
      "command": "mcp-server-postgres",
      "args": [],
      "env": {
        "POSTGRES_URL": "postgresql://user:pass@localhost/contentflow"
      }
    }
  }
}
```

### Part 2: File System Integration (20 min)

#### Enable File Operations for Content Management

##### Setup File System Server
```javascript
// Start MCP filesystem server
const { spawn } = require('child_process');

const fileServer = spawn('mcp-server-filesystem', [
  '--root', '/workspace/content',
  '--allow-write',
  '--allow-delete'
]);

fileServer.stdout.on('data', (data) => {
  console.log(`MCP FS: ${data}`);
});
```

##### Implement Content Storage Agent
```javascript
// Agent that can read/write files via MCP
const ContentStorageAgent = {
  async saveContent(content, metadata) {
    // Use MCP to save content
    const filename = `${metadata.slug}.md`;
    const path = `/content/blog/${filename}`;
    
    const result = await mcp.tools.writeFile({
      path: path,
      content: content,
      encoding: 'utf8'
    });
    
    // Save metadata
    await mcp.tools.writeFile({
      path: `/content/metadata/${metadata.slug}.json`,
      content: JSON.stringify(metadata, null, 2)
    });
    
    return { path, success: true };
  },
  
  async readContent(slug) {
    const contentPath = `/content/blog/${slug}.md`;
    const metaPath = `/content/metadata/${slug}.json`;
    
    const [content, metadata] = await Promise.all([
      mcp.tools.readFile({ path: contentPath }),
      mcp.tools.readFile({ path: metaPath })
    ]);
    
    return {
      content: content,
      metadata: JSON.parse(metadata)
    };
  },
  
  async listContent(directory = '/content/blog') {
    const files = await mcp.tools.listDirectory({ path: directory });
    return files.filter(f => f.endsWith('.md'));
  }
};
```

### Part 3: Database Integration (20 min)

#### Connect Content Database via MCP

##### Setup PostgreSQL Schema
```sql
-- Content management schema
CREATE TABLE content (
  id SERIAL PRIMARY KEY,
  slug VARCHAR(255) UNIQUE,
  title VARCHAR(500),
  content TEXT,
  metadata JSONB,
  created_at TIMESTAMP DEFAULT NOW(),
  updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE media (
  id SERIAL PRIMARY KEY,
  content_id INTEGER REFERENCES content(id),
  type VARCHAR(50),
  url TEXT,
  provider VARCHAR(50),
  cost DECIMAL(10, 4),
  created_at TIMESTAMP DEFAULT NOW()
);

CREATE TABLE analytics (
  id SERIAL PRIMARY KEY,
  content_id INTEGER REFERENCES content(id),
  views INTEGER DEFAULT 0,
  engagement_rate DECIMAL(5, 2),
  conversion_rate DECIMAL(5, 2),
  date DATE DEFAULT CURRENT_DATE
);
```

##### Database Operations Agent
```javascript
const DatabaseAgent = {
  async storeContent(contentData) {
    const query = `
      INSERT INTO content (slug, title, content, metadata)
      VALUES ($1, $2, $3, $4)
      RETURNING id
    `;
    
    const result = await mcp.tools.queryDatabase({
      query: query,
      params: [
        contentData.slug,
        contentData.title,
        contentData.content,
        JSON.stringify(contentData.metadata)
      ]
    });
    
    return result.rows[0].id;
  },
  
  async getContentAnalytics(slug) {
    const query = `
      SELECT c.title, c.created_at,
             SUM(a.views) as total_views,
             AVG(a.engagement_rate) as avg_engagement,
             COUNT(m.id) as media_count
      FROM content c
      LEFT JOIN analytics a ON c.id = a.content_id
      LEFT JOIN media m ON c.id = m.content_id
      WHERE c.slug = $1
      GROUP BY c.id
    `;
    
    const result = await mcp.tools.queryDatabase({
      query: query,
      params: [slug]
    });
    
    return result.rows[0];
  },
  
  async trackMediaCost(contentId, mediaType, url, provider, cost) {
    const query = `
      INSERT INTO media (content_id, type, url, provider, cost)
      VALUES ($1, $2, $3, $4, $5)
    `;
    
    await mcp.tools.queryDatabase({
      query: query,
      params: [contentId, mediaType, url, provider, cost]
    });
  }
};
```

### Part 4: Web Search Integration (15 min)

#### Add Research Capabilities

##### Configure Web Search
```javascript
// Research Agent with web search via MCP
const ResearchAgent = {
  async research(topic, depth = 'basic') {
    const searches = {
      basic: [topic],
      medium: [topic, `${topic} trends`, `${topic} statistics`],
      deep: [
        topic,
        `${topic} trends 2024`,
        `${topic} statistics`,
        `${topic} best practices`,
        `${topic} case studies`
      ]
    };
    
    const searchQueries = searches[depth];
    const results = [];
    
    for (const query of searchQueries) {
      const searchResult = await mcp.tools.webSearch({
        query: query,
        maxResults: 5
      });
      
      results.push({
        query: query,
        results: searchResult
      });
    }
    
    return this.synthesizeResearch(results);
  },
  
  synthesizeResearch(searchResults) {
    // Combine and summarize research findings
    const insights = {
      mainPoints: [],
      statistics: [],
      sources: [],
      trends: []
    };
    
    searchResults.forEach(search => {
      search.results.forEach(result => {
        insights.sources.push({
          title: result.title,
          url: result.url,
          snippet: result.snippet
        });
        
        // Extract key information
        if (result.snippet.match(/\d+%/)) {
          insights.statistics.push(result.snippet);
        }
      });
    });
    
    return insights;
  }
};
```

### Part 5: Multi-Tool Orchestration (20 min)

#### Combine MCP Tools for Complex Tasks

##### Content Creation Pipeline with MCP
```javascript
class MCPContentPipeline {
  async createContentCampaign(brief) {
    // 1. Research Phase
    const research = await ResearchAgent.research(brief.topic, 'deep');
    
    // 2. Content Generation
    const content = await this.generateContent(brief, research);
    
    // 3. Save to File System
    const filePath = await ContentStorageAgent.saveContent(
      content.body,
      content.metadata
    );
    
    // 4. Store in Database
    const contentId = await DatabaseAgent.storeContent({
      slug: content.metadata.slug,
      title: content.title,
      content: content.body,
      metadata: content.metadata
    });
    
    // 5. Generate Media
    const media = await this.generateMedia(content);
    
    // 6. Track Costs
    for (const [type, data] of Object.entries(media)) {
      await DatabaseAgent.trackMediaCost(
        contentId,
        type,
        data.url,
        data.provider,
        data.cost
      );
    }
    
    return {
      contentId: contentId,
      filePath: filePath,
      media: media,
      research: research
    };
  }
  
  async auditContent() {
    // Use MCP to analyze existing content
    const files = await ContentStorageAgent.listContent();
    const audit = [];
    
    for (const file of files) {
      const slug = file.replace('.md', '');
      const analytics = await DatabaseAgent.getContentAnalytics(slug);
      
      audit.push({
        file: file,
        analytics: analytics,
        recommendations: this.generateRecommendations(analytics)
      });
    }
    
    return audit;
  }
}
```

### Part 6: Security & Isolation (10 min)

#### Implement MCP Security Best Practices

```javascript
// Security configuration
const MCPSecurity = {
  // Limit file system access
  filesystem: {
    allowedPaths: ['/workspace/content'],
    deniedPaths: ['/etc', '/usr', '/system'],
    maxFileSize: 10 * 1024 * 1024, // 10MB
    allowedExtensions: ['.md', '.json', '.txt', '.csv']
  },
  
  // Database access control
  database: {
    allowedTables: ['content', 'media', 'analytics'],
    readOnly: false,
    maxQueryTime: 5000, // 5 seconds
    preventDrops: true
  },
  
  // Rate limiting
  rateLimits: {
    filesystem: { calls: 100, per: 60000 },
    database: { calls: 200, per: 60000 },
    websearch: { calls: 50, per: 60000 }
  }
};

// Validation middleware
function validateMCPRequest(tool, operation, params) {
  // Check permissions
  if (tool === 'filesystem') {
    const path = params.path;
    if (!MCPSecurity.filesystem.allowedPaths.some(p => path.startsWith(p))) {
      throw new Error(`Access denied: ${path}`);
    }
  }
  
  // Check rate limits
  if (!checkRateLimit(tool)) {
    throw new Error(`Rate limit exceeded for ${tool}`);
  }
  
  return true;
}
```

## 💡 Best Practices

1. **Connection Management**
   - Keep MCP servers running as long-lived processes
   - Implement reconnection logic
   - Monitor server health

2. **Error Handling**
   - Gracefully handle MCP server failures
   - Provide fallback mechanisms
   - Log all MCP operations

3. **Performance**
   - Cache frequently accessed data
   - Batch database operations
   - Use connection pooling

## ✅ Success Criteria

- [ ] MCP servers installed and configured
- [ ] File system operations working
- [ ] Database integration complete
- [ ] Web search implemented
- [ ] Multi-tool workflow created

## 🚀 Bonus Challenge

Create an "Intelligent Content Assistant" that:
1. Monitors content performance via database
2. Suggests improvements based on analytics
3. Automatically updates underperforming content
4. Generates reports using file system access
5. Researches trending topics via web search

## 📊 Expected Output

```json
{
  "campaign": "AI Content Strategy 2024",
  "mcp_operations": {
    "research": {
      "searches_performed": 5,
      "sources_found": 23,
      "insights_extracted": 12
    },
    "storage": {
      "files_created": 4,
      "database_records": 4,
      "total_size": "245KB"
    },
    "media": {
      "images_generated": 3,
      "videos_created": 1,
      "total_cost": 0.75
    }
  },
  "performance": {
    "total_time": 34.5,
    "mcp_calls": 27,
    "success_rate": "100%"
  }
}
```

## 🔗 Resources

- [MCP Documentation](https://modelcontextprotocol.io/docs)
- [Official MCP Servers](https://github.com/modelcontextprotocol)
- [MCP Security Guide](../../resources/mcp-security.md)

## 📝 Solution

<details>
<summary>Click to reveal complete MCP integration</summary>

```javascript
// Complete MCP Integration for ContentFlow AI
class ContentFlowMCP {
  constructor() {
    this.servers = this.initializeServers();
    this.agents = this.initializeAgents();
  }
  
  initializeServers() {
    return {
      filesystem: new MCPFileSystemServer({
        root: '/workspace/content',
        permissions: 'rw'
      }),
      database: new MCPPostgresServer({
        connectionString: process.env.DATABASE_URL
      }),
      websearch: new MCPWebSearchServer({
        apiKey: process.env.SEARCH_API_KEY
      })
    };
  }
  
  initializeAgents() {
    return {
      storage: new ContentStorageAgent(this.servers.filesystem),
      database: new DatabaseAgent(this.servers.database),
      research: new ResearchAgent(this.servers.websearch)
    };
  }
  
  async executeContentWorkflow(brief) {
    try {
      // Complete workflow with all MCP tools
      const research = await this.agents.research.research(brief.topic);
      const content = await this.generateContent(brief, research);
      const stored = await this.agents.storage.saveContent(content);
      const dbRecord = await this.agents.database.storeContent(content);
      
      return {
        success: true,
        contentId: dbRecord.id,
        path: stored.path,
        research: research
      };
    } catch (error) {
      console.error('MCP Workflow Error:', error);
      return this.handleError(error);
    }
  }
}
```

</details>

## Next Exercise
[Exercise 13.1: Complete ContentFlow AI System →](../13-complete-system/final-integration.md)
