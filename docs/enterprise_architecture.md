# Right-Sized Enterprise Architecture - API Microservices + Workflow Engine

## Architecture Overview Based on Your Requirements

**Scale Profile:**
- 100 concurrent users
- 5,000 surveys/month (~167 per day)
- Organizations up to 100 employees
- Real-time org schema analysis required
- 30-second survey generation acceptable
- Reports can be generated asynchronously

**Architecture Decision:**
- **API-based microservices** for clean separation and moderate scalability
- **Workflow engine** (Temporal.io) for complex multi-step processes
- **Separate databases** for true service independence
- **Background job processing** for non-critical operations

## Service Architecture

### Core Services with Independent Databases

```
┌─────────────────┐    REST API    ┌─────────────────┐
│ Organization    │◄──────────────►│ Library         │
│ Service         │                │ Service         │
│ (PostgreSQL)    │                │ (PostgreSQL)    │
└─────────────────┘                └─────────────────┘
         ▲                                   ▲
         │ REST API                          │ REST API
         ▼                                   ▼
┌─────────────────┐                ┌─────────────────┐
│ Delivery        │◄──────────────►│ Reporting       │
│ Service         │    REST API    │ Service         │
│ (PostgreSQL)    │                │ (PostgreSQL)    │
└─────────────────┘                └─────────────────┘
         ▲                                   ▲
         │                                   │
         └───────────┬───────────────────────┘
                     ▼
            ┌─────────────────┐
            │ Temporal.io     │
            │ Workflow Engine │
            └─────────────────┘
```

### Database Independence Strategy

Each service owns its data with cross-service communication via APIs:

```sql
-- Organization Service Database
CREATE DATABASE org_service;
\c org_service;
CREATE TABLE companies (...);
CREATE TABLE employees (...);
CREATE TABLE org_schemas (...);

-- Library Service Database  
CREATE DATABASE library_service;
\c library_service;
CREATE TABLE questions (...);
CREATE TABLE surveys (...);
CREATE TABLE themes (...);

-- Delivery Service Database
CREATE DATABASE delivery_service;
\c delivery_service;
CREATE TABLE survey_assignments (...);
CREATE TABLE responses (...);
CREATE TABLE email_logs (...);

-- Reporting Service Database
CREATE DATABASE reporting_service;
\c reporting_service;
CREATE TABLE analytics (...);
CREATE TABLE insights (...);
CREATE TABLE generated_reports (...);
```

## Workflow Engine Integration

### Temporal.io for Complex Processes

**Survey Creation Workflow:**
```typescript
@WorkflowMethod
export class SurveyCreationWorkflow {
  
  async execute(request: SurveyCreationRequest): Promise<Survey> {
    // Step 1: Validate organization exists (real-time)
    const organization = await this.activities.validateOrganization(
      request.organizationId
    );
    
    // Step 2: Generate survey with AI (30 seconds acceptable)
    const generatedSurvey = await this.activities.generateSurveyWithAI(
      request.requirements
    );
    
    // Step 3: Save to library service (real-time)
    const savedSurvey = await this.activities.saveSurveyToLibrary(
      generatedSurvey
    );
    
    // Step 4: Create assignments if requested (real-time)
    if (request.assignImmediately) {
      await this.activities.createSurveyAssignments(
        savedSurvey.id, 
        organization.employees
      );
    }
    
    return savedSurvey;
  }
}
```

**Org Schema Processing Workflow:**
```typescript
@WorkflowMethod
export class OrgSchemaProcessingWorkflow {
  
  async execute(uploadRequest: OrgSchemaUpload): Promise<OrgSchema> {
    // Step 1: Parse uploaded file (real-time - user waits)
    const parsedData = await this.activities.parseUploadedFile(
      uploadRequest.fileId
    );
    
    // Step 2: AI analysis (real-time - required by user)
    // This must complete within reasonable time since real-time is mandatory
    const aiAnalysis = await this.activities.analyzeOrgDataWithAI(
      parsedData,
      { timeout: '2 minutes' } // Fail fast if AI takes too long
    );
    
    // Step 3: Build interactive schema (real-time)
    const initialSchema = await this.activities.buildInteractiveSchema(
      aiAnalysis
    );
    
    // Step 4: Present to user for refinement (human task)
    const userApproval = await this.activities.waitForUserApproval(
      initialSchema,
      { timeout: '24 hours' }
    );
    
    // Step 5: Finalize schema (real-time)
    const finalSchema = await this.activities.finalizeOrgSchema(
      userApproval.approvedSchema
    );
    
    return finalSchema;
  }
}
```

