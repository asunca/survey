# Smart Organization Schema Parser Module - System Architecture

_Project Requirements Document - Module Overview_

## Executive Summary

The Smart Organization Schema Parser is an AI-powered module that automatically extracts and structures organizational hierarchies from various input sources (Excel, CSV, text) in Turkish and English. The system intelligently analyzes employee data, infers hierarchical relationships, and provides an interactive interface for users to refine the organizational structure through conversational feedback until the final schema is approved.

## Module Scope

**Primary Function:** Automated extraction, analysis, and refinement of organizational hierarchies from unstructured data sources

**Integration Role:** Complementary submodule within the existing project ecosystem, sharing infrastructure with the Smart Survey Generator Module

**Target Capacity:** Support for organizations with 1000+ employees across 50+ concurrent parsing sessions

## Key Features

### Core Functionality

- **Multi-Source Input Processing:** Support for Excel (.xlsx), CSV files, and natural language text input
- **Intelligent Hierarchy Inference:** AI-powered analysis to determine organizational structure from roles, teams, and reporting relationships
- **Interactive Refinement:** Conversational interface for iterative schema correction and approval
- **Multi-language Support:** Process organizational data in Turkish and English
- **Visual Tree Representation:** Dynamic organizational chart generation with drag-drop capabilities
- **Real-time Validation:** Continuous validation of organizational structure integrity

### Advanced Capabilities

- **Role-Based Hierarchy Detection:** Automatic inference of reporting relationships based on job titles and roles
- **Team Clustering:** Intelligent grouping of employees into teams/tribes based on contextual information
- **Missing Data Inference:** AI-powered suggestions for incomplete organizational information
- **Conflict Resolution:** Smart handling of ambiguous or conflicting hierarchical relationships
- **Export Flexibility:** Multiple output formats (JSON, XML, Excel, visual diagrams)

## System Architecture

### High-Level Components (Shared Infrastructure)

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   API Gateway   │───▶│   MCP Server    │───▶│   LLM Service   │
│   (Interface)   │    │ (Core Logic)    │    │  (AI Analysis)  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   React UI      │    │   PostgreSQL    │    │  File Processor │
│ (Tree Builder)  │    │ + Org Schema DB │    │   (Excel/CSV)   │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                                               │
         ▼                                               ▼
