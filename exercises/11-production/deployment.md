# Exercise 11.1: Production Deployment

## 🎯 Learning Goals
- Deploy ContentFlow AI to production
- Implement environment management
- Set up monitoring and alerting
- Configure auto-scaling and failover

## 📋 Prerequisites
- Complete ContentFlow AI system
- Server/cloud access (or local Docker)
- Understanding of deployment concepts

## 🔨 Task Description

Deploy your ContentFlow AI system to a production environment with proper monitoring, security, and scalability.

### Part 1: Environment Configuration (20 min)

#### Setup Multiple Environments

```javascript
// Environment Configuration
const environments = {
  development: {
    name: 'dev',
    n8n_url: 'http://localhost:5678',
    api_endpoints: {
      claude: 'https://api.anthropic.com/v1/messages',
      falai: 'https://fal.run/fal-ai/',
      unsplash: 'https://api.unsplash.com/'
    },
    database: {
      host: 'localhost',
      port: 5432,
      name: 'contentflow_dev'
    },
    features: {
      debug: true,
      mockAPIs: true,
      rateLimit: false
    }
  },
  
  staging: {
    name: 'staging',
    n8n_url: 'https://staging-n8n.contentflow.ai',
    api_endpoints: {
      claude: 'https://api.anthropic.com/v1/messages',
      falai: 'https://fal.run/fal-ai/',
      unsplash: 'https://api.unsplash.com/'
    },
    database: {
      host: 'staging-db.contentflow.ai',
      port: 5432,
      name: 'contentflow_staging'
    },
    features: {
      debug: true,
      mockAPIs: false,
      rateLimit: true
    }
  },
  
  production: {
    name: 'production',
    n8n_url: 'https://n8n.contentflow.ai',
    api_endpoints: {
      claude: 'https://api.anthropic.com/v1/messages',
      falai: 'https://fal.run/fal-ai/',
      unsplash: 'https://api.unsplash.com/'
    },
    database: {
      host: 'prod-db.contentflow.ai',
      port: 5432,
      name: 'contentflow_prod',
      ssl: true,
      poolSize: 20
    },
    features: {
      debug: false,
      mockAPIs: false,
      rateLimit: true,
      monitoring: true,
      alerting: true
    }
  }
};
```

#### Docker Configuration

```dockerfile
# Dockerfile for ContentFlow AI
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy application
COPY . .

# Create non-root user
RUN addgroup -g 1001 -S nodejs
RUN adduser -S nodejs -u 1001
USER nodejs

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD node healthcheck.js

EXPOSE 5678

CMD ["npm", "start"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n
    restart: always
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - NODE_ENV=production
    volumes:
      - n8n_data:/home/node/.n8n
      - ./workflows:/workflows
    depends_on:
      - postgres
    networks:
      - contentflow

  postgres:
    image: postgres:14
    restart: always
    environment:
      - POSTGRES_USER=${POSTGRES_USER}
      - POSTGRES_PASSWORD=${POSTGRES_PASSWORD}
      - POSTGRES_DB=${POSTGRES_DB}
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init.sql:/docker-entrypoint-initdb.d/init.sql
    networks:
      - contentflow

  redis:
    image: redis:7-alpine
    restart: always
    volumes:
      - redis_data:/data
    networks:
      - contentflow

volumes:
  n8n_data:
  postgres_data:
  redis_data:

networks:
  contentflow:
    driver: bridge
```

### Part 2: Security Configuration (20 min)

#### Implement Security Best Practices

```javascript
// Security Manager
class SecurityManager {
  constructor() {
    this.secrets = new SecretManager();
    this.encryption = new EncryptionService();
    this.audit = new AuditLogger();
  }
  
  // Secret Management
  async getSecret(key) {
    // Use environment variables in dev
    if (process.env.NODE_ENV === 'development') {
      return process.env[key];
    }
    
    // Use secret manager in production
    return await this.secrets.retrieve(key);
  }
  
  // API Key Rotation
  async rotateAPIKeys() {
    const keys = [
      'CLAUDE_API_KEY',
      'FALAI_API_KEY',
      'UNSPLASH_ACCESS_KEY'
    ];
    
    for (const key of keys) {
      const newKey = await this.generateNewKey(key);
      await this.secrets.update(key, newKey);
      await this.notifyKeyRotation(key);
    }
  }
  
  // Request Validation
  validateRequest(req) {
    // Check authentication
    if (!req.headers.authorization) {
      throw new Error('Authentication required');
    }
    
    // Validate API key
    const apiKey = req.headers['x-api-key'];
    if (!this.isValidAPIKey(apiKey)) {
      this.audit.log('Invalid API key attempt', { ip: req.ip });
      throw new Error('Invalid API key');
    }
    
    // Rate limiting
    if (!this.checkRateLimit(apiKey)) {
      throw new Error('Rate limit exceeded');
    }
    
    return true;
  }
}
```

### Part 3: Monitoring & Alerting (25 min)

#### Implement Comprehensive Monitoring