**Report Generation Workflow:**
```typescript
@WorkflowMethod  
export class ReportGenerationWorkflow {
  
  async execute(surveyId: string): Promise<GeneratedReport> {
    // Step 1: Validate survey completion
    const survey = await this.activities.validateSurveyCompleted(surveyId);
    
    // Step 2: Extract response data
    const responses = await this.activities.extractSurveyResponses(surveyId);
    
    // Step 3: Run AI analysis (background - can take time)
    const insights = await this.activities.generateAIInsights(
      responses,
      { timeout: '10 minutes' }
    );
    
    // Step 4: Create visualizations (parallel processing)
    const [charts, summaries, recommendations] = await Promise.all([
      this.activities.generateCharts(insights),
      this.activities.generateSummaries(insights), 
      this.activities.generateRecommendations(insights)
    ]);
    
    // Step 5: Compile final report
    const report = await this.activities.compileReport({
      survey,
      insights,
      charts,
      summaries,
      recommendations
    });
    
    // Step 6: Notify completion
    await this.activities.notifyReportCompletion(survey.createdBy, report);
    
    return report;
  }
}
```

## Service Implementation Details

### Organization Service

**Responsibilities:**
- Company and employee management
- Organizational schema CRUD operations
- Employee hierarchy management
- Integration with AI for schema processing

**API Design:**
```typescript
// REST API endpoints
GET    /api/v1/organizations/{id}
POST   /api/v1/organizations
PUT    /api/v1/organizations/{id}
DELETE /api/v1/organizations/{id}

GET    /api/v1/organizations/{id}/employees
POST   /api/v1/organizations/{id}/employees
PUT    /api/v1/organizations/{id}/employees/{employeeId}

POST   /api/v1/org-schemas/upload        // Triggers Temporal workflow
GET    /api/v1/org-schemas/{id}/status   // Check processing status
```

**Database Schema:**
```sql
CREATE TABLE organizations (
  id UUID PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  industry VARCHAR(100),
  size_category VARCHAR(50),
  created_by UUID,
  created_at TIMESTAMP,
  updated_at TIMESTAMP
);

CREATE TABLE employees (
  id UUID PRIMARY KEY,
  organization_id UUID REFERENCES organizations(id),
  name VARCHAR(255) NOT NULL,
  email VARCHAR(255) UNIQUE NOT NULL,
  role VARCHAR(255),
  department VARCHAR(255),
  manager_id UUID REFERENCES employees(id),
  created_at TIMESTAMP
);

CREATE TABLE org_schemas (
  id UUID PRIMARY KEY,
  organization_id UUID REFERENCES organizations(id),
  schema_data JSONB NOT NULL,
  status VARCHAR(50) DEFAULT 'draft',
  ai_confidence_score DECIMAL(3,2),
  created_at TIMESTAMP
);
```

### Library Service

**Responsibilities:**
- Question and theme management
- Survey template creation
- AI-powered survey generation
- Survey versioning and approval workflow

**API Design:**
```typescript
GET    /api/v1/surveys
POST   /api/v1/surveys/generate          // Triggers Temporal workflow  
GET    /api/v1/surveys/{id}
PUT    /api/v1/surveys/{id}
DELETE /api/v1/surveys/{id}

GET    /api/v1/questions
POST   /api/v1/questions
GET    /api/v1/themes
```

### Delivery Service  

**Responsibilities:**
- Survey assignment management
- Email delivery and tracking
- Response collection and storage
- Survey completion monitoring

