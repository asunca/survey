# Reporting Module - Architectural Design Guide

## 1. Module Overview

### Core Responsibilities
- Generate comprehensive reports from completed survey data
- Provide AI-powered insights and analytics
- Create interactive web-based visualizations
- Support role-based report access control
- Handle PDF generation on-demand
- Manage data retention and archival processes

### Key Design Principles
- **Asynchronous Processing:** All report generation happens in background
- **Real-time Basic Stats:** Response counts available immediately
- **Role-Based Access:** Pre-generated reports with visibility controls
- **No Static Storage:** Generate PDFs on-demand, store configurations only
- **AI Feature Progression:** Start simple, evolve to advanced analytics

---

## 2. System Architecture

### Service Integration Pattern
```
┌─────────────────┐    REST API     ┌─────────────────┐
│ Organization    │◄──────────────► │ Reporting       │
│ Service         │                 │ Service         │
│ (Org Schema)    │                 │ (Analytics)     │
└─────────────────┘                 └─────────────────┘
         ▲                                   ▲
         │ REST API                          │ REST API
         ▼                                   ▼
┌─────────────────┐    REST API     ┌─────────────────┐
│ Delivery        │◄──────────────► │ Library         │
│ Service         │                 │ Service         │
│ (Responses)     │                 │ (Survey Meta)   │
└─────────────────┘                 └─────────────────┘
         ▲                                   ▲
         │                                   │
         └─────────────┬─────────────────────┘
                       ▼
              ┌─────────────────┐
              │ Temporal.io     │
              │ Workflow Engine │
              └─────────────────┘
```

### Temporal.io Workflow Integration
- **Report Generation Workflow:** Orchestrates multi-step analysis process
- **Data Retention Workflow:** Manages automated data archival
- **Batch Processing Workflow:** Handles bulk report generation
- **AI Processing Workflow:** Coordinates external AI service calls

---

## 3. Database Design

### Core Database Schema

```sql
-- Reporting Service Database
CREATE DATABASE reporting_service;

-- Generated Reports Storage
CREATE TABLE generated_reports (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    report_type VARCHAR(100) NOT NULL, -- 'full', 'summary', 'departmental'
    status VARCHAR(50) DEFAULT 'generating', -- 'generating', 'completed', 'failed'
    
    -- Report Configuration
    config JSONB NOT NULL, -- Report parameters and settings
    
    -- Access Control
    required_roles TEXT[] NOT NULL, -- ['company_admin', 'dept_manager']
    visibility_scope JSONB, -- {'departments': ['IT', 'HR'], 'roles': ['manager']}
    
    -- Data References
    raw_data_snapshot_id UUID, -- Reference to archived raw data
    
    -- Timestamps
    generation_started_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    generation_completed_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_survey_id (survey_id),
    INDEX idx_organization_id (organization_id),
    INDEX idx_status (status),
    INDEX idx_report_type (report_type)
);

-- Report Components (for progressive loading)
CREATE TABLE report_components (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES generated_reports(id) ON DELETE CASCADE,
    component_type VARCHAR(100) NOT NULL, -- 'basic_stats', 'sentiment', 'charts'
    component_name VARCHAR(255) NOT NULL, -- 'response_distribution', 'dept_comparison'
    
    -- Component Data
    data JSONB NOT NULL, -- Visualization data, statistics, etc.
    metadata JSONB, -- Chart configs, styling, etc.
    
    -- Processing Status
    status VARCHAR(50) DEFAULT 'pending', -- 'pending', 'processing', 'completed', 'failed'
    processing_duration_ms INTEGER,
    ai_confidence_score DECIMAL(3,2), -- For AI-generated components
    
    -- Timestamps
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    completed_at TIMESTAMP,
    
    INDEX idx_report_id (report_id),
    INDEX idx_component_type (component_type),
    INDEX idx_status (status)
);

-- AI Analysis Results Cache
CREATE TABLE ai_analysis_cache (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id UUID NOT NULL,
    analysis_type VARCHAR(100) NOT NULL, -- 'sentiment', 'themes', 'anomalies'
    analysis_version VARCHAR(20) NOT NULL, -- 'v1.0', 'v1.1' for model versioning
    
    -- Analysis Results
    results JSONB NOT NULL,
    confidence_metrics JSONB, -- Accuracy scores, reliability indicators
    
    -- Processing Info
    model_used VARCHAR(100), -- 'openai-gpt-4', 'custom-sentiment-v1'
    processing_cost_cents INTEGER, -- Track AI costs
    processing_time_ms INTEGER,
    
    -- Data Retention
    expires_at TIMESTAMP, -- Based on retention policy
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE(survey_id, analysis_type, analysis_version),
    INDEX idx_survey_analysis (survey_id, analysis_type),
    INDEX idx_expires_at (expires_at)
);

-- Raw Data Snapshots (for retention management)
CREATE TABLE raw_data_snapshots (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    survey_id UUID NOT NULL,
    organization_id UUID NOT NULL,
    
    -- Data Storage
    storage_location VARCHAR(500), -- S3 path, database partition, etc.
    storage_type VARCHAR(50), -- 'hot', 'warm', 'cold'
    data_size_bytes BIGINT,
    
    -- Retention Management
    tier_level INTEGER DEFAULT 1, -- 1=hot, 2=warm, 3=cold
    next_tier_date TIMESTAMP,
    deletion_date TIMESTAMP,
    
    -- Metadata
    response_count INTEGER,
    question_count INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    last_accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_survey_id (survey_id),
    INDEX idx_storage_type (storage_type),
    INDEX idx_next_tier_date (next_tier_date),
    INDEX idx_deletion_date (deletion_date)
);

-- Report Access Audit Log
CREATE TABLE report_access_log (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    report_id UUID REFERENCES generated_reports(id),
    user_id UUID NOT NULL,
    access_type VARCHAR(50), -- 'view', 'download_pdf', 'export'
    
    -- Access Details
    ip_address INET,
    user_agent TEXT,
    
    -- Timestamps
    accessed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_report_id (report_id),
    INDEX idx_user_id (user_id),
    INDEX idx_accessed_at (accessed_at)
);
```