┌─────────────────┐                            ┌─────────────────┐
│   WebSocket     │                            │   LLM Analysis  │
│ (Real-time UI)  │                            │   Engine        │
└─────────────────┘                            └─────────────────┘
```

### Module-Specific Services

**Organization Parser Service (Node.js)**

- File upload and processing coordination
- Hierarchical relationship inference engine
- Schema validation and integrity checking
- Conversational refinement management

**File Processing Pipeline**

- **Excel/CSV Parser:** Advanced spreadsheet analysis with LLM-driven column detection
- **Text Processor:** NLP-based extraction from unstructured text
- **Data Normalizer:** Standardization of extracted organizational data
- **Hierarchy Builder:** Tree structure generation from flat data

**Interactive Refinement Engine**

- **Conversation Handler:** Process user feedback and modification requests
- **Schema Transformer:** Apply structural changes based on user input
- **Conflict Resolver:** Handle ambiguous organizational relationships
- **Approval Workflow:** Track changes and maintain approval states

**Visualization Service**

- **Tree Renderer:** Generate interactive organizational charts
- **Export Engine:** Multiple format output generation
- **Real-time Updates:** WebSocket-based live schema updates

## Technical Specifications

### Enhanced Database Schema (PostgreSQL)

````sql
-- Organization Sessions with Flexible Context Storage
CREATE TABLE org_sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID NOT NULL,
    session_name VARCHAR(255),
    input_type VARCHAR(20) CHECK (input_type IN ('file', 'text')),
    input_source TEXT, -- File path or raw text
    language VARCHAR(5) DEFAULT 'en',

    -- Flexible organizational context as JSON
    organizational_context JSONB NOT NULL DEFAULT '{}', -- All LLM-detected context
    agile_maturity JSONB, -- Agile transformation assessment
    consultation_context JSONB, -- Consulting-specific context

    status VARCHAR(20) DEFAULT 'processing',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes for JSON queries
CREATE INDEX idx_org_sessions_org_context ON org_sessions USING GIN (organizational_context);
CREATE INDEX idx_org_sessions_agile_maturity ON org_sessions USING GIN (agile_maturity);
CREATE INDEX idx_org_sessions_consultation ON org_sessions USING GIN (consultation_context);

-- Enhanced Organizational Entities with Flexible Structure
CREATE TABLE org_entities (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES org_sessions(id) ON DELETE CASCADE,
    name VARCHAR(255),
    email VARCHAR(255) NOT NULL,
    role VARCHAR(255),

    -- Flexible organizational data as JSON
    organizational_data JSONB NOT NULL DEFAULT '{}', -- All org-specific fields
    agile_roles JSONB, -- Agile-specific role information
    team_memberships JSONB, -- All team memberships and allocations

    -- LLM analysis results
    hierarchy_level INTEGER, -- 1-10 scale (1 = highest)
    extraction_confidence DECIMAL(3,2), -- AI confidence in data extraction
    hierarchy_confidence DECIMAL(3,2), -- AI confidence in hierarchy placement
    is_validated BOOLEAN DEFAULT false,

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes for JSON queries
CREATE INDEX idx_org_entities_org_data ON org_entities USING GIN (organizational_data);
CREATE INDEX idx_org_entities_agile_roles ON org_entities USING GIN (agile_roles);
CREATE INDEX idx_org_entities_team_memberships ON org_entities USING GIN (team_memberships);

-- Enhanced Hierarchical Relationships
CREATE TABLE org_relationships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES org_sessions(id) ON DELETE CASCADE,
    parent_id UUID REFERENCES org_entities(id),
    child_id UUID REFERENCES org_entities(id),

    -- Relationship types for different org structures
    relationship_type VARCHAR(50), -- 'direct_report', 'functional_report', 'matrix_report', 'team_lead', 'peer'
    relationship_strength VARCHAR(20), -- 'primary', 'secondary', 'temporary', 'cross_functional'

    -- Context-aware confidence
    confidence_score DECIMAL(3,2),
    inference_method VARCHAR(50), -- 'explicit_manager', 'role_hierarchy', 'team_structure', 'llm_inference'

    is_approved BOOLEAN DEFAULT false,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(session_id, parent_id, child_id, relationship_type)
);

-- Simplified Teams with JSON Flexibility
CREATE TABLE org_teams (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES org_sessions(id) ON DELETE CASCADE,
    team_name VARCHAR(255) NOT NULL,
    parent_team_id UUID REFERENCES org_teams(id),
    team_lead_id UUID REFERENCES org_entities(id),

    -- All team data as flexible JSON
    team_data JSONB NOT NULL DEFAULT '{}', -- All team characteristics and attributes
    agile_practices JSONB, -- Agile-specific team information

    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Add indexes for JSON queries
CREATE INDEX idx_org_teams_team_data ON org_teams USING GIN (team_data);
CREATE INDEX idx_org_teams_agile_practices ON org_teams USING GIN (agile_practices);

-- Team Memberships (for matrix organizations)
CREATE TABLE org_team_memberships (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    entity_id UUID REFERENCES org_entities(id),
    team_id UUID REFERENCES org_teams(id),
    membership_type VARCHAR(20), -- 'primary', 'secondary', 'temporary'
    role_in_team VARCHAR(100), -- 'member', 'lead', 'coordinator'
    allocation_percentage INTEGER DEFAULT 100, -- For matrix organizations
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    UNIQUE(entity_id, team_id, membership_type)
);

-- LLM Analysis Results Storage
CREATE TABLE llm_analysis_results (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    session_id UUID REFERENCES org_sessions(id) ON DELETE CASCADE,
    analysis_type VARCHAR(50), -- 'column_detection', 'hierarchy_inference', 'team_structure'
    input_data JSONB, -- Input sent to LLM
    llm_response JSONB, -- Raw LLM response
    processed_result JSONB, -- Processed and validated result
    confidence_score DECIMAL(3,2),
    processing_time_ms INTEGER,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

**Example JSON Data Structures:**

```javascript
// organizational_context example
{
  "detectedIndustry": "technology",
  "organizationType": "scale-up",
  "currentStructure": "hybrid",
  "teamNamingConventions": ["squad", "tribe", "chapter"],
  "leadershipModel": "servant_leadership",
  "decisionMaking": "distributed",
  "workOrganization": "product_based"
}

