# Temporal.io Implementation Guide - Workflows, Workers & Containerization

## 1. Understanding Temporal.io Architecture

### Core Concepts

**Temporal Server:**

- Central orchestration service
- Maintains workflow state
- Handles task queues
- Provides durability and reliability
- Runs continuously as a separate service

**Workers:**

- Containers/processes that execute workflow code
- Poll Temporal Server for tasks
- Can scale horizontally
- Stateless - all state is in Temporal Server

**Workflows:**

- Orchestration logic (what to do)
- Deterministic and resumable
- Can run for days/weeks/months
- State persisted by Temporal Server

**Activities:**

- Individual units of work (actual business logic)
- Non-deterministic operations (API calls, DB queries)
- Can fail and retry independently
- Executed by Workers

---

## 2. Container Architecture

### Container Layout

```
┌─────────────────────────────────────────────────────────────┐
│                    Temporal Server Cluster                   │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐      │
│  │   Frontend   │  │   History    │  │   Matching   │      │
│  │   Service    │  │   Service    │  │   Service    │      │
│  └──────────────┘  └──────────────┘  └──────────────┘      │
│                          ▼                                    │
│                  ┌──────────────┐                            │
│                  │  PostgreSQL  │                            │
│                  │  (State DB)  │                            │
│                  └──────────────┘                            │
└─────────────────────────────────────────────────────────────┘
                           ▲
                           │ gRPC (Port 7233)
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌──────────────┐
│   Worker 1   │  │   Worker 2   │  │   Worker N   │
│ (Container)  │  │ (Container)  │  │ (Container)  │
├──────────────┤  ├──────────────┤  ├──────────────┤
│ - Workflows  │  │ - Workflows  │  │ - Workflows  │
│ - Activities │  │ - Activities │  │ - Activities │
└──────────────┘  └──────────────┘  └──────────────┘
        │                  │                  │
        └──────────────────┼──────────────────┘
                           │
                           ▼
        ┌─────────────────────────────────────┐
        │    Your Services (API, Database)    │
        │  - Organization Service             │
        │  - Library Service                  │
        │  - Delivery Service                 │
        │  - Reporting Service                │
        └─────────────────────────────────────┘
```

### Docker Compose Configuration

```yaml
version: "3.8"

services:
  # Temporal Server
  temporal:
    image: temporalio/auto-setup:1.22.0
    container_name: temporal-server
    ports:
      - "7233:7233" # gRPC endpoint
      - "8233:8233" # Web UI
    environment:
      - DB=postgresql
      - DB_PORT=5432
      - POSTGRES_USER=temporal
      - POSTGRES_PWD=temporal
      - POSTGRES_SEEDS=temporal-postgres
      - DYNAMIC_CONFIG_FILE_PATH=config/dynamicconfig/development-sql.yaml
    depends_on:
      - temporal-postgres
    networks:
      - survey-network
    volumes:
      - ./temporal-config:/etc/temporal/config/dynamicconfig

  # Temporal Database
  temporal-postgres:
    image: postgres:15
    container_name: temporal-postgres
    environment:
      POSTGRES_USER: temporal
      POSTGRES_PASSWORD: temporal
      POSTGRES_DB: temporal
    ports:
      - "5433:5432"
    volumes:
      - temporal-db-data:/var/lib/postgresql/data
    networks:
      - survey-network

  # Temporal Web UI
  temporal-ui:
    image: temporalio/ui:2.21.0
    container_name: temporal-ui
    ports:
      - "8080:8080"
    environment:
      - TEMPORAL_ADDRESS=temporal:7233
      - TEMPORAL_CORS_ORIGINS=http://localhost:3000
    depends_on:
      - temporal
    networks:
      - survey-network

  # Reporting Service (Your API)
  reporting-service:
    build: ./services/reporting
    container_name: reporting-service
    ports:
      - "3001:3001"
    environment:
      - DATABASE_URL=postgresql://user:pass@reporting-postgres:5432/reporting_db
      - TEMPORAL_HOST=temporal:7233
      - REDIS_URL=redis://redis:6379
    depends_on:
      - reporting-postgres
      - temporal
    networks:
      - survey-network

  # Temporal Worker for Reporting (This is key!)
  reporting-worker:
    build:
      context: ./services/reporting
      dockerfile: Dockerfile.worker
    container_name: reporting-worker
    environment:
      - TEMPORAL_HOST=temporal:7233
      - TEMPORAL_NAMESPACE=default
      - TASK_QUEUE=reporting-tasks
      - DATABASE_URL=postgresql://user:pass@reporting-postgres:5432/reporting_db
      - OPENAI_API_KEY=${OPENAI_API_KEY}
      - REPORTING_SERVICE_URL=http://reporting-service:3001
    depends_on:
      - temporal
      - reporting-service
    networks:
      - survey-network
    deploy:
      replicas: 3 # Multiple workers for scalability
      resources:
        limits:
          cpus: "2"
          memory: 4G

  # Organization Worker
  organization-worker:
    build:
      context: ./services/organization
      dockerfile: Dockerfile.worker
    container_name: organization-worker
    environment:
      - TEMPORAL_HOST=temporal:7233
      - TASK_QUEUE=organization-tasks
    depends_on:
      - temporal
    networks:
      - survey-network
    deploy:
      replicas: 2

  # Other services...
  reporting-postgres:
    image: postgres:15
    # ... config

  redis:
    image: redis:7-alpine
    # ... config

networks:
  survey-network:
    driver: bridge

volumes:
  temporal-db-data:
  reporting-db-data:
```