```javascript
// Monitoring System
class MonitoringSystem {
  constructor() {
    this.metrics = new MetricsCollector();
    this.alerts = new AlertManager();
    this.healthChecks = new HealthCheckService();
  }
  
  // Metrics Collection
  collectMetrics() {
    return {
      system: this.collectSystemMetrics(),
      application: this.collectApplicationMetrics(),
      business: this.collectBusinessMetrics()
    };
  }
  
  collectSystemMetrics() {
    return {
      cpu: process.cpuUsage(),
      memory: process.memoryUsage(),
      uptime: process.uptime(),
      diskUsage: this.getDiskUsage(),
      networkLatency: this.measureNetworkLatency()
    };
  }
  
  collectApplicationMetrics() {
    return {
      requestsPerSecond: this.metrics.get('requests_per_second'),
      averageResponseTime: this.metrics.get('avg_response_time'),
      errorRate: this.metrics.get('error_rate'),
      activeWorkflows: this.metrics.get('active_workflows'),
      queueSize: this.metrics.get('queue_size')
    };
  }
  
  collectBusinessMetrics() {
    return {
      contentGenerated: this.metrics.get('content_generated'),
      apiCosts: this.metrics.get('api_costs'),
      successRate: this.metrics.get('success_rate'),
      averageQuality: this.metrics.get('avg_quality_score')
    };
  }
  
  // Health Checks
  async performHealthCheck() {
    const checks = {
      n8n: await this.checkN8N(),
      database: await this.checkDatabase(),
      apis: await this.checkAPIs(),
      storage: await this.checkStorage()
    };
    
    const overall = Object.values(checks).every(c => c.healthy);
    
    return {
      healthy: overall,
      checks: checks,
      timestamp: new Date().toISOString()
    };
  }
  
  // Alert Configuration
  setupAlerts() {
    // Critical alerts
    this.alerts.register({
      name: 'High Error Rate',
      condition: () => this.metrics.get('error_rate') > 0.05,
      severity: 'critical',
      action: 'page_oncall'
    });
    
    this.alerts.register({
      name: 'API Failure',
      condition: () => this.metrics.get('api_failures') > 3,
      severity: 'critical',
      action: 'page_oncall'
    });
    
    // Warning alerts
    this.alerts.register({
      name: 'High API Costs',
      condition: () => this.metrics.get('api_costs') > 100,
      severity: 'warning',
      action: 'email_team'
    });
  }
}
```

### Part 4: Auto-scaling Configuration (15 min)

#### Implement Auto-scaling

```javascript
// Auto-scaling Manager
class AutoScalingManager {
  constructor() {
    this.minInstances = 1;
    this.maxInstances = 10;
    this.currentInstances = 1;
    this.metrics = new MetricsCollector();
  }
  
  async evaluateScaling() {
    const metrics = {
      cpu: await this.metrics.getAverage('cpu_usage'),
      memory: await this.metrics.getAverage('memory_usage'),
      queueSize: await this.metrics.get('queue_size'),
      responseTime: await this.metrics.getAverage('response_time')
    };
    
    const decision = this.makeScalingDecision(metrics);
    
    if (decision !== 0) {
      await this.scale(decision);
    }
    
    return decision;
  }
  
  makeScalingDecision(metrics) {
    // Scale up conditions
    if (metrics.cpu > 70 || 
        metrics.memory > 80 || 
        metrics.queueSize > 100 ||
        metrics.responseTime > 5000) {
      return 1; // Scale up
    }
    
    // Scale down conditions
    if (metrics.cpu < 20 && 
        metrics.memory < 30 && 
        metrics.queueSize < 10 &&
        metrics.responseTime < 1000 &&
        this.currentInstances > this.minInstances) {
      return -1; // Scale down
    }
    
    return 0; // No change
  }
  
  async scale(direction) {
    if (direction > 0 && this.currentInstances < this.maxInstances) {
      await this.addInstance();
      this.currentInstances++;
    } else if (direction < 0 && this.currentInstances > this.minInstances) {
      await this.removeInstance();
      this.currentInstances--;
    }
  }
}
```

### Part 5: Deployment Pipeline (10 min)

#### CI/CD Implementation

```yaml
# .github/workflows/deploy.yml
name: Deploy ContentFlow AI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '18'
      - name: Install dependencies
        run: npm ci
      - name: Run tests
        run: npm test

  build:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Build Docker image
        run: docker build -t contentflow:${{ github.sha }} .
      - name: Push to registry
        run: docker push contentflow:${{ github.sha }}

  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Deploy to production
        run: |
          kubectl set image deployment/contentflow contentflow=contentflow:${{ github.sha }}
          kubectl rollout status deployment/contentflow
```

## 💡 Production Tips

1. **Blue-Green Deployment**: Minimize downtime
2. **Database Migrations**: Version control schemas
3. **Feature Flags**: Gradual rollout
4. **Backup Strategy**: Regular automated backups
5. **Disaster Recovery**: Test recovery procedures

## ✅ Success Criteria

- [ ] Multi-environment setup complete
- [ ] Security measures implemented
- [ ] Monitoring dashboard active
- [ ] Auto-scaling configured
- [ ] CI/CD pipeline working

## 🚀 Bonus Challenge

Implement:
1. **Canary Deployments**: Gradual rollout to subset of users
2. **A/B Testing Infrastructure**: Test variations in production
3. **Distributed Tracing**: Track requests across services
4. **Cost Optimization**: Auto-shutdown unused resources
5. **Compliance Logging**: GDPR/CCPA compliance tracking

## 📊 Expected Output

```json
{
  "deployment": {
    "environment": "production",
    "version": "1.2.3",
    "status": "healthy",
    "instances": 3,
    "uptime": "99.95%"
  },
  "metrics": {
    "requests_per_second": 145,
    "average_response_time": 230,
    "error_rate": 0.001,
    "cpu_usage": 45,
    "memory_usage": 62
  },
  "health": {
    "n8n": "healthy",
    "database": "healthy",
    "apis": "healthy",
    "overall": "healthy"
  },
  "costs": {
    "infrastructure": 250,
    "api_usage": 180,
    "total_monthly": 430
  }
}
```

## 🔗 Resources

- [Docker Best Practices](../../resources/docker-best-practices.md)
- [Kubernetes Deployment](../../resources/k8s-deployment.md)
- [Monitoring Setup](../../resources/monitoring-setup.md)

## Next Exercise
[Exercise 12.1: Performance Optimization →](../12-optimization/performance.md)