// agile_maturity example
{
  "currentMaturity": "developing",
  "agileIndicators": ["scrum_masters_present", "product_owners", "sprints_mentioned"],
  "frameworks": ["scrum", "kanban"],
  "transformationReadiness": "high",
  "scalingApproach": "spotify_model",
  "impediments": ["limited_autonomy", "traditional_governance"],
  "opportunities": ["strong_tech_culture", "leadership_buy_in"]
}

// consultation_context example
{
  "engagementType": "transformation",
  "focusAreas": ["team_formation", "leadership_development"],
  "successFactors": ["executive_sponsorship", "change_champions"],
  "riskFactors": ["middle_management_resistance"],
  "quickWins": ["team_retrospectives", "daily_standups"]
}

// organizational_data (for entities) example
{
  "primaryTeam": "payments_squad",
  "secondaryTeams": ["platform_guild", "security_chapter"],
  "department": "engineering",
  "location": "istanbul",
  "employeeId": "EMP001",
  "startDate": "2023-01-15",
  "seniority": "senior"
}

// agile_roles example
{
  "currentRole": "team_lead",
  "agileEquivalent": "scrum_master",
  "certifications": ["PSM I", "CSPO"],
  "coachingCapability": "developing",
  "changeAgent": true
}

// team_data example
{
  "teamType": "feature_team",
  "teamSize": 8,
  "teamFunction": "payments",
  "crossFunctional": true,
  "autonomyLevel": "high",
  "maturityLevel": "performing"
}

// agile_practices example
{
  "ceremonies": ["daily_standup", "sprint_planning", "retrospective"],
  "framework": "scrum",
  "sprintLength": "2_weeks",
  "velocityTrend": "stable",
  "impediments": ["dependency_on_legacy_team"]
}
````

````

### LLM Service Integration