---

## 3. How Workflows Actually Work

### The Complete Flow

```
┌──────────────┐
│   User API   │
│   Request    │
└──────┬───────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 1: API Endpoint Triggers Workflow                │
│  POST /api/v1/reports/generate/survey-123              │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 2: API Service Starts Workflow                   │
│                                                         │
│  const client = new WorkflowClient();                  │
│  const handle = await client.start(                   │
│    ReportGenerationWorkflow,                           │
│    {                                                   │
│      taskQueue: 'reporting-tasks',                    │
│      workflowId: 'report-survey-123',                 │
│      args: [{ surveyId: 'survey-123' }]              │
│    }                                                   │
│  );                                                    │
│                                                         │
│  // Return immediately to user                         │
│  return { workflowId: handle.workflowId };            │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 3: Temporal Server Receives Workflow Start      │
│  - Assigns unique workflow ID                          │
│  - Creates workflow execution                          │
│  - Puts first task in queue "reporting-tasks"         │
│  - Stores state in PostgreSQL                          │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 4: Worker Polls for Tasks                        │
│                                                         │
│  Worker Container (running continuously):              │
│  while (true) {                                        │
│    task = poll("reporting-tasks");                    │
│    if (task) execute(task);                           │
│  }                                                     │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 5: Worker Executes Workflow Code                │
│                                                         │
│  async execute(surveyId) {                            │
│    // Activity 1                                       │
│    const data = await activities.fetchData(surveyId); │
│    // Activity 2                                       │
│    const stats = await activities.calcStats(data);    │
│    // Activity 3                                       │
│    const ai = await activities.runAI(data);           │
│    return { stats, ai };                              │
│  }                                                     │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 6: Temporal Orchestrates Activities              │
│  - Each activity is a separate task                    │
│  - State saved after each activity                     │
│  - Automatic retries on failure                        │
│  - Can take hours/days - state is durable             │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 7: Workflow Completes                            │
│  - Final state persisted                               │
│  - Result stored in Temporal                           │
│  - Workflow marked as complete                         │
└──────┬─────────────────────────────────────────────────┘
       │
       ▼
┌────────────────────────────────────────────────────────┐
│  Step 8: Your API Retrieves Result                     │
│                                                         │
│  GET /api/v1/reports/survey-123/status                │
│                                                         │
│  const handle = client.getHandle('report-survey-123');│
│  const status = await handle.describe();               │
│  if (status.status === 'COMPLETED') {                 │
│    const result = await handle.result();              │
│    return result;                                      │
│  }                                                     │
└────────────────────────────────────────────────────────┘
```

---

## 4. Practical Implementation Code

### API Service (Triggers Workflows)

```typescript
// reporting-service/src/controllers/report.controller.ts

import { WorkflowClient } from "@temporalio/client";
import { ReportGenerationWorkflow } from "../workflows";

export class ReportController {
  private temporalClient: WorkflowClient;

  constructor() {
    this.temporalClient = new WorkflowClient({
      connection: {
        address: process.env.TEMPORAL_HOST || "temporal:7233",
      },
    });
  }

  /**
   * Endpoint: POST /api/v1/reports/generate/:surveyId
   * This triggers the workflow and returns immediately
   */
  async generateReport(req: Request, res: Response) {
    const { surveyId } = req.params;
    const userId = req.user.id;

    try {
      // Start workflow (non-blocking)
      const handle = await this.temporalClient.start(ReportGenerationWorkflow, {
        taskQueue: "reporting-tasks",
        workflowId: `report-${surveyId}-${Date.now()}`,
        args: [
          {
            surveyId,
            userId,
            organizationId: req.user.organizationId,
            includeAI: true,
          },
        ],
      });

      // Return immediately - workflow runs in background
      return res.status(202).json({
        message: "Report generation started",
        workflowId: handle.workflowId,
        statusUrl: `/api/v1/reports/${surveyId}/status`,
        estimatedTime: "2-5 minutes",
      });
    } catch (error) {
      return res.status(500).json({ error: error.message });
    }
  }

  /**
   * Endpoint: GET /api/v1/reports/:surveyId/status
   * Check workflow status
   */
  async getReportStatus(req: Request, res: Response) {
    const { surveyId } = req.params;

    try {
      // Find the workflow
      const workflowId = await this.findWorkflowBySurvey(surveyId);
      const handle = this.temporalClient.getHandle(workflowId);

      // Get current status
      const description = await handle.describe();

      if (description.status.name === "RUNNING") {
        return res.json({
          status: "generating",
          progress: await this.getProgress(workflowId),
          message: "Report is being generated",
        });
      }

      if (description.status.name === "COMPLETED") {
        // Get the result
        const result = await handle.result();

        return res.json({
          status: "completed",
          reportId: result.reportId,
          reportUrl: `/api/v1/reports/${surveyId}`,
          generatedAt: result.generatedAt,
        });
      }

      if (description.status.name === "FAILED") {
        return res.status(500).json({
          status: "failed",
          error: "Report generation failed",
        });
      }
    } catch (error) {
      return res.status(404).json({ error: "Report not found" });
    }
  }

  /**
   * Endpoint: GET /api/v1/reports/:surveyId
   * Get completed report
   */
  async getReport(req: Request, res: Response) {
    const { surveyId } = req.params;

    // Check if report exists in database
    const report = await this.reportRepository.findBySurveyId(surveyId);

    if (!report) {
      return res.status(404).json({ error: "Report not found" });
    }

    if (report.status !== "completed") {
      return res.status(202).json({
        status: report.status,
        message: "Report is still generating",
      });
    }

    // Return the report data
    return res.json(report);
  }
}
```