### Data Retention Schema

```sql
-- Data Retention Policies
CREATE TABLE retention_policies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    organization_id UUID NOT NULL,
    policy_name VARCHAR(255) NOT NULL,
    
    -- Retention Rules
    raw_data_retention_months INTEGER DEFAULT 60, -- 5 years
    analysis_data_retention_months INTEGER DEFAULT 6,
    report_cache_retention_months INTEGER DEFAULT 12,
    
    -- Tiering Rules
    hot_to_warm_months INTEGER DEFAULT 6,
    warm_to_cold_months INTEGER DEFAULT 24,
    
    -- Policy Status
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_organization_id (organization_id)
);
```

---

## 4. API Design

### REST API Endpoints

```typescript
// Report Generation
POST   /api/v1/reports/generate/{surveyId}
GET    /api/v1/reports/{surveyId}
GET    /api/v1/reports/{surveyId}/status
DELETE /api/v1/reports/{surveyId}

// Report Components
GET    /api/v1/reports/{surveyId}/components
GET    /api/v1/reports/{surveyId}/components/{componentType}

// PDF Generation
GET    /api/v1/reports/{surveyId}/pdf
POST   /api/v1/reports/{surveyId}/pdf/generate

// Analytics & Insights
GET    /api/v1/reports/{surveyId}/insights
GET    /api/v1/reports/{surveyId}/analytics
GET    /api/v1/reports/{surveyId}/visualizations

// Data Management
GET    /api/v1/reports/{surveyId}/raw-data
POST   /api/v1/reports/{surveyId}/archive
GET    /api/v1/reports/retention-status

// Access Control
GET    /api/v1/reports/accessible
POST   /api/v1/reports/{surveyId}/access/verify
GET    /api/v1/reports/{surveyId}/permissions
```

### API Response Formats