**Agile Consulting Specialized LLM Service:**
```javascript
class AgileConsultingLLMService {
  constructor() {
    this.openaiClient = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });
    this.agileKnowledgeBase = this.loadAgileFrameworks();
  }

  async analyzeOrganizationalData(rawData, context) {
    const systemPrompt = `
      You are a senior agile transformation consultant with 15+ years of experience helping organizations adopt agile practices. Your expertise includes:

      AGILE FRAMEWORKS & METHODOLOGIES:
      - Scrum, Kanban, SAFe (Scaled Agile Framework), LeSS (Large-Scale Scrum)
      - Spotify Model (Squads, Tribes, Chapters, Guilds)
      - Framework-less agile approaches and custom implementations
      - DevOps and continuous delivery practices
      - Design thinking and lean startup methodologies

      AGILE TRANSFORMATION PATTERNS:
      - Traditional to agile organizational restructuring
      - Scaling agile across enterprise organizations
      - Building agile operating models and governance
      - Agile maturity assessment and roadmapping
      - Change management for agile adoption

      ORGANIZATIONAL DESIGN FOR AGILITY:
      - Cross-functional team formation
      - Product-oriented team structures vs project-based
      - Network organizations and decentralized decision making
      - Agile roles: Product Owner, Scrum Master, Agile Coach, etc.
      - Leadership models for agile organizations

      CLIENT CONTEXT AWARENESS:
      - Assess current agile maturity level (Traditional, Developing, Defined, Optimizing)
      - Identify transformation challenges and opportunities
      - Recognize anti-patterns and organizational impediments
      - Understand business domain and market pressures
    `;

    const analysisPrompt = `
      As an agile consultant, analyze this client organization's structure for agile transformation consulting.

      Client Context: ${context.clientType || 'Unknown'} organization in ${context.sector} sector
      Transformation Stage: ${context.transformationStage || 'Assessment phase'}

      Data Sample:
      Headers: ${JSON.stringify(rawData[0])}
      Sample Rows: ${JSON.stringify(rawData.slice(1, 6))}

      ANALYSIS OBJECTIVES:
      1. Map current organizational structure and identify agile maturity indicators
      2. Detect existing agile practices, roles, and team structures
      3. Identify transformation opportunities and potential challenges
      4. Assess readiness for different agile scaling approaches

      LOOK FOR AGILE INDICATORS:
      - Agile roles: Scrum Master, Product Owner, Agile Coach, DevOps Engineer
      - Team structures: Cross-functional teams, feature teams, component teams
      - Agile terminology: Sprint, Epic, Story, Backlog, Retrospective
      - Modern team names: Squad, Tribe, Chapter, Guild, Pod, Stream
      - DevOps practices: CI/CD roles, Site Reliability, Platform teams

      ASSESS ORGANIZATIONAL PATTERNS:
      - Command and control vs servant leadership indicators
      - Functional silos vs cross-functional collaboration
      - Project vs product organization
      - Traditional titles vs modern agile roles
      - Hierarchical depth vs flat structure

      TRANSFORMATION CONTEXT ANALYSIS:
      - Current agile maturity level based on roles and structure
      - Potential scaling framework fit (SAFe, LeSS, Spotify, custom)
      - Change readiness indicators
      - Cultural and leadership alignment signals

      Return comprehensive consulting-focused analysis:
      {
        "columnMappings": {
          "name": "detected_column_for_names",
          "email": "detected_column_for_emails",
          "role": "detected_column_for_roles",
          "team": "detected_column_for_teams",
          "manager": "detected_column_for_managers",
          "additionalFields": {
            "agileRole": "if_agile_specific_roles_detected",
            "chapter": "if_spotify_model_detected",
            "guild": "if_guild_structure_detected",
            "stream": "if_value_stream_structure"
          }
        },
        "organizationalContext": {
          "detectedIndustry": "specific_industry",
          "organizationType": "startup|scale-up|enterprise|government",
          "currentStructure": "hierarchical|flat|matrix|network|hybrid",
          "teamNamingConventions": ["actual_terms_found"],
          "leadershipModel": "command_control|servant_leadership|hybrid",
          "decisionMaking": "centralized|distributed|mixed",
          "workOrganization": "project_based|product_based|mixed"
        },
        "agileMaturityAssessment": {
          "currentMaturity": "traditional|developing|defined|optimizing",
          "agileIndicators": ["detected_agile_practices_and_roles"],
          "frameworks": ["detected_frameworks: scrum|kanban|safe|less|spotify"],
          "transformationReadiness": "low|medium|high",
          "scalingApproach": "recommended_scaling_framework",
          "impediments": ["identified_organizational_barriers"],
          "opportunities": ["transformation_opportunities"]
        },
        "consultingContext": {
          "engagementType": "assessment|coaching|transformation|scaling",
          "focusAreas": ["areas_needing_attention"],
          "successFactors": ["critical_success_factors"],
          "riskFactors": ["transformation_risks"],
          "quickWins": ["immediate_improvement_opportunities"]
        },
        "teamAnalysis": {
          "teamMaturity": "forming|storming|norming|performing",
          "crossFunctional": "assessment_of_cross_functionality",
          "autonomy": "team_decision_making_authority",
          "collaboration": "inter_team_collaboration_patterns"
        },
        "recommendations": [
          "specific_consulting_recommendations_for_this_organization"
        ]
      }
    `;

    const response = await this.openaiClient.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: systemPrompt },
        { role: "user", content: analysisPrompt }
      ],
      temperature: 0.1,
      response_format: { type: "json_object" }
    });

    return JSON.parse(response.choices[0].message.content);
  }

  async inferHierarchicalStructure(entities, organizationalContext) {
    const hierarchyPrompt = `
      As an agile transformation consultant, analyze this organization's hierarchy for agile coaching opportunities.

      Client Profile:
      - Industry: ${organizationalContext.detectedIndustry}
      - Current Structure: ${organizationalContext.currentStructure}
      - Agile Maturity: ${organizationalContext.agileMaturityAssessment?.currentMaturity}
      - Team Conventions: ${organizationalContext.teamNamingConventions?.join(', ')}

      Employee Data:
      ${JSON.stringify(entities.slice(0, 50))}

      CONSULTING OBJECTIVES:
      1. Map current reporting structures and identify agile transformation opportunities
      2. Assess leadership alignment with agile principles
      3. Identify servant leadership vs command-and-control patterns
      4. Recommend optimal team formations for agility
      5. Plan change management approach based on current structure

      AGILE HIERARCHY CONSIDERATIONS:
      - Minimize management layers for faster decision-making
      - Identify servant leaders vs traditional managers
      - Assess span of control for agile team formation
      - Look for product ownership structures
      - Identify coaching and mentoring relationships

      TRANSFORMATION FOCUS AREAS:
      - Leadership: Who are the change champions and potential resistors?
      - Teams: How can current teams be reformed into agile, cross-functional units?
      - Governance: What decision-making authorities need to be distributed?
      - Roles: Which traditional roles need agile counterparts or evolution?
      - Culture: What cultural patterns support or hinder agile adoption?

      SCALING FRAMEWORK ASSESSMENT:
      Based on size and structure, evaluate fit for:
      - Scrum of Scrums (smaller organizations)
      - SAFe (large enterprises with governance needs)
      - LeSS (product-focused organizations)
      - Spotify Model (technology companies)
      - Custom agile operating model

      For each employee, determine:
      - Current hierarchy level and transformation impact
      - Agile role potential (current role → future agile role)
      - Change readiness and influence level
      - Team formation recommendations
      - Leadership development needs

      Return consulting-focused hierarchy analysis with transformation roadmap insights.
    `;

    const response = await this.openaiClient.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are a senior agile transformation consultant specializing in organizational design for agility." },
        { role: "user", content: hierarchyPrompt }
      ],
      temperature: 0.1,
      response_format: { type: "json_object" }
    });

    return JSON.parse(response.choices[0].message.content);
  }

  async analyzeTextInput(textInput, organizationType, sector) {
    const textAnalysisPrompt = `
      As an agile consultant, extract organizational insights from this client description for transformation planning.

      Client: ${organizationType} in ${sector} sector
      Description: "${textInput}"

      EXTRACTION GOALS:
      1. Current organizational structure and roles
      2. Existing agile practices and maturity indicators
      3. Leadership and cultural patterns
      4. Team structures and collaboration patterns
      5. Transformation context and readiness signals

      AGILE CONSULTING LENS:
      - Identify current agile adoption level
      - Look for transformation readiness indicators
      - Spot organizational impediments to agility
      - Assess leadership support for change
      - Evaluate team formation potential

      Handle various input formats:
      - Strategic transformation documents
      - Current state assessments
      - Organizational charts with commentary
      - Team descriptions and role definitions
      - Mixed Turkish/English agile terminology

      Return structured data optimized for agile transformation consulting.
    `;

    const response = await this.openaiClient.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are an agile transformation consultant analyzing client organizational data." },
        { role: "user", content: textAnalysisPrompt }
      ],
      temperature: 0.1,
      response_format: { type: "json_object" }
    });

    return JSON.parse(response.choices[0].message.content);
  }

  async resolveOrganizationalAmbiguities(conflictingData, context) {
    const resolutionPrompt = `
      As an agile consultant, resolve organizational ambiguities using agile transformation best practices.

      Client Context: ${JSON.stringify(context)}
      Conflicting Data: ${JSON.stringify(conflictingData)}

      CONSULTING APPROACH:
      1. Apply agile principles to resolve structural conflicts
      2. Recommend changes that support transformation goals
      3. Balance current state reality with target state vision
      4. Consider change management and adoption challenges

      AGILE RESOLUTION PRINCIPLES:
      - Favor cross-functional over functional structures
      - Prefer flat over hierarchical when possible
      - Support servant leadership over command-and-control
      - Enable team autonomy while maintaining alignment
      - Optimize for flow of value over resource utilization

      TRANSFORMATION CONSIDERATIONS:
      - What resolution best supports agile adoption?
      - How can conflicts become transformation opportunities?
      - What interim structures might be needed during transition?
      - Which stakeholders need change management support?

      Provide consulting recommendations that resolve ambiguities while advancing agile transformation.
    `;

    const response = await this.openaiClient.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are an agile transformation consultant resolving organizational design challenges." },
        { role: "user", content: resolutionPrompt }
      ],
      temperature: 0.2
    });

    return JSON.parse(response.choices[0].message.content);
  }
}
````

### LLM-Powered Hierarchy Inference

**Context-Aware Organizational Analysis:**

```javascript
class LLMHierarchyInferenceEngine {
  async inferOrganizationalStructure(entities, organizationalContext) {
    // Industry and structure-specific analysis
    const hierarchyAnalysisPrompt = `
      Analyze this ${organizationalContext.industryType} organization with ${
      organizationalContext.structureType
    } structure.
      
      Team naming conventions in this context: ${organizationalContext.teamNamingConvention.join(", ")}
      Expected hierarchy levels: ${organizationalContext.hierarchyLevels.join(" → ")}
      
      Employee data:
      ${JSON.stringify(entities.slice(0, 20))} // Sample for analysis
      
      Please determine:
      1. Hierarchical relationships between employees
      2. Team/department groupings with proper naming
      3. Reporting structure based on roles and titles
      4. Any matrix or cross-functional relationships
      5. Identify organizational patterns specific to this industry
      
      For each employee, determine:
      - Their position in the hierarchy (level 1-10, 1 being highest)
      - Direct reports (if any)
      - Team/department membership
      - Cross-functional relationships
      
      Consider industry-specific patterns:
      - Technology: Engineering Teams, Product Teams, Squads, Tribes
      - Healthcare: Departments, Units, Specialties, Medical Teams
      - Finance: Business Units, Functions, Divisions
      - Manufacturing: Plants, Lines, Departments, Shifts
      - Consulting: Practices, Offices, Client Teams, Competencies
      - Startups: Flat structures, Cross-functional teams
      
      Return structured hierarchy with confidence scores.
    `;

    const hierarchyResult = await this.llmService.analyzeHierarchy(hierarchyAnalysisPrompt);
    return this.processHierarchyResult(hierarchyResult, entities);
  }