### Worker Service (Executes Workflows)

```typescript
// reporting-service/src/worker/index.ts

import { Worker, NativeConnection } from "@temporalio/worker";
import * as activities from "./activities";

async function runWorker() {
  // Connect to Temporal Server
  const connection = await NativeConnection.connect({
    address: process.env.TEMPORAL_HOST || "temporal:7233",
  });

  // Create worker
  const worker = await Worker.create({
    connection,
    namespace: "default",
    taskQueue: "reporting-tasks", // THIS IS KEY - must match trigger
    workflowsPath: require.resolve("./workflows"), // Workflow definitions
    activities, // Activity implementations
    maxConcurrentActivityTaskExecutions: 10,
    maxConcurrentWorkflowTaskExecutions: 5,
  });

  console.log("🔄 Reporting Worker started");
  console.log("📋 Listening on task queue: reporting-tasks");

  // Run worker (this blocks - keeps container running)
  await worker.run();
}

runWorker().catch((err) => {
  console.error("Worker failed:", err);
  process.exit(1);
});
```

### Workflow Definition

```typescript
// reporting-service/src/worker/workflows/report-generation.workflow.ts

import { proxyActivities } from "@temporalio/workflow";
import type * as activities from "../activities";

// Create activity proxies with timeouts
const {
  validateSurveyAccess,
  extractSurveyData,
  generateBasicStatistics,
  generateSentimentAnalysis,
  generateThemeAnalysis,
  generateVisualizationData,
  saveReportComponent,
  compileReport,
  notifyCompletion,
} = proxyActivities<typeof activities>({
  startToCloseTimeout: "5 minutes",
  retry: {
    initialInterval: "1s",
    maximumInterval: "30s",
    maximumAttempts: 3,
  },
});

export async function ReportGenerationWorkflow(request: ReportGenerationRequest): Promise<GeneratedReport> {
  console.log(`Starting report generation for survey: ${request.surveyId}`);

  // Step 1: Validate (fast)
  await validateSurveyAccess(request.surveyId, request.userId);

  // Step 2: Extract data (fast)
  const rawData = await extractSurveyData(request.surveyId);

  // Step 3: Basic stats (fast)
  const basicStats = await generateBasicStatistics(rawData);
  const reportId = await saveReportComponent("basic_stats", basicStats);

  // Step 4: Parallel AI processing (slow)
  const [sentiment, themes, visualizations] = await Promise.all([
    generateSentimentAnalysis(rawData),
    generateThemeAnalysis(rawData),
    generateVisualizationData(rawData),
  ]);

  // Step 5: Save components
  await saveReportComponent(reportId, "sentiment", sentiment);
  await saveReportComponent(reportId, "themes", themes);
  await saveReportComponent(reportId, "visualizations", visualizations);

  // Step 6: Compile final report
  const finalReport = await compileReport(reportId, {
    basicStats,
    sentiment,
    themes,
    visualizations,
  });

  // Step 7: Notify user
  await notifyCompletion(request.userId, finalReport);

  console.log(`Report generation completed: ${reportId}`);

  return finalReport;
}
```

### Activity Implementations

```typescript
// reporting-service/src/worker/activities/index.ts

import { OpenAI } from "openai";
import { ReportRepository } from "../../repositories";

const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

/**
 * Activities are the actual work - these can fail and retry
 */
export async function validateSurveyAccess(surveyId: string, userId: string): Promise<void> {
  // Call your API service
  const response = await fetch(`${process.env.REPORTING_SERVICE_URL}/api/internal/surveys/${surveyId}/access`, {
    headers: { "X-User-Id": userId },
  });

  if (!response.ok) {
    throw new Error("Access denied");
  }
}

export async function extractSurveyData(surveyId: string): Promise<SurveyData> {
  // Call Delivery Service API to get responses
  const response = await fetch(`${process.env.DELIVERY_SERVICE_URL}/api/internal/surveys/${surveyId}/responses`);

  return await response.json();
}

export async function generateBasicStatistics(data: SurveyData): Promise<BasicStats> {
  // Pure computation - fast
  return {
    totalResponses: data.responses.length,
    completionRate: calculateCompletionRate(data),
    averageTime: calculateAverageTime(data),
    responseDistribution: calculateDistribution(data),
  };
}

export async function generateSentimentAnalysis(data: SurveyData): Promise<SentimentAnalysis> {
  // AI call - slow, can fail
  const textResponses = extractTextResponses(data);

  const result = await openai.chat.completions.create({
    model: "gpt-4",
    messages: [
      {
        role: "system",
        content: "Analyze sentiment of survey responses",
      },
      {
        role: "user",
        content: JSON.stringify(textResponses),
      },
    ],
    temperature: 0.1,
  });

  return JSON.parse(result.choices[0].message.content);
}

export async function saveReportComponent(reportId: string, componentType: string, data: any): Promise<void> {
  const repository = new ReportRepository();
  await repository.saveComponent({
    reportId,
    componentType,
    data,
    status: "completed",
  });
}

// ... more activities
```