**API Design:**
```typescript
POST   /api/v1/assignments                // Create survey assignments
GET    /api/v1/assignments/{surveyId}     // Get assignment status
POST   /api/v1/responses                  // Submit survey response  
GET    /api/v1/responses/{surveyId}       // Get all responses

POST   /api/v1/surveys/{id}/send          // Send survey emails
GET    /api/v1/surveys/{id}/progress      // Get completion progress
```

### Reporting Service

**Responsibilities:**  
- Analytics calculation and storage
- Report generation using AI
- Visualization creation
- Insight generation and recommendations

**API Design:**
```typescript
POST   /api/v1/reports/generate/{surveyId}  // Triggers Temporal workflow
GET    /api/v1/reports/{surveyId}           // Get report status/results
GET    /api/v1/reports/{surveyId}/insights  // Get AI insights
GET    /api/v1/reports/{surveyId}/charts    // Get visualizations
```

## Infrastructure Requirements

### Right-Sized Infrastructure

**Container Platform:**
```yaml
Kubernetes Cluster or Docker Compose:
  Organization Service: 2 replicas, 1 CPU, 2GB RAM each
  Library Service: 2 replicas, 1 CPU, 2GB RAM each  
  Delivery Service: 3 replicas, 1 CPU, 2GB RAM each (email heavy)
  Reporting Service: 2 replicas, 2 CPU, 4GB RAM each (AI processing)
  
  Total: 12-16 GB RAM, 10-12 CPU cores
```

**Database Setup:**
```yaml  
PostgreSQL Instances:
  Primary: 4 vCPU, 16GB RAM, 500GB SSD
  Read Replicas: 1 per service (2 vCPU, 8GB RAM each)
  
  Or managed databases:
  - 4 separate Cloud SQL instances (smaller but independent)
```

**Temporal.io Setup:**
```yaml
Temporal Cluster:
  Temporal Server: 2 vCPU, 4GB RAM
  Database: PostgreSQL (can share with other services)
  Workers: 3 instances, 1 vCPU, 2GB RAM each
```

**Additional Infrastructure:**
```yaml
Redis: 1GB instance for caching and session storage
Load Balancer: Cloud Load Balancer or nginx
Monitoring: Prometheus + Grafana (lightweight setup)
```

## Cost Analysis

### Monthly Infrastructure Costs (Google Cloud)

```yaml
Kubernetes/Compute:
  GKE Cluster (3 nodes, 4 vCPU each): $300-500
  
Databases:
  PostgreSQL instances (4 x 2vCPU, 8GB): $400-800
  
Temporal Infrastructure:
  Additional compute and storage: $100-200
  
Supporting Services:
  Redis, Load Balancer, Storage: $100-200
  
Total Infrastructure: $900-1,700/month

Development Team (3-4 people): $25,000-40,000/month
LLM APIs (OpenAI): $500-2,000/month

Total Monthly Cost: $26,400-43,700/month
```

## Implementation Approach

### Phase 1: Service Foundation (Months 1-2)

**Deliverables:**
- Basic CRUD APIs for all services
- Independent database setup
- Container deployment configuration
- Basic authentication and authorization

**Architecture:**
- Start with simple REST API communication
- No Temporal.io yet (use simple background jobs)
- Basic error handling and logging

### Phase 2: Workflow Integration (Months 3-4)

**Deliverables:**
- Temporal.io integration for complex workflows
- AI processing integration for org schema analysis
- Real-time org schema processing
- Survey generation workflows

**Key Features:**
- Org schema upload and AI processing workflow
- Survey generation with 30-second timeout
- User approval workflows for schema refinement

### Phase 3: Advanced Features (Months 5-6)

**Deliverables:**
- Complete reporting workflow with AI insights
- Advanced monitoring and alerting
- Performance optimization
- Production deployment and security hardening

## Event Sourcing Integration

### Adding Event Sourcing to API-based Microservices