  async analyzeTeamStructures(entities, organizationalContext) {
    const teamAnalysisPrompt = `
      Given this ${organizationalContext.industryType} organization, identify and structure all teams/departments.
      
      Context:
      - Structure Type: ${organizationalContext.structureType}
      - Industry: ${organizationalContext.industryType}
      - Cultural Context: ${organizationalContext.culturalContext}
      
      Employee data with roles and teams:
      ${JSON.stringify(entities)}
      
      Please:
      1. Group employees into logical teams/departments
      2. Identify team leaders/managers
      3. Recognize cross-functional team memberships
      4. Handle industry-specific organizational units:
         * Technology: Feature Teams, Platform Teams, Infrastructure
         * Healthcare: Clinical Departments, Support Services, Administration
         * Finance: Front Office, Middle Office, Back Office
         * Retail: Store Operations, Regional Management, Corporate
         * Manufacturing: Production, Quality, Maintenance, Logistics
         * Education: Academic Departments, Administration, Student Services
      
      5. Account for modern organizational patterns:
         * Agile/Scrum teams in tech companies
         * Cross-functional pods in startups
         * Center of Excellence models in enterprises
         * Matrix structures in consulting
         * Regional structures in multinational companies
      
      Return team structure with proper naming conventions for this industry.
    `;

    return await this.llmService.analyzeTeams(teamAnalysisPrompt);
  }