---

## 5. How to Get Notified When Job Finishes

### Option 1: Polling (Simple)

```typescript
// Frontend polls for status
async function checkReportStatus(surveyId: string) {
  const interval = setInterval(async () => {
    const response = await fetch(`/api/v1/reports/${surveyId}/status`);
    const status = await response.json();

    if (status.status === "completed") {
      clearInterval(interval);
      // Report ready!
      loadReport(surveyId);
    }
  }, 5000); // Check every 5 seconds
}
```

### Option 2: WebSocket (Real-time)

```typescript
// API Service sends updates via WebSocket
export class ReportController {
  async generateReport(req: Request, res: Response) {
    const { surveyId } = req.params;

    // Start workflow
    const handle = await this.temporalClient.start(ReportGenerationWorkflow, {
      taskQueue: "reporting-tasks",
      workflowId: `report-${surveyId}`,
      args: [{ surveyId }],
    });

    // Listen for workflow completion (in background)
    this.listenForCompletion(handle, surveyId);

    return res.json({ workflowId: handle.workflowId });
  }

  private async listenForCompletion(handle: WorkflowHandle, surveyId: string) {
    try {
      // This blocks until workflow completes
      const result = await handle.result();

      // Notify via WebSocket
      this.websocketService.emit(`report:${surveyId}`, {
        status: "completed",
        reportId: result.reportId,
      });
    } catch (error) {
      this.websocketService.emit(`report:${surveyId}`, {
        status: "failed",
        error: error.message,
      });
    }
  }
}

// Frontend listens for updates
const socket = io("http://localhost:3001");
socket.on(`report:${surveyId}`, (data) => {
  if (data.status === "completed") {
    loadReport(data.reportId);
  }
});
```

### Option 3: Webhook (Recommended - Event-Driven)

**Architecture Overview:**

```
Temporal Workflow completes
        ↓
Activity calls webhook with reportId only (lightweight)
        ↓
Reporting Service receives webhook
        ↓
Updates report status in database
        ↓
Notifies user via WebSocket/Email
        ↓
User requests report via reportId
        ↓
Service fetches from database and renders HTML
```

**Important: Webhooks carry metadata, NOT full report data**

```typescript
// Activity in workflow sends webhook
export async function notifyCompletion(userId: string, report: GeneratedReport): Promise<void> {
  // Send lightweight webhook - only metadata, NOT the full report
  await fetch(`${process.env.REPORTING_SERVICE_URL}/api/webhooks/report-completed`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "X-Webhook-Secret": process.env.WEBHOOK_SECRET, // Security
    },
    body: JSON.stringify({
      userId,
      reportId: report.id, // Just the ID!
      surveyId: report.surveyId,
      organizationId: report.organizationId,
      status: "completed",
      componentCount: report.components.length,
      completedAt: new Date().toISOString(),
    }),
  });

  console.log(`Webhook sent for report: ${report.id}`);
}

// Webhook handler in Reporting Service API
app.post("/api/webhooks/report-completed", async (req, res) => {
  const { reportId, userId, surveyId, status } = req.body;

  // Verify webhook signature
  if (!verifyWebhookSignature(req)) {
    return res.status(401).json({ error: "Invalid signature" });
  }

  try {
    // Update report status in database
    await reportRepository.updateStatus(reportId, status);

    // Create notification for user
    await notificationService.create({
      userId,
      type: "report_completed",
      title: "Your survey report is ready!",
      data: {
        reportId,
        surveyId,
        viewUrl: `/reports/${reportId}`,
      },
    });

    // Send real-time notification via WebSocket
    await websocketService.emit(`user:${userId}`, {
      event: "report_completed",
      reportId,
      surveyId,
      message: "Your report is ready to view",
    });

    // Send email notification
    await emailService.send({
      to: await getUserEmail(userId),
      template: "report-ready",
      data: {
        surveyId,
        reportUrl: `${process.env.APP_URL}/reports/${reportId}`,
      },
    });

    console.log(`Report ${reportId} completion processed successfully`);
    res.sendStatus(200);
  } catch (error) {
    console.error("Webhook processing failed:", error);
    res.status(500).json({ error: "Failed to process webhook" });
  }
});
```

---

## 6. Scalability & Multiple Containers

### How Scaling Works

```
Temporal Server (manages all state)
         │
         ├─── Task Queue: "reporting-tasks"
         │    (has 50 pending tasks)
         │
         ▼
    ┌────────────────────────────────┐
    │  3 Worker Containers Running   │
    ├────────────────────────────────┤
    │  Worker 1: Processing Task 1   │
    │  Worker 2: Processing Task 2   │
    │  Worker 3: Processing Task 3   │
    └────────────────────────────────┘
         │
         │ (Workers auto-scale)
         ▼
    ┌────────────────────────────────┐
    │  5 Worker Containers Running   │
    ├────────────────────────────────┤
    │  Worker 1: Processing Task 4   │
    │  Worker 2: Processing Task 5   │
    │  Worker 3: Processing Task 6   │
    │  Worker 4: Processing Task 7   │
    │  Worker 5: Processing Task 8   │
    └────────────────────────────────┘
```

