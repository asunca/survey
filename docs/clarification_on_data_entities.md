# Critical Clarifications Needed

## User Types

- Super Admin: Owner of an Agile Transformation Company
- Company Admin: Consultants in an Agile Transformation Company
- Standard Users: Externally registered users

## Business Model

- This company provides agile transformation consulting services.
- Company Admins are the employees/consultants serving multiple client companies.
- Standard Users are external users who can self-serve for smaller needs
- Super Admin manages the internal team and platform

## Hierarchical Structure

### Unified Hierarchical Structure

```
Super Admin
├── Company Admin 1
│   ├── Company A
│   │   ├── Organization 1
│   │   │   ├── Employee 1
│   │   │   │   └── Surveys
│   │   │   └── Employee 2
│   │   │       └── Surveys
│   │   └── Organization 2
│   │       └── Employees...
│   │           └── Surveys
│   └── Company B
│       └── Organizations...
│           └── Employees...
│               └── Surveys
├── Company Admin 2
│   └── Companies...
│       └── Organizations...
│           └── Employees...
│               └── Surveys
├── User 1
│   └── Auto-Created Company 1
│       └── Organizations
│           └── Employees
│               └── Surveys
└── User 2
    └── Auto-Created Company 2
        └── Organizations
            └── Employees
                └── Surveys
```

## Unified Access Pattern

### Single Hierarchical Flow

```
Super Admin
    ├── Company Admin → Company → Organization → Employee → Survey
    └── User → Auto-Created Company → Organization → Employee → Survey
```

## Entity Relationships

| Entity                  | Contains/Manages                                           | Access Level        |
| ----------------------- | ---------------------------------------------------------- | ------------------- |
| Super Admin             | Company Admins, Users                                      | Full System         |
| Company Admin           | Companies → Organizations → Employees → Surveys            | Company Level       |
| User                    | Auto-Created Company → Organizations → Employees → Surveys | User Scope          |
| Company (Admin-managed) | Organizations → Employees → Surveys                        | Company Scope       |
| Auto-Created Company    | Organizations → Employees → Surveys                        | Company Scope       |
| Organization            | Employees → Surveys                                        | Org Scope           |
| Employee                | Survey Responses                                           | Individual          |
| Question                | Question Pool                                              | Content Management  |
| Theme                   | Survey Templates                                           | Template Management |
| Draft                   | Draft Surveys                                              | Draft State         |
| Survey                  | Active Surveys                                             | Live Surveys        |

## Personas

- Super Admin creates two company admins who are responsible for different/common companies which might have multiple organizational schemas.
- Company Admin creates and assigns surveys to different companies, which can be under his/her responsibility, and performs reporting on those companies.
- Company Admin can create different reports on different organizational schemas on the same company.
- Standard User, registers from web interface and create multiple organizational schemas to create surveys, and observe reports on those.

### 1. Company Entity Creation:

Standard Users: When they register, do they automatically create a new Company entity, or do they join existing companies?
Scenario: If two Standard Users from the same real company (e.g., "Acme Corp") register separately, should they:

See each other's organizational schemas?
Be automatically linked to the same Company entity?
Remain completely separate?

### 2. Business Model Boundaries:

- Standard User Limitations: You mentioned "limited to defined number of organization schema creation" - what triggers this limit?

- Per user? Per company? Per time period?

- Monetization: Are Standard Users paying customers or free trial users?
- Graduation Path: Can Standard Users become Company Admin clients?

### 3. Data Isolation Concerns:

- Company Admin Access: Can your Company Admins see Standard User data from companies they don't actively serve?
- Cross-Contamination: How do you prevent competitive companies' data from being accessible to the same Company Admin?

### 4. Organizational Schema Ownership:

- Within Same Company: If Company Admin creates a schema for "Acme Corp" and later a Standard User from Acme Corp registers, who owns what?
- Schema Conflicts: Multiple schemas for the same real organizational unit - how are these managed?

### 5. Survey Assignment Logic:

- Cross-Schema Surveys: Can a survey span multiple organizational schemas within the same company?
- External Recipients: Can Standard Users survey people outside their organizational schema (contractors, partners)?