  async resolveAmbiguousRelationships(conflictingData, organizationalContext) {
    const resolutionPrompt = `
      Resolve these conflicting organizational relationships in a ${organizationalContext.industryType} context:
      
      Conflicting data: ${JSON.stringify(conflictingData)}
      
      Industry context: ${organizationalContext.industryType}
      Common patterns: ${organizationalContext.culturalContext}
      
      Consider:
      1. Industry-specific hierarchy patterns
      2. Regional/cultural organizational norms
      3. Modern organizational structures (flat, agile, matrix)
      4. Dual reporting relationships in matrix organizations
      5. Temporary vs permanent team assignments
      
      Provide the most logical organizational structure with explanations.
    `;

    return await this.llmService.resolveConflicts(resolutionPrompt);
  }
}
```

### Conversational Refinement System

**Interactive Modification Handler:**

```javascript
class SchemaRefinementEngine {
  async processUserFeedback(sessionId, userInput, currentSchema) {
    // Parse user intent
    const intent = await this.parseModificationIntent(userInput);

    switch (intent.action) {
      case "move_employee":
        return await this.moveEmployee(sessionId, intent.employee, intent.newParent);
      case "create_team":
        return await this.createTeam(sessionId, intent.teamName, intent.members);
      case "merge_departments":
        return await this.mergeDepartments(sessionId, intent.departments);
      case "update_role":
        return await this.updateEmployeeRole(sessionId, intent.employee, intent.newRole);
    }
  }