### Kubernetes Auto-Scaling

```yaml
# kubernetes/reporting-worker-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: reporting-worker
spec:
  replicas: 3 # Start with 3
  selector:
    matchLabels:
      app: reporting-worker
  template:
    metadata:
      labels:
        app: reporting-worker
    spec:
      containers:
        - name: worker
          image: your-registry/reporting-worker:latest
          env:
            - name: TEMPORAL_HOST
              value: "temporal.default.svc.cluster.local:7233"
            - name: TASK_QUEUE
              value: "reporting-tasks"
          resources:
            requests:
              memory: "2Gi"
              cpu: "1"
            limits:
              memory: "4Gi"
              cpu: "2"

---
# Auto-scaling based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: reporting-worker-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: reporting-worker
  minReplicas: 3
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

### Task Distribution

```
Time: 10:00 AM
- 10 report requests come in
- Temporal creates 10 workflow tasks in queue
- 3 workers available

Worker 1: Takes task 1, 4, 7, 10
Worker 2: Takes task 2, 5, 8
Worker 3: Takes task 3, 6, 9

Each worker processes its tasks one at a time
No conflicts - Temporal handles distribution
```

---

## 7. Worker Lifecycle Management

### Dockerfile for Worker

```dockerfile
# Dockerfile.worker
FROM node:18-alpine

WORKDIR /app

# Install dependencies
COPY package*.json ./
RUN npm ci --only=production

# Copy worker code
COPY src/worker ./src/worker
COPY src/activities ./src/activities
COPY src/workflows ./src/workflows

# Worker runs indefinitely
CMD ["node", "src/worker/index.js"]
```

### Health Checks

```typescript
// worker/index.ts - with health check
import express from "express";

async function runWorker() {
  // Start Temporal worker
  const worker = await Worker.create({
    connection,
    taskQueue: "reporting-tasks",
    workflowsPath: require.resolve("./workflows"),
    activities,
  });

  // Health check server (for Kubernetes)
  const app = express();

  app.get("/health", (req, res) => {
    res.json({
      status: "healthy",
      taskQueue: "reporting-tasks",
      uptime: process.uptime(),
    });
  });

  app.listen(8080, () => {
    console.log("Health check server running on :8080");
  });

  // Run worker (blocks)
  await worker.run();
}
```

### Graceful Shutdown

```typescript
// Handle shutdown signals
process.on("SIGTERM", async () => {
  console.log("SIGTERM received, shutting down gracefully");

  // Worker completes current tasks before shutting down
  await worker.shutdown();

  process.exit(0);
});
```

---

## 8. Multiple Task Queues for Different Purposes

```typescript
// Different workers for different workloads

// High-priority quick tasks
const quickWorker = await Worker.create({
  taskQueue: "quick-tasks",
  maxConcurrentActivityTaskExecutions: 20,
});

// AI-heavy slow tasks
const aiWorker = await Worker.create({
  taskQueue: "ai-tasks",
  maxConcurrentActivityTaskExecutions: 5, // Limit concurrent AI calls
});

// Batch processing
const batchWorker = await Worker.create({
  taskQueue: "batch-tasks",
  maxConcurrentActivityTaskExecutions: 10,
});
```

### Triggering Different Queues

```typescript
// Quick report (basic stats only)
await client.start(QuickReportWorkflow, {
  taskQueue: 'quick-tasks', // Fast workers
  args: [{ surveyId }]
});

// Full report with AI
await client.start(FullReportWorkflow, {
  taskQueue: 'ai-tasks', // Specialized AI workers
  args: [{ surveyId }]
});

// Batch report generation
await client.start(BatchReportWorkflow, {
  taskQueue: 'batch-tasks', // Dedicated batch workers
  args: [{ surveyIds: [...] }]
});
```

---

## 9. Monitoring & Observability

### Temporal Web UI

Access at `http://localhost:8080`

Features:

- See all running workflows
- View workflow history
- Debug failed workflows
- Replay workflows
- Monitor worker health

### Metrics Collection

```typescript
import { Counter, Histogram } from "prom-client";

const workflowStarted = new Counter({
  name: "temporal_workflows_started_total",
  help: "Total workflows started",
  labelNames: ["workflow_type", "task_queue"],
});

const workflowDuration = new Histogram({
  name: "temporal_workflow_duration_seconds",
  help: "Workflow execution duration",
  labelNames: ["workflow_type", "status"],
});

// In your activities
export async function generateReport(data: any) {
  const start = Date.now();

  try {
    const result = await doWork(data);

    workflowDuration.observe({ workflow_type: "report_generation", status: "success" }, (Date.now() - start) / 1000);

    return result;
  } catch (error) {
    workflowDuration.observe({ workflow_type: "report_generation", status: "failure" }, (Date.now() - start) / 1000);

    throw error;
  }
}
```

---

## 11. Complete Report Storage & Retrieval Flow

### Where Report Data Lives

