# n8n DevOps Incident Automation with LLM

## Description
Technical implementation of a **DevOps incident automation pipeline** using **n8n**.  
The workflow ingests monitoring alerts, applies severity logic, analyzes logs using an LLM, triggers notifications, and persists incident data.


## Stack
- n8n
- Docker / Docker Compose
- OpenAI API (LLM)
- REST Webhooks
- Slack / Email 



## Architecture
Webhook → Function → IF → LLM → Notify → Incident Data Store



## Setup

### 1. Create Project Directory

```bash
mkdir n8n-devops-incident
cd n8n-devops-incident
```


### 2. Docker Compose
```bash
version: "3.8"

services:
  n8n:
    image: n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=admin
      - N8N_BASIC_AUTH_PASSWORD=admin
      - TZ=UTC
    volumes:
      - ./n8n_data:/home/node/.n8n
```

### 3. Start n8n
```bash
docker-compose up -d
```

### 4. Access UI
```bash
http://localhost:5678
```

Credentials:
```bash
admin / admin
```
## Workflow Implementation

### Step 1: Webhook Trigger

* Method: POST
* Path: /incident-alert

###Test Payload
```bash
curl -X POST http://localhost:5678/webhook-test/incident-alert \
-H "Content-Type: application/json" \
-d '{
  "service": "payment-api",
  "metric": "cpu",
  "value": 92,
  "threshold": 80,
  "logs": "CPU spike after deployment v1.2.3"
}'
```
### Step 2: Data Normalization (Function Node)
```bash
return [{
  service: $json.service,
  metric: $json.metric,
  value: $json.value,
  severity: $json.value >= 90 ? "critical" : "warning",
  logs: $json.logs,
  timestamp: new Date().toISOString()
}];
```

### Step 3: Decision Logic (IF Node)
Condition:


severity == "critical"

* True → AI analysis path
* False → Skip / Log only

### Step 4: LLM Log Analysis (HTTP Request Node)
Method: POST

URL:

```bash
https://api.openai.com/v1/chat/completions
```
Headers:

```bash
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json
```

Body:

```bash
{
  "model": "gpt-4o-mini",
  "messages": [
    {
      "role": "system",
      "content": "You are a DevOps incident analysis assistant."
    },
    {
      "role": "user",
      "content": "Analyze logs and provide root cause and remediation steps:\n{{$json.logs}}"
    }
  ]
}
```

### Step 5: Extract LLM Output (Function Node)
```bash
return [{
  root_cause: $json.choices[0].message.content
}];
```

### Step 6: Notification (ChatOps)
Example Slack message:

```bash
🚨 CRITICAL INCIDENT
Service: {{ $json.service }}
Root Cause:
{{ $json.root_cause }}
```

### Step 7: Incident Storage
Store JSON output using:

* Write Binary File
* Database node (PostgreSQL / SQLite)

Example record:

```bash
{
  "service": "payment-api",
  "severity": "critical",
  "root_cause": "High CPU after deployment v1.2.3",
  "timestamp": "2025-12-31T12:00:00Z"
}
```

### Export Workflow
```bash
Workflow → Export → JSON
```
Save as:
```bash
workflows/incident-automation.json
```
