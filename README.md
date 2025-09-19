# Kong AI Gateway Workshop: Step-by-Step Setup Guide

## Prerequisites
- Docker and Docker Compose installed
- OpenAI API key (or access to local LLM like Ollama)
- Basic understanding of APIs and containers
- Terminal/command line access

## Quick Start Guide

### Step 1: Environment Setup

1. **Create workshop directory**:
   ```bash
   mkdir kong-ai-workshop
   cd kong-ai-workshop
   ```

2. **Clone Git Repo**:
   ```bash
   git clone https://github.com/r-rai/kong-ai-gateway-demo.git
   cd kong-ai-gateway-demo```

3. **Configure environment variables**:
   Edit `.env` file and add your OpenAI API key:
   ```bash
   OPENAI_API_KEY=sk-your-actual-openai-key-here
   ```

### Step 2: Start the Kong AI Gateway Stack

1. **Start all services**:
   ```bash
   docker compose up -d kong-database
   #wait for DB to come up and become healthy
   docker compose run --rm kong-migrations
   # after running of kong-migration bring down everything
   docker compose down
   #start all services
   docker compose up -d
   ```

2. **Check service health**:
   ```bash
   docker ps
   ```

3. **Verify Kong Gateway is running**:
   ```bash
   curl -i http://localhost:8001/
   #Expected response: Kong Admin API information
   ```
4. **You can access Kong Manager**:
   ```bash
   curl -i http://localhost:8002/
   ```

### Step 3: Configure AI Proxy Plugin

#### Create a Service for AI Provider

```bash
curl --location 'http://localhost:8001/services/' \
--header 'Content-Type: application/json' \
--data '{
    "name": "openai-service", 
    "url": "https://api.openai.com/v1"
  }'
```

#### Create a Route for Chat Endpoint

```bash
curl --location 'http://localhost:8001/routes' \
--header 'Content-Type: application/json' \
--data '{
    "name": "ai-chat-route",
    "paths": ["/ai"],
    "service": {"name": "openai-service"}
  }'
```

#### Configure AI Proxy Plugin (replace YOUR_OPENAI_KEY_HERE with you open API Key)

```bash
curl --location 'http://localhost:8001/plugins/' \
--header 'Content-Type: application/json' \
--data '{
    "name": "ai-proxy",
    "route": {"name": "ai-chat-route"},
    "config": {
      "route_type": "llm/v1/chat",
      "model": {
        "provider": "openai",
        "name": "gpt-3.5-turbo",
        "options": {
          "temperature": 0.7,
          "max_tokens": 500
        }
      },
      "auth": {
        "header_name": "Authorization",
        "header_value": "Bearer YOUR_OPENAI_KEY_HERE"
      }
    }
  }'
```

#### Test AI Proxy

```bash
curl --location 'http://localhost:8000/ai/llm/v1/chat' \
--header 'Content-Type: application/json' \
--data '{
    "messages": [
      {"role": "user", "content": "Explain quantum computing in simple terms"}
    ]
  }'
```

### Step 4: Configure AI Prompt Template Plugin

#### Enable AI Prompt Template Plugin

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "ai-prompt-template",
    "route": {"name": "ai-chat-route"},
    "config": {
      "templates": [
        {
          "name": "code-assistant",
          "template": {
            "messages": [
              {
                "role": "system", 
                "content": "You are a {{language}} programming expert."
              },
              {
                "role": "user",
                "content": "Write a {{language}} function to {{task}}."
              }
            ]
          }
        },
        {
          "name": "translator", 
          "template": {
            "messages": [
              {
                "role": "system",
                "content": "You are a professional translator."
              },
              {
                "role": "user", 
                "content": "Translate the following text from {{from_lang}} to {{to_lang}}: {{text}}"
              }
            ]
          }
        }
      ]
    }
  }'
```

#### Test Prompt Template

```bash
curl -i -X POST http://localhost:8000/chat \
  --header "Content-Type: application/json" \
  --data '{
    "template": "code-assistant",
    "properties": {
      "language": "Python",
      "task": "sort a list of numbers"
    }
  }'
```

### Step 5: Configure AI Prompt Guard Plugin

#### Enable AI Prompt Guard for Security

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "ai-prompt-guard", 
    "route": {"name": "ai-chat-route"},
    "config": {
      "allow_patterns": [
        ".*code.*",
        ".*help.*",
        ".*explain.*"
      ],
      "deny_patterns": [
        ".*hack.*",
        ".*password.*",
        ".*illegal.*"
      ]
    }
  }'
```

#### Test Prompt Guard (Should be blocked)