```
┌─────────────────────────────────────────────────────────────┐
│                  WORKFLOW EXECUTION                          │
│                                                              │
│  Activities save data progressively:                         │
│  ├─ Basic Statistics → report_components table              │
│  ├─ Sentiment Analysis → report_components table            │
│  ├─ Theme Analysis → report_components table                │
│  ├─ Visualizations → report_components table                │
│  └─ Final Compilation → generated_reports table             │
│                                                              │
│  Workflow returns: { reportId: "uuid-123" }                 │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              WEBHOOK NOTIFICATION (Lightweight)              │
│                                                              │
│  POST /api/webhooks/report-completed                        │
│  Body: { reportId: "uuid-123", status: "completed" }       │
│  Size: < 1KB                                                 │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│            REPORTING SERVICE PROCESSES WEBHOOK               │
│                                                              │
│  ├─ Update report status in database                        │
│  ├─ Send WebSocket to frontend                              │
│  └─ Send email with report link                             │
└─────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────┐
│              USER VIEWS REPORT IN BROWSER                    │
│                                                              │
│  GET /reports/{reportId}                                     │
│  ├─ Frontend requests report                                 │
│  ├─ Service fetches from database                           │
│  ├─ Builds HTML dynamically                                 │
│  └─ Returns interactive web page                            │
└─────────────────────────────────────────────────────────────┘
```

### Complete User Flow for Viewing Reports

```
1. Workflow completes with reportId: "abc-123"
2. Webhook notifies service: { reportId: "abc-123", status: "completed" }
3. Service sends email to user: "View report: /reports/abc-123"
4. User clicks link
5. Frontend: GET /api/v1/reports/abc-123
6. Backend: Fetch from database, return JSON with all component data
7. Frontend: Render interactive HTML with charts, stats, insights
8. User clicks "Download PDF"
9. Backend: Generate PDF on-demand from database data, return as file
```

### Database Structure Recap

```sql
-- Main report record
CREATE TABLE generated_reports (
    id UUID PRIMARY KEY,
    survey_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    status VARCHAR(50) DEFAULT 'generating',

    -- Metadata only, NOT the full report data
    config JSONB NOT NULL,
    required_roles TEXT[],

    -- Timestamps
    generation_started_at TIMESTAMP,
    generation_completed_at TIMESTAMP,

    INDEX idx_survey_id (survey_id)
);

-- All report components (THIS is where data lives)
CREATE TABLE report_components (
    id UUID PRIMARY KEY,
    report_id UUID REFERENCES generated_reports(id),
    component_type VARCHAR(100) NOT NULL, -- 'basic_stats', 'sentiment', 'charts'
    component_name VARCHAR(255) NOT NULL,

    -- THIS IS WHERE THE ACTUAL REPORT DATA IS STORED
    data JSONB NOT NULL, -- Can be large (100KB - 5MB per component)
    metadata JSONB,

    status VARCHAR(50) DEFAULT 'completed',
    ai_confidence_score DECIMAL(3,2),

    created_at TIMESTAMP,

    INDEX idx_report_id (report_id),
    INDEX idx_component_type (component_type)
);
```

### Activity Implementation - Saving Report Data

```typescript
// reporting-service/src/worker/activities/report-storage.activities.ts

export async function saveReportComponent(reportId: string, componentType: string, componentData: any): Promise<void> {
  const repository = new ReportRepository();

  // Save to database - this is where report data actually lives
  await repository.saveComponent({
    reportId,
    componentType,
    componentName: componentData.name,
    data: componentData, // Full visualization data, statistics, etc.
    metadata: {
      generatedAt: new Date(),
      dataSize: JSON.stringify(componentData).length,
      version: "1.0",
    },
    status: "completed",
    aiConfidenceScore: componentData.confidenceScore,
  });

  console.log(`Saved component ${componentType} for report ${reportId}`);
}

export async function compileReport(reportId: string, components: any): Promise<GeneratedReport> {
  const repository = new ReportRepository();

  // Update main report record
  await repository.update(reportId, {
    status: "completed",
    generationCompletedAt: new Date(),
    config: {
      componentTypes: Object.keys(components),
      totalComponents: Object.keys(components).length,
    },
  });

  // Return only metadata, not full data
  return {
    id: reportId,
    surveyId: await repository.getSurveyId(reportId),
    status: "completed",
    componentCount: Object.keys(components).length,
    generatedAt: new Date(),
  };
}
```

### Workflow Returns Lightweight Response

```typescript
// reporting-service/src/worker/workflows/report-generation.workflow.ts

export async function ReportGenerationWorkflow(request: ReportGenerationRequest): Promise<GeneratedReport> {
  // ... all processing steps ...

  // Step 1: Extract and save basic stats
  const basicStats = await generateBasicStatistics(rawData);
  await saveReportComponent(reportId, "basic_stats", basicStats);

  // Step 2: Save AI analysis results
  const sentiment = await generateSentimentAnalysis(rawData);
  await saveReportComponent(reportId, "sentiment", sentiment);

  // Step 3: Save visualizations
  const charts = await generateVisualizationData(rawData);
  await saveReportComponent(reportId, "charts", charts);

  // Step 4: Mark report as complete
  const finalReport = await compileReport(reportId, {
    basicStats,
    sentiment,
    charts,
  });

  // Step 5: Send webhook notification
  await notifyCompletion(request.userId, finalReport);

  // Return lightweight metadata only
  return finalReport; // { id, surveyId, status, componentCount, generatedAt }
}
```