**Event Sourcing Layer:**
```sql
-- Shared Event Store (can be in any service database)
CREATE TABLE event_store (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  stream_id VARCHAR(255) NOT NULL,        -- Entity identifier (surveyId, orgId, etc.)
  stream_type VARCHAR(100) NOT NULL,      -- 'survey', 'organization', 'user'
  event_type VARCHAR(100) NOT NULL,       -- 'survey.created', 'org.updated'
  event_version INTEGER NOT NULL,         -- For optimistic concurrency
  event_data JSONB NOT NULL,              -- The actual event payload
  metadata JSONB,                         -- Correlation IDs, user info, etc.
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(stream_id, event_version)
);

CREATE INDEX idx_stream_id ON event_store(stream_id);
CREATE INDEX idx_event_type ON event_store(event_type);
CREATE INDEX idx_created_at ON event_store(created_at);
```

**Event-Driven API Pattern:**
```typescript
class SurveyService {
  async createSurvey(surveyData: CreateSurveyRequest): Promise<Survey> {
    // 1. Create the survey (business logic)
    const survey = await this.buildSurvey(surveyData);
    
    // 2. Save to database (current state)
    const savedSurvey = await this.repository.save(survey);
    
    // 3. Append event for audit trail
    await this.eventStore.appendEvent({
      streamId: savedSurvey.id,
      streamType: 'survey',
      eventType: 'survey.created',
      eventData: {
        surveyId: savedSurvey.id,
        title: savedSurvey.title,
        organizationId: savedSurvey.organizationId,
        questionCount: savedSurvey.questions.length
      },
      metadata: {
        userId: surveyData.createdBy,
        correlationId: generateCorrelationId(),
        timestamp: new Date()
      }
    });
    
    // 4. Trigger workflows or notify other services
    await this.temporalClient.startWorkflow(SurveyCreationWorkflow, {
      surveyId: savedSurvey.id
    });
    
    return savedSurvey;
  }
  
  async updateSurvey(surveyId: string, updates: UpdateSurveyRequest): Promise<Survey> {
    const currentSurvey = await this.repository.findById(surveyId);
    const updatedSurvey = await this.repository.update(surveyId, updates);
    
    // Log what changed
    await this.eventStore.appendEvent({
      streamId: surveyId,
      streamType: 'survey',
      eventType: 'survey.updated',
      eventData: {
        surveyId,
        changes: this.calculateDiff(currentSurvey, updatedSurvey),
        previousVersion: currentSurvey.version,
        newVersion: updatedSurvey.version
      },
      metadata: {
        userId: updates.updatedBy,
        correlationId: updates.correlationId
      }
    });
    
    return updatedSurvey;
  }
}
```

**Event Store Service:**
```typescript
class EventStoreService {
  async appendEvent(event: DomainEvent): Promise<void> {
    // Get current version for optimistic concurrency
    const currentVersion = await this.getCurrentVersion(event.streamId);
    
    await this.db.query(`
      INSERT INTO event_store 
      (stream_id, stream_type, event_type, event_version, event_data, metadata)
      VALUES ($1, $2, $3, $4, $5, $6)
    `, [
      event.streamId,
      event.streamType, 
      event.eventType,
      currentVersion + 1,
      event.eventData,
      event.metadata
    ]);
  }
  
  async getEntityHistory(streamId: string): Promise<DomainEvent[]> {
    const result = await this.db.query(`
      SELECT * FROM event_store 
      WHERE stream_id = $1 
      ORDER BY event_version ASC
    `, [streamId]);
    
    return result.rows.map(this.mapToEvent);
  }
  
  async replayEntityState(streamId: string): Promise<any> {
    const events = await this.getEntityHistory(streamId);
    return events.reduce((state, event) => {
      return this.applyEvent(state, event);
    }, {});
  }
}
```

### Event Sourcing Benefits for Your Platform

**Audit and Compliance:**
- Complete history of all survey changes
- Who changed what and when
- Ability to replay any entity's full history
- Regulatory compliance for enterprise clients

**Business Intelligence:**
- Analyze user behavior patterns
- Track survey creation and modification patterns
- Understand organizational change patterns
- Generate insights from historical data