```bash
curl -i -X POST http://localhost:8000/chat \
  --header "Content-Type: application/json" \
  --data '{
    "messages": [
      {"role": "user", "content": "How to hack into a system?"}
    ]
  }'
```

### Step 6: Enable Monitoring with Prometheus Plugin

#### Configure Prometheus Plugin

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "prometheus",
    "config": {
      "status_code_metrics": true,
      "latency_metrics": true, 
      "bandwidth_metrics": true,
      "upstream_health_metrics": true,
      "ai_metrics": true
    }
  }'
```

#### Verify Metrics Endpoint

```bash
curl http://localhost:8100/metrics
```

### Step 7: Configure Rate Limiting

#### Add Rate Limiting Plugin

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "rate-limiting",
    "route": {"name": "ai-chat-route"},
    "config": {
      "minute": 10,
      "hour": 100,
      "policy": "local"
    }
  }'
```

### Step 8: Access Monitoring Dashboards

#### Prometheus Dashboard
- URL: http://localhost:9090
- Check targets: http://localhost:9090/targets
- Query example: `kong_http_requests_total`

#### Grafana Dashboard  
- URL: http://localhost:3000
- Username: `admin`
- Password: `admin123`
- Add Prometheus data source: `http://prometheus:9090`

### Step 9: Advanced Configuration Examples

#### AI Request Transformer Example

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "ai-request-transformer",
    "route": {"name": "ai-chat-route"},
    "config": {
      "llm": {
        "provider": "openai",
        "model": "gpt-3.5-turbo"
      },
      "prompt": "Enhance this request for better AI processing: {{request_body}}"
    }
  }'
```

#### AI Response Transformer Example

```bash
curl -i -X POST http://localhost:8001/plugins/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "ai-response-transformer", 
    "route": {"name": "ai-chat-route"},
    "config": {
      "llm": {
        "provider": "openai",
        "model": "gpt-3.5-turbo"
      },
      "prompt": "Summarize this AI response in 2 sentences: {{response_body}}"
    }
  }'
```

## Testing Scenarios

### Scenario 1: Multi-Provider Setup

Test switching between different AI providers:

```bash
# Create another service for different provider
curl -i -X POST http://localhost:8001/services/ \
  --header "Content-Type: application/json" \
  --data '{
    "name": "anthropic-service",
    "url": "https://api.anthropic.com"
  }'

# Create route with different AI configuration
curl -i -X POST http://localhost:8001/routes \
  --header "Content-Type: application/json" \
  --data '{
    "name": "anthropic-route", 
    "paths": ["/claude"],
    "service": {"name": "anthropic-service"}
  }'
```

### Scenario 2: Load Testing

Generate traffic to test rate limiting:

```bash
for i in {1..15}; do
  curl -X POST http://localhost:8000/chat \
    --header "Content-Type: application/json" \
    --data '{"messages":[{"role":"user","content":"Test message '$i'"}]}'
  sleep 1
done
```

## Troubleshooting

### Common Issues

1. **Kong Gateway won't start**:
   ```bash
   docker-compose logs kong-gateway
   ```

2. **AI plugin not working**:
   - Check API key configuration
   - Verify plugin is enabled: `curl http://localhost:8001/plugins`
   - Check Kong logs for errors

3. **Prometheus metrics not appearing**:
   ```bash
   curl http://localhost:8100/metrics | grep kong
   ```

4. **Rate limiting not working**:
   - Check plugin configuration
   - Send multiple requests quickly to trigger

### Useful Commands

#### List all services
```bash
curl http://localhost:8001/services | jq
```

#### List all routes  
```bash
curl http://localhost:8001/routes | jq
```

#### List all plugins
```bash
curl http://localhost:8001/plugins | jq
```

#### Check plugin status
```bash
curl http://localhost:8001/plugins/{plugin-id} | jq
```

## Workshop Cleanup

Stop and remove all containers:
```bash
docker-compose down -v
```

## Next Steps

1. **Production Deployment**: 
   - Use external database
   - Configure SSL certificates
   - Set up proper authentication

2. **Advanced Features**:
   - Explore Kong Enterprise AI plugins
   - Implement semantic routing
   - Configure advanced caching

3. **Integration**:
   - Connect to existing APIs  
   - Implement in CI/CD pipeline
   - Add custom plugins

## Resources

- [Kong AI Gateway Documentation](https://docs.konghq.com/ai-gateway/)
- [Kong Admin API Reference](https://docs.konghq.com/gateway/admin-api/)
- [Kong Plugin Development](https://docs.konghq.com/gateway/custom-plugins/)
- [Kong Community Forum](https://discuss.konghq.com/)

---

**Workshop Complete! 🎉**

You now have a fully functional Kong AI Gateway with monitoring, security, and multiple AI plugins configured.