### Frontend Report Viewing Flow

```typescript
// Frontend: User clicks on report notification
async function viewReport(reportId: string) {
  // Step 1: Request report data from API
  const response = await fetch(`/api/v1/reports/${reportId}`, {
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  const reportData = await response.json();

  // Step 2: Render HTML with report data
  renderReportPage(reportData);
}

// Reporting Service API Endpoint
app.get("/api/v1/reports/:reportId", async (req, res) => {
  const { reportId } = req.params;
  const userId = req.user.id;

  try {
    // Check permissions
    const hasAccess = await checkReportAccess(reportId, userId);
    if (!hasAccess) {
      return res.status(403).json({ error: "Access denied" });
    }

    // Fetch report from database
    const report = await reportRepository.findById(reportId);

    if (!report) {
      return res.status(404).json({ error: "Report not found" });
    }

    if (report.status !== "completed") {
      return res.status(202).json({
        status: report.status,
        message: "Report is still generating",
      });
    }

    // Fetch all components
    const components = await reportRepository.getComponents(reportId);

    // Build response with full report data
    const fullReport = {
      reportId: report.id,
      surveyId: report.surveyId,
      organizationId: report.organizationId,
      metadata: {
        surveyTitle: await getSurveyTitle(report.surveyId),
        generatedAt: report.generationCompletedAt,
        componentCount: components.length,
      },
      components: components.map((c) => ({
        type: c.componentType,
        name: c.componentName,
        data: c.data, // Full data (charts, stats, etc.)
        metadata: c.metadata,
        confidenceScore: c.aiConfidenceScore,
      })),
      permissions: {
        canDownloadPdf: true,
        canExport: true,
        canShare: hasSharePermission(userId, report),
      },
    };

    res.json(fullReport);
  } catch (error) {
    console.error("Error fetching report:", error);
    res.status(500).json({ error: "Failed to fetch report" });
  }
});
```

### HTML Rendering on Frontend

```typescript
// Frontend React component (example)
function ReportViewer({ reportId }: { reportId: string }) {
  const [report, setReport] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    fetchReport(reportId).then((data) => {
      setReport(data);
      setLoading(false);
    });
  }, [reportId]);

  if (loading) return <Loading />;

  return (
    <div className="report-container">
      <ReportHeader title={report.metadata.surveyTitle} generatedAt={report.metadata.generatedAt} />

      {/* Render each component dynamically */}
      {report.components.map((component) => {
        switch (component.type) {
          case "basic_stats":
            return <StatisticsSection data={component.data} />;

          case "sentiment":
            return <SentimentAnalysisSection data={component.data} />;

          case "charts":
            return <ChartsSection data={component.data} />;

          case "themes":
            return <ThemeAnalysisSection data={component.data} />;

          default:
            return null;
        }
      })}

      <ReportActions
        onDownloadPdf={() => downloadPdf(reportId)}
        onExport={() => exportData(reportId)}
        onShare={() => shareReport(reportId)}
      />
    </div>
  );
}
```

### PDF Download Flow