**Error Recovery:**
- Rebuild corrupted data from events
- Debug complex issues by replaying events
- Undo operations by applying compensating events

**Example Event Streams:**

```typescript
// Survey lifecycle events
const surveyEvents = [
  { eventType: 'survey.created', data: { title: 'Team Health Check', orgId: 'org-123' }},
  { eventType: 'survey.questions_added', data: { questionIds: ['q1', 'q2', 'q3'] }},
  { eventType: 'survey.published', data: { publishedAt: '2024-01-15' }},
  { eventType: 'survey.assigned', data: { employeeIds: ['emp1', 'emp2'], assignedBy: 'user-123' }},
  { eventType: 'survey.completed', data: { completedAt: '2024-01-20', responseCount: 25 }}
];

// Organization schema events  
const orgEvents = [
  { eventType: 'org.schema_created', data: { name: 'Acme Corp', employeeCount: 50 }},
  { eventType: 'org.employee_added', data: { employeeId: 'emp-456', name: 'John Doe' }},
  { eventType: 'org.hierarchy_updated', data: { managerId: 'emp-123', reportIds: ['emp-456'] }},
  { eventType: 'org.schema_approved', data: { approvedBy: 'user-789', approvedAt: '2024-01-15' }}
];
```

### Implementation Strategy

**Option 1: Event Store per Service (Distributed)**
```
Organization Service → Organization Event Store
Library Service → Library Event Store  
Delivery Service → Delivery Event Store
Reporting Service → Reporting Event Store
```

**Option 2: Centralized Event Store (Recommended)**
```
All Services → Shared Event Store
- Single source of truth for all events
- Cross-service event correlation
- Easier event replay and analysis
```

## Benefits of This Approach

### Technical Benefits
- **Service Independence:** Each service can be developed, tested, and deployed independently
- **Workflow Reliability:** Temporal.io handles complex multi-step processes with automatic retries
- **Complete Audit Trail:** Event sourcing provides full history of all changes
- **Scalability:** Can scale services independently based on load patterns
- **Data Ownership:** Clear data boundaries prevent service coupling
- **Debugging Capability:** Can replay events to understand system behavior

### Business Benefits  
- **Meets Your Requirements:** Real-time org analysis, acceptable survey generation time
- **Right-Sized:** Infrastructure costs match your scale (100 users, 5K surveys/month)
- **Professional:** Workflow engine provides enterprise-grade reliability
- **Compliance Ready:** Complete audit trail for enterprise clients
- **Growth Ready:** Architecture can handle 10x growth without major changes
- **Business Intelligence:** Rich event data enables advanced analytics

### Development Benefits
- **Team Productivity:** Clear service boundaries allow parallel development
- **Testing:** Independent services are easier to test in isolation
- **Debugging:** Workflow engine + event history provides excellent debugging capabilities
- **Maintenance:** Well-defined APIs and event contracts make maintenance manageable
- **Data Recovery:** Events provide natural backup and recovery mechanisms

## Potential Challenges

### Technical Risks
- **Network Latency:** Cross-service API calls add latency compared to shared database
- **Data Consistency:** Need careful design to handle eventual consistency
- **Complexity:** Temporal.io has learning curve for the team
- **Deployment Coordination:** Multiple services require coordinated deployments

### Mitigation Strategies
- **API Design:** Minimize chatty API calls through efficient endpoint design
- **Caching:** Use Redis caching for frequently accessed data
- **Monitoring:** Comprehensive monitoring for cross-service communication
- **Documentation:** Thorough API documentation and service contracts

## Recommendation

This architecture balances your requirements perfectly:

- **Meets Scale Needs:** Handles 100 concurrent users and 5K surveys/month comfortably
- **Provides Enterprise Features:** Workflow engine gives you reliability and observability
- **Reasonable Cost:** ~$1,700/month infrastructure vs $12,000+ for full enterprise architecture
- **Growth Path:** Can evolve to event-driven architecture when you reach 500+ concurrent users

The API-based microservices approach with Temporal.io workflows gives you the benefits of service independence while maintaining reasonable complexity for your team size and scale requirements.