```typescript
// Report Status Response
interface ReportStatusResponse {
  reportId: string;
  surveyId: string;
  status: 'generating' | 'partial' | 'completed' | 'failed';
  progress: {
    totalComponents: number;
    completedComponents: number;
    currentStep: string;
    estimatedTimeRemaining?: number;
  };
  components: ComponentStatus[];
  generatedAt?: string;
  error?: string;
}

// Component Status
interface ComponentStatus {
  type: string;
  name: string;
  status: 'pending' | 'processing' | 'completed' | 'failed';
  processingDuration?: number;
  confidenceScore?: number;
  error?: string;
}

// Report Data Response
interface ReportDataResponse {
  reportId: string;
  surveyId: string;
  organizationId: string;
  reportType: string;
  metadata: {
    surveyTitle: string;
    responseCount: number;
    completionRate: number;
    generatedAt: string;
  };
  components: ReportComponent[];
  permissions: {
    canDownloadPdf: boolean;
    canExport: boolean;
    canShare: boolean;
  };
}

// Report Component
interface ReportComponent {
  type: string;
  name: string;
  data: any; // Chart data, statistics, etc.
  metadata: {
    chartType?: string;
    styling?: any;
    description?: string;
  };
  aiInsights?: {
    summary: string;
    keyFindings: string[];
    recommendations?: string[];
    confidenceScore: number;
  };
}
```

---

## 5. Temporal.io Workflows

### Report Generation Workflow

```typescript
@WorkflowMethod
export class ReportGenerationWorkflow {
  
  async execute(request: ReportGenerationRequest): Promise<GeneratedReport> {
    
    // Step 1: Initialize report generation
    const reportId = await this.activities.initializeReport(request);
    
    // Step 2: Validate survey completion and access
    await this.activities.validateSurveyAccess(
      request.surveyId, 
      request.organizationId
    );
    
    // Step 3: Extract raw survey data
    const rawData = await this.activities.extractSurveyData(
      request.surveyId
    );
    
    // Step 4: Generate basic statistics (fast)
    const basicStats = await this.activities.generateBasicStatistics(
      rawData
    );
    await this.activities.saveReportComponent(
      reportId, 
      'basic_stats', 
      basicStats
    );
    
    // Step 5: Process components in parallel
    const componentPromises = [];
    
    if (request.includeAI) {
      // AI-powered components
      componentPromises.push(
        this.activities.generateSentimentAnalysis(rawData),
        this.activities.generateThemeAnalysis(rawData),
        this.activities.generateAnomalyDetection(rawData)
      );
    }
    
    // Standard components  
    componentPromises.push(
      this.activities.generateDemographicBreakdown(rawData),
      this.activities.generateVisualizationData(rawData),
      this.activities.generateComparisonAnalysis(rawData)
    );
    
    // Wait for all components to complete
    const components = await Promise.all(componentPromises);
    
    // Step 6: Save all components
    for (const component of components) {
      await this.activities.saveReportComponent(
        reportId,
        component.type,
        component
      );
    }
    
    // Step 7: Generate final report compilation
    const finalReport = await this.activities.compileReport(
      reportId,
      components
    );
    
    // Step 8: Mark report as completed
    await this.activities.completeReport(reportId, finalReport);
    
    // Step 9: Notify stakeholders
    await this.activities.notifyReportCompletion(
      request.organizationId,
      reportId
    );
    
    return finalReport;
  }
}
```

### Data Retention Workflow

```typescript
@WorkflowMethod
export class DataRetentionWorkflow {
  
  @WorkflowMethod
  async execute(): Promise<void> {
    
    // Step 1: Get retention policies for all organizations
    const policies = await this.activities.getRetentionPolicies();
    
    for (const policy of policies) {
      
      // Step 2: Process data tiering
      await this.activities.processDataTiering({
        organizationId: policy.organizationId,
        hotToWarmMonths: policy.hotToWarmMonths,
        warmToColdMonths: policy.warmToColdMonths
      });
      
      // Step 3: Archive expired analysis data
      await this.activities.archiveExpiredAnalysis({
        organizationId: policy.organizationId,
        retentionMonths: policy.analysisDataRetentionMonths
      });
      
      // Step 4: Clean up expired cache
      await this.activities.cleanupExpiredCache({
        organizationId: policy.organizationId,
        retentionMonths: policy.reportCacheRetentionMonths
      });
    }
    
    // Step 5: Generate retention report
    await this.activities.generateRetentionReport();
  }
}
```

---

## 6. AI Integration Architecture

### AI Processing Pipeline