```typescript
// Frontend: User clicks "Download PDF"
async function downloadPdf(reportId: string) {
  // Request PDF generation (on-demand)
  const response = await fetch(`/api/v1/reports/${reportId}/pdf`, {
    method: "GET",
    headers: {
      Authorization: `Bearer ${token}`,
    },
  });

  // Get PDF as blob
  const blob = await response.blob();

  // Download file
  const url = window.URL.createObjectURL(blob);
  const a = document.createElement("a");
  a.href = url;
  a.download = `report-${reportId}.pdf`;
  a.click();
}

// Backend: Generate PDF on-demand
app.get("/api/v1/reports/:reportId/pdf", async (req, res) => {
  const { reportId } = req.params;

  try {
    // Fetch report data from database
    const report = await reportRepository.findById(reportId);
    const components = await reportRepository.getComponents(reportId);

    // Generate HTML
    const html = buildReportHTML(report, components);

    // Convert to PDF using Puppeteer
    const browser = await puppeteer.launch();
    const page = await browser.newPage();
    await page.setContent(html);

    const pdf = await page.pdf({
      format: "A4",
      printBackground: true,
      margin: {
        top: "1cm",
        right: "1cm",
        bottom: "1cm",
        left: "1cm",
      },
    });

    await browser.close();

    // Log access
    await auditService.log({
      userId: req.user.id,
      reportId,
      action: "download_pdf",
    });

    // Return PDF
    res.setHeader("Content-Type", "application/pdf");
    res.setHeader("Content-Disposition", `attachment; filename="report-${reportId}.pdf"`);
    res.send(pdf);
  } catch (error) {
    console.error("PDF generation failed:", error);
    res.status(500).json({ error: "Failed to generate PDF" });
  }
});

function buildReportHTML(report: Report, components: Component[]): string {
  return `
    <!DOCTYPE html>
    <html>
    <head>
      <meta charset="UTF-8">
      <title>${report.metadata.surveyTitle} - Report</title>
      <style>
        body { font-family: Arial, sans-serif; margin: 0; padding: 20px; }
        .header { border-bottom: 2px solid #333; padding-bottom: 20px; }
        .section { margin: 30px 0; page-break-inside: avoid; }
        .chart { max-width: 100%; height: auto; }
        table { width: 100%; border-collapse: collapse; }
        td, th { border: 1px solid #ddd; padding: 8px; text-align: left; }
      </style>
    </head>
    <body>
      <div class="header">
        <h1>${report.metadata.surveyTitle}</h1>
        <p>Generated: ${new Date(report.generationCompletedAt).toLocaleDateString()}</p>
      </div>
      
      ${components.map((component) => renderComponent(component)).join("\n")}
      
      <div class="footer">
        <p>Report ID: ${report.id}</p>
      </div>
    </body>
    </html>
  `;
}
```

### Data Size Considerations

**Component Data Sizes:**

- Basic Statistics: ~10-50 KB
- Sentiment Analysis: ~20-100 KB
- Theme Analysis: ~50-200 KB
- Chart Data: ~50-500 KB per chart
- Total per report: ~500 KB - 5 MB

**Why Webhooks Work:**

```
Webhook payload: ~500 bytes (just metadata)
{
  "reportId": "uuid-123",
  "surveyId": "uuid-456",
  "status": "completed",
  "userId": "uuid-789",
  "completedAt": "2024-01-15T10:30:00Z"
}

Actual report data: 2 MB (stays in database)
- User fetches it when they view the report
- PDF generated on-demand from database data
```

**Benefits of this approach:**

1. **Fast Webhooks:** No timeout issues with large payloads
2. **Flexible Rendering:** Can update HTML templates without regenerating reports
3. **Efficient Storage:** Data stored once, rendered multiple times
4. **Progressive Loading:** Can load report sections on-demand
5. **Security:** Access control checked when viewing, not at webhook time

### Report Access by ID - Complete Example

```typescript
// User Journey:

// 1. User receives notification (via webhook-triggered email)
Email: "Your survey report is ready! View it here:
       https://yourapp.com/reports/abc-123-def-456"

// 2. User clicks link
// Frontend route: /reports/:reportId

// 3. Frontend component loads
function ReportPage() {
  const { reportId } = useParams();
  const { data, loading, error } = useReport(reportId);

  if (loading) return <LoadingSpinner />;
  if (error) return <ErrorMessage error={error} />;

  return <ReportViewer report={data} />;
}

// 4. useReport hook fetches from API
function useReport(reportId: string) {
  return useQuery(['report', reportId], async () => {
    const response = await fetch(`/api/v1/reports/${reportId}`);
    if (!response.ok) throw new Error('Failed to fetch report');
    return response.json();
  });
}

// 5. Backend returns full report data from database
// 6. Frontend renders interactive HTML
// 7. User can download PDF (generated on-demand)
```

### Progressive Component Loading (Optional Optimization)

```typescript
// If reports are very large, load components progressively

// Frontend
function ReportViewer({ reportId }: { reportId: string }) {
  const [metadata, setMetadata] = useState(null);
  const [components, setComponents] = useState({});

  useEffect(() => {
    // Load metadata first (fast)
    fetch(`/api/v1/reports/${reportId}/metadata`)
      .then((res) => res.json())
      .then(setMetadata);

    // Load components progressively
    metadata?.componentTypes.forEach((type) => {
      fetch(`/api/v1/reports/${reportId}/components/${type}`)
        .then((res) => res.json())
        .then((data) => {
          setComponents((prev) => ({ ...prev, [type]: data }));
        });
    });
  }, [reportId, metadata]);

  return (
    <div>
      {metadata && <ReportHeader {...metadata} />}
      {components.basic_stats && <StatisticsSection data={components.basic_stats} />}
      {components.sentiment && <SentimentSection data={components.sentiment} />}
      {!components.charts && <LoadingPlaceholder />}
      {components.charts && <ChartsSection data={components.charts} />}
    </div>
  );
}

// Backend API for progressive loading
app.get("/api/v1/reports/:reportId/components/:type", async (req, res) => {
  const { reportId, type } = req.params;

  const component = await reportRepository.getComponent(reportId, type);

  if (!component) {
    return res.status(404).json({ error: "Component not found" });
  }

  res.json(component.data);
});
```

### Resource Planning

```yaml
# For 100 concurrent users, 5K surveys/month

Temporal Server:
  - 1 instance: 4 vCPU, 8GB RAM
  - Cost: ~$200/month

Reporting Workers:
  - 3-5 containers: 2 vCPU, 4GB RAM each
  - Auto-scale to 10 during peak
  - Cost: ~$300-500/month

Organization Workers:
  - 2 containers: 1 vCPU, 2GB RAM each
  - Cost: ~$100/month

Total Temporal Infrastructure: ~$600-800/month
```

### Optimization Strategies

1. **Worker Pooling:** Reuse workers for multiple workflows
2. **Activity Caching:** Cache AI results for similar surveys
3. **Batch Processing:** Group similar reports together
4. **Smart Scheduling:** Run heavy AI tasks during off-peak hours
5. **Progressive Results:** Return basic stats immediately, AI later

---

This comprehensive guide shows how Temporal.io works in practice. The key takeaway: **Workers are just containers that poll for work** - Temporal Server handles all orchestration, state management, and distribution. You can scale workers independently without worrying about coordination.