  async parseModificationIntent(userInput) {
    const analysisPrompt = `
      Analyze this organizational modification request:
      "${userInput}"
      
      Extract:
      - Action type (move_employee, create_team, merge_departments, update_role)
      - Target entities (employee names, team names)
      - Desired changes
      
      Return structured JSON with the intent.
    `;

    return await this.llmService.parseIntent(analysisPrompt);
  }
}
```

### Real-time Visualization Component

**Interactive Org Chart (React):**

```javascript
const OrganizationalChart = ({ sessionId }) => {
  const [orgData, setOrgData] = useState(null);
  const [selectedNode, setSelectedNode] = useState(null);

  // Real-time updates via WebSocket
  useEffect(() => {
    const ws = new WebSocket(`/ws/org-session/${sessionId}`);
    ws.onmessage = (event) => {
      const update = JSON.parse(event.data);
      setOrgData((prevData) => applySchemaUpdate(prevData, update));
    };
  }, [sessionId]);

  const handleNodeDrag = async (nodeId, newParentId) => {
    await fetch(`/api/v1/org-sessions/${sessionId}/move`, {
      method: "POST",
      body: JSON.stringify({ nodeId, newParentId }),
    });
  };

  return (
    <div className="org-chart-container">
      <Tree
        data={orgData}
        onNodeSelect={setSelectedNode}
        onNodeDrag={handleNodeDrag}
        renderCustomNode={({ node }) => (
          <OrgNode data={node} isValidated={node.is_validated} confidenceScore={node.confidence_score} />
        )}
      />
      {selectedNode && (
        <NodeDetailsPanel node={selectedNode} onUpdate={(updates) => updateNode(selectedNode.id, updates)} />
      )}
    </div>
  );
};
```

## Data Flow

### Complete Processing Pipeline

1. **Input Reception & Analysis**

   - User uploads file or provides text input
   - System detects input type and language
   - File validation and format analysis

2. **LLM-Driven Data Extraction & Normalization**

   - **File Processing:** LLM analyzes spreadsheet with industry expertise
   - **Column Detection:** AI identifies all organizational fields with contextual understanding
   - **Text Processing:** NLP-based entity extraction from unstructured text
   - **Data Validation:** Email format validation, duplicate detection with LLM assistance
   - **Confidence Scoring:** AI-based confidence assessment for each data point

3. **Hierarchical Relationship Inference**

   - **Industry-Specific Analysis:** LLM applies sector knowledge for hierarchy patterns
   - **Context-Aware Inference:** Considers organizational type, culture, and structure
   - **Multi-Dimensional Relationships:** Handles matrix, functional, and cross-functional structures
   - **Conflict Resolution:** LLM resolves ambiguous or conflicting relationships
   - **Confidence Weighting:** Score each relationship inference with contextual awareness

4. **Initial Schema Generation**

   - **Tree Construction:** Build hierarchical organization structure using LLM insights
   - **Validation Rules:** Check for circular references, orphaned nodes with AI assistance
   - **Gap Analysis:** Identify missing relationships or incomplete data
   - **Visual Representation:** Generate interactive organizational chart

5. **Interactive Refinement Loop**

   - **User Review:** Present schema with confidence indicators and LLM explanations
   - **Feedback Processing:** Parse natural language modification requests using LLM
   - **Schema Updates:** Apply changes while maintaining data integrity
   - **Real-time Visualization:** Update charts and notify connected clients
   - **Approval Tracking:** Monitor user satisfaction and completion status

6. **Final Schema Output**
   - **Export Generation:** Create files in requested formats (JSON, Excel, PDF)
   - **Database Storage:** Persist final approved schema with full context
   - **Integration Ready:** Prepare for consumption by other system modules

## Integration with Existing Infrastructure

### Shared Components Utilization

**Database Layer:**

- Extend existing PostgreSQL setup with organizational schema tables
- Utilize existing LLM service infrastructure for consistent AI processing
- Leverage caching layer for frequently accessed organizational data

**LLM Service Integration:**

- Reuse existing OpenAI/Claude API integration and rate limiting
- Extend prompt templates for organizational analysis
- Utilize same error handling and retry mechanisms

**Authentication & Authorization:**

- Integrate with existing user management system
- Implement role-based access for organizational data
- Maintain same security protocols and audit trails

### API Extensions

**New Endpoints:**

```
POST /api/v1/org-sessions                    # Create new parsing session
PUT  /api/v1/org-sessions/{id}/upload        # Upload file for processing
POST /api/v1/org-sessions/{id}/text          # Submit text input
GET  /api/v1/org-sessions/{id}/schema        # Get current schema
POST /api/v1/org-sessions/{id}/modify        # Apply schema modifications
POST /api/v1/org-sessions/{id}/approve       # Approve final schema
GET  /api/v1/org-sessions/{id}/export        # Export schema in various formats
```

**WebSocket Endpoints:**

```
/ws/org-session/{sessionId}                 # Real-time schema updates
/ws/org-chart/{sessionId}                   # Live chart synchronization
```

## Performance Considerations

### Scalability Targets

- **File Processing:** Handle Excel files up to 10MB with 5,000+ employees
- **Real-time Updates:** Support 50+ concurrent users editing same schema
- **Response Times:** < 3 seconds for initial schema generation, < 500ms for modifications
- **Memory Management:** Efficient handling of large organizational datasets
- **LLM Performance:** Optimized prompt batching for large employee datasets

### Optimization Strategies

- **Streaming Processing:** Process large files in chunks with LLM analysis
- **Intelligent Caching:** Cache LLM analysis results and organizational patterns
- **Database Optimization:** Efficient tree queries using recursive CTEs
- **WebSocket Management:** Optimized real-time updates with message batching
- **LLM Optimization:** Batch similar analysis requests, cache industry patterns

## Security & Compliance

### Data Protection

- **PII Handling:** Secure processing of employee personal information
- **Access Control:** Granular permissions for organizational data viewing/editing
- **Data Retention:** Configurable retention policies for processed schemas
- **Audit Logging:** Complete trail of all schema modifications and approvals
- **LLM Privacy:** Ensure organizational data privacy in LLM processing

### Privacy Considerations

- **Minimal Data Storage:** Store only essential organizational information
- **Anonymization Options:** Support for anonymized organizational charts
- **GDPR Compliance:** Data subject rights and consent management
- **Cross-border Data:** Compliance with data localization requirements

## Implementation Timeline (AI-Assisted)

### Phase 1: LLM-Driven Core Processing Engine

- **AI Tasks:** File parsing with LLM analysis, data extraction, basic hierarchy inference
- **Human Tasks:** Database schema design, LLM service integration, security implementation
- **Deliverables:** File upload system, LLM-powered org schema extraction

### Phase 2: Advanced Intelligence & Inference

- **AI Tasks:** Industry-specific hierarchy inference, conflict resolution, team structure analysis
- **Human Tasks:** Performance optimization, database integration, error handling
- **Deliverables:** Smart hierarchy generation with industry expertise, confidence scoring

### Phase 3: Interactive Refinement

- **AI Tasks:** Conversational modification system, intent parsing with LLM
- **Human Tasks:** WebSocket implementation, real-time architecture
- **Deliverables:** Interactive schema editing, real-time updates

### Phase 4: Visualization & Export

- **AI Tasks:** Export formatters, visualization components
- **Human Tasks:** UI/UX optimization, performance tuning
- **Deliverables:** Interactive org charts, multiple export formats

## Resource Requirements

### Development Resources (AI-Assisted)

- **Technical Lead:** 0.5 FTE (architecture, AI coordination, LLM integration)
- **Frontend Specialist:** 0.3 FTE (React components, visualization)
- **Integration Engineer:** 0.3 FTE (API integration, database)
- **QA Specialist:** 0.2 FTE (testing, validation)

## Success Metrics

### Functional KPIs

- **Schema Accuracy:** > 90% correct hierarchy inference from files
- **User Approval Rate:** > 95% of sessions result in approved schema
- **Data Completeness:** < 5% missing critical organizational information
- **Refinement Efficiency:** Average 3-5 modifications to reach approval
- **Industry Recognition:** > 85% accuracy in detecting industry-specific patterns

### Performance KPIs

- **Processing Speed:** < 30 seconds for 1000-employee organization
- **Real-time Responsiveness:** < 200ms for schema modifications
- **File Processing Success:** > 99% successful file parsing with LLM
- **System Availability:** > 99.9% uptime
- **LLM Processing:** < 5 seconds for organizational analysis

## Conclusion

The Smart Organization Schema Parser Module seamlessly integrates with your existing infrastructure while providing powerful LLM-driven organizational hierarchy extraction and refinement capabilities. By leveraging comprehensive industry expertise through AI, the system handles diverse organizational structures across all sectors and cultural contexts. The conversational refinement interface ensures user satisfaction while the real-time visualization provides intuitive schema management. This approach eliminates the limitations of rule-based systems while providing sophisticated, context-aware organizational analysis.