```typescript
class AIAnalysisService {
  
  // Sentiment Analysis
  async analyzeSentiment(responses: SurveyResponse[]): Promise<SentimentAnalysis> {
    const textResponses = this.extractTextResponses(responses);
    
    const sentimentResults = await this.openAIClient.analyze({
      model: 'gpt-4',
      prompt: this.buildSentimentPrompt(textResponses),
      temperature: 0.1 // Low temperature for consistent results
    });
    
    return this.parseSentimentResults(sentimentResults);
  }
  
  // Theme Extraction  
  async extractThemes(responses: SurveyResponse[]): Promise<ThemeAnalysis> {
    const textResponses = this.extractTextResponses(responses);
    
    const themeResults = await this.openAIClient.analyze({
      model: 'gpt-4',
      prompt: this.buildThemeExtractionPrompt(textResponses),
      temperature: 0.3
    });
    
    return this.parseThemeResults(themeResults);
  }
  
  // Anomaly Detection
  async detectAnomalies(responses: SurveyResponse[]): Promise<AnomalyAnalysis> {
    const statisticalAnalysis = this.performStatisticalAnalysis(responses);
    
    const anomalies = await this.openAIClient.analyze({
      model: 'gpt-4',
      prompt: this.buildAnomalyPrompt(statisticalAnalysis),
      temperature: 0.1
    });
    
    return this.parseAnomalyResults(anomalies);
  }
}
```

### AI Prompt Templates

```typescript
const AI_PROMPTS = {
  sentiment: `
    Analyze the sentiment of these survey responses:
    {responses}
    
    Provide:
    1. Overall sentiment score (-1 to 1)
    2. Sentiment distribution (positive/neutral/negative percentages)
    3. Key emotional indicators
    4. Department-specific sentiment if applicable
    
    Format as JSON.
  `,
  
  themes: `
    Extract main themes from these survey responses:
    {responses}
    
    Identify:
    1. Top 5 themes with frequency counts
    2. Theme sentiment associations
    3. Relationships between themes
    4. Actionable themes for management
    
    Format as JSON.
  `,
  
  anomalies: `
    Analyze these survey statistics for anomalies:
    {statistics}
    
    Look for:
    1. Unusual response patterns
    2. Statistical outliers
    3. Inconsistent responses from individuals
    4. Potential data quality issues
    
    Format as JSON with confidence scores.
  `
};
```

---

## 7. Visualization Architecture

### Chart Component System

```typescript
interface ChartComponent {
  type: 'bar' | 'pie' | 'line' | 'scatter' | 'heatmap' | 'sankey';
  data: ChartData;
  options: ChartOptions;
  insights?: AIInsight[];
}

interface ChartData {
  labels: string[];
  datasets: Dataset[];
  metadata?: {
    totalResponses: number;
    dateRange: string;
    filters?: any;
  };
}

// Example chart configurations
const CHART_CONFIGS = {
  responseDistribution: {
    type: 'bar',
    title: 'Response Distribution by Question',
    responsive: true,
    exportable: true
  },
  
  sentimentOverTime: {
    type: 'line',
    title: 'Sentiment Trends',
    timeAxis: true,
    responsive: true
  },
  
  departmentComparison: {
    type: 'radar',
    title: 'Department Performance Comparison',
    multiAxis: true,
    responsive: true
  }
};
```

### PDF Generation System

```typescript
class PDFGenerationService {
  
  async generateReportPDF(reportId: string): Promise<Buffer> {
    
    // Step 1: Get report data
    const report = await this.getReportData(reportId);
    
    // Step 2: Build HTML template
    const html = await this.buildReportHTML(report);
    
    // Step 3: Generate PDF using Puppeteer
    const pdf = await this.puppeteerService.generatePDF({
      html: html,
      options: {
        format: 'A4',
        printBackground: true,
        margin: {
          top: '1in',
          right: '1in',
          bottom: '1in',
          left: '1in'
        }
      }
    });
    
    // Step 4: Log access for audit
    await this.auditService.logAccess({
      reportId: reportId,
      accessType: 'download_pdf',
      userId: this.getCurrentUser().id
    });
    
    return pdf;
  }
  
  private async buildReportHTML(report: ReportData): Promise<string> {
    return `
      <!DOCTYPE html>
      <html>
      <head>
        <title>${report.metadata.surveyTitle} - Report</title>
        <style>${await this.loadPDFStyles()}</style>
      </head>
      <body>
        <header>
          <h1>${report.metadata.surveyTitle}</h1>
          <div class="report-meta">
            Generated: ${report.metadata.generatedAt}
            Responses: ${report.metadata.responseCount}
          </div>
        </header>
        
        <main>
          ${this.renderComponents(report.components)}
        </main>
        
        <footer>
          <div class="page-number"></div>
        </footer>
      </body>
      </html>
    `;
  }
}
```

---

## 8. Role-Based Access Control

### Permission System

```typescript
interface ReportPermissions {
  canView: boolean;
  canDownload: boolean;
  canExport: boolean;
  canShare: boolean;
  visibleComponents: string[];
  dataFilters?: {
    departments?: string[];
    roles?: string[];
    dateRange?: DateRange;
  };
}

class ReportAccessControl {
  
  async checkReportAccess(
    reportId: string, 
    userId: string
  ): Promise<ReportPermissions> {
    
    const user = await this.userService.getUser(userId);
    const report = await this.reportService.getReport(reportId);
    
    // Check organization membership
    if (!this.isUserInOrganization(user, report.organizationId)) {
      return this.denyAccess();
    }
    
    // Check role requirements
    const hasRequiredRole = report.requiredRoles.some(role => 
      user.roles.includes(role)
    );
    
    if (!hasRequiredRole) {
      return this.denyAccess();
    }
    
    // Determine data filters based on user role
    const dataFilters = this.calculateDataFilters(user, report);
    
    return {
      canView: true,
      canDownload: this.canDownload(user, report),
      canExport: this.canExport(user, report),
      canShare: this.canShare(user, report),
      visibleComponents: this.getVisibleComponents(user, report),
      dataFilters: dataFilters
    };
  }
  
  private calculateDataFilters(user: User, report: Report): any {
    
    // Super Admin sees everything
    if (user.roles.includes('super_admin')) {
      return null;
    }
    
    // Company Admin sees their companies
    if (user.roles.includes('company_admin')) {
      return {
        organizations: user.managedOrganizations
      };
    }
    
    // Department Manager sees their department
    if (user.roles.includes('dept_manager')) {
      return {
        departments: [user.department],
        roles: ['employee'] // Can't see other manager responses
      };
    }
    
    // Standard user sees only aggregated data
    return {
      aggregatedOnly: true
    };
  }
}
```

---

## 9. Performance & Scalability

### Caching Strategy

```typescript
class ReportCacheService {
  
  // Multi-level caching
  private readonly levels = {
    L1: 'memory', // Fast access for active reports
    L2: 'redis',  // Session-based caching
    L3: 'database' // Persistent component caching
  };
  
  async getCachedComponent(
    reportId: string, 
    componentType: string
  ): Promise<ReportComponent | null> {
    
    // Try L1 cache first
    let component = await this.memoryCache.get(`${reportId}:${componentType}`);
    if (component) return component;
    
    // Try L2 cache
    component = await this.redisCache.get(`${reportId}:${componentType}`);
    if (component) {
      // Promote to L1
      await this.memoryCache.set(`${reportId}:${componentType}`, component, 300);
      return component;
    }
    
    // Try L3 cache
    component = await this.databaseCache.getComponent(reportId, componentType);
    if (component) {
      // Promote to L2 and L1
      await this.redisCache.set(`${reportId}:${componentType}`, component, 3600);
      await this.memoryCache.set(`${reportId}:${componentType}`, component, 300);
      return component;
    }
    
    return null;
  }
}
```

### Database Optimization

```sql
-- Partitioning for large datasets
CREATE TABLE survey_responses_partitioned (
    id UUID,
    survey_id UUID,
    created_at TIMESTAMP,
    -- other columns
) PARTITION BY RANGE (created_at);

-- Monthly partitions
CREATE TABLE survey_responses_2024_01 PARTITION OF survey_responses_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE survey_responses_2024_02 PARTITION OF survey_responses_partitioned
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Indexes for common query patterns
CREATE INDEX CONCURRENTLY idx_survey_responses_survey_created 
    ON survey_responses_partitioned (survey_id, created_at);

CREATE INDEX CONCURRENTLY idx_survey_responses_org_created
    ON survey_responses_partitioned (organization_id, created_at);
```

---

## 10. Monitoring & Observability

### Key Metrics

```typescript
const REPORTING_METRICS = {
  // Performance Metrics
  reportGenerationTime: 'histogram',
  componentProcessingTime: 'histogram',
  aiProcessingLatency: 'histogram',
  pdfGenerationTime: 'histogram',
  
  // Business Metrics
  reportsGenerated: 'counter',
  aiAnalysisRequests: 'counter',
  pdfDownloads: 'counter',
  reportViews: 'counter',
  
  // Error Metrics
  reportGenerationFailures: 'counter',
  aiAnalysisFailures: 'counter',
  pdfGenerationFailures: 'counter',
  
  // Resource Metrics
  aiCostPerReport: 'gauge',
  storageUsage: 'gauge',
  cacheHitRate: 'gauge'
};
```

### Health Checks

```typescript
class ReportingHealthCheck {
  
  async checkHealth(): Promise<HealthStatus> {
    const checks = await Promise.allSettled([
      this.checkDatabase(),
      this.checkAIService(),
      this.checkTemporalWorkflows(),
      this.checkCacheService(),
      this.checkStorageService()
    ]);
    
    return {
      status: this.aggregateStatus(checks),
      timestamp: new Date(),
      details: this.formatCheckResults(checks)
    };
  }
}
```

---

## 11. Deployment & Infrastructure

### Container Configuration

```yaml
# docker-compose.yml for Reporting Service
services:
  reporting-service:
    build: ./reporting-service
    environment:
      - DATABASE_URL=postgresql://user:pass@postgres:5432/reporting_service
      - REDIS_URL=redis://redis:6379
      - TEMPORAL_HOST=temporal:7233
      - OPENAI_API_KEY=${OPENAI_API_KEY}
    resources:
      memory: 4GB
      cpu: 2
    replicas: 2
    
  temporal-worker:
    build: ./temporal-worker
    environment:
      - TEMPORAL_HOST=temporal:7233
      - REPORTING_SERVICE_URL=http://reporting-service:3000
    resources:
      memory: 2GB  
      cpu: 1
    replicas: 3
```

### Environment Configuration

```typescript
const CONFIG = {
  development: {
    ai: {
      provider: 'openai',
      model: 'gpt-3.5-turbo', // Cheaper for dev
      maxTokens: 2000,
      timeout: 30000
    },
    storage: {
      type: 'local',
      retentionDays: 30
    }
  },
  
  production: {
    ai: {
      provider: 'openai', 
      model: 'gpt-4',
      maxTokens: 4000,
      timeout: 120000,
      fallbackModel: 'gpt-3.5-turbo'
    },
    storage: {
      type: 's3',
      bucket: 'survey-reports-prod',
      retentionPolicy: 'tiered'
    }
  }
};
```

---

## 12. Implementation Roadmap

### Phase 1: Core Infrastructure (4-6 weeks)
- Database schema implementation
- Basic REST API endpoints
- Temporal.io workflow setup
- Role-based access control
- Basic report generation workflow

### Phase 2: Basic Analytics (2-3 weeks)
- Statistical analysis components
- Basic visualization generation
- PDF generation system
- Report caching implementation

### Phase 3: AI Integration (3-4 weeks)
- OpenAI API integration
- Sentiment analysis implementation
- Basic theme extraction
- AI result caching

### Phase 4: Advanced Features (4-6 weeks)
- Advanced AI analytics
- Data retention automation
- Performance optimization
- Comprehensive monitoring

### Phase 5: Production Ready (2-3 weeks)
- Security hardening
- Performance testing
- Documentation
- Deployment automation

---

## 13. Security Considerations

### Data Protection
- Encrypt sensitive survey responses at rest
- Use secure connections for all AI API calls
- Implement proper access logging and audit trails
- Regular security scans for dependencies

### AI Security
- Sanitize inputs sent to AI services
- Implement rate limiting for AI requests
- Monitor for prompt injection attempts
- Use separate AI keys per environment

### Access Control
- JWT-based authentication
- Role-based authorization
- API rate limiting
- Request/response validation

---

This architectural design guide provides a comprehensive foundation for implementing the Reporting Module. Each section can be expanded based on specific requirements and technical constraints discovered during implementation.