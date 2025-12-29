# Backend-Controlled UI Configuration with RBAC/OPA
## Complete Flow Documentation + Database Schema Design

---

## Table of Contents
1. [System Overview](#system-overview)
2. [Database Schema - Hierarchical Structure](#database-schema---hierarchical-structure)
3. [Configuration Management Strategy](#configuration-management-strategy)
4. [Use Case: Compliance Officer Role](#use-case-compliance-officer-role)
5. [Complete Request-Response Flow](#complete-request-response-flow)
6. [OPA Rego Policies](#opa-rego-policies)
7. [Sequence Diagram](#sequence-diagram)

---

## System Overview

### Architecture Components

```
┌─────────────┐      ┌──────────────────┐      ┌─────────────────┐      ┌─────────┐
│   Frontend  │─────>│  UI Config       │─────>│  RBAC/OPA       │─────>│   OPA   │
│   (React)   │<─────│  Service         │<─────│  Service        │<─────│  Engine │
└─────────────┘      └──────────────────┘      └─────────────────┘      └─────────┘
                              │
                              ▼
                     ┌─────────────────┐
                     │  Screen Config  │
                     │  Database       │
                     │  (PostgreSQL)   │
                     └─────────────────┘
```

---

## Database Schema - Hierarchical Structure

### Entity Hierarchy Overview

```
Application
    └── Module
         └── Screen
              ├── Section
              │    ├── Field
              │    └── Component
              ├── Action
              └── Navigation
```

---

### 1. **TABLE: applications**

Represents the top-level application (e.g., CLM System)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique identifier | `app_001` |
| name | VARCHAR(100) | NOT NULL, UNIQUE | Application name | `CLM Application` |
| code | VARCHAR(50) | NOT NULL, UNIQUE | Application code | `CLM_APP` |
| description | TEXT | | Application description | `Compliance Lifecycle Management` |
| version | VARCHAR(20) | NOT NULL | Application version | `2.1.0` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| created_by | VARCHAR(100) | | Creator user ID | `admin@bank.com` |
| metadata | JSONB | | Additional metadata | `{"theme": "dark", "locale": "en-US"}` |

**Sample Data:**
```sql
INSERT INTO applications VALUES 
('app_001', 'CLM Application', 'CLM_APP', 'Compliance Lifecycle Management', '2.1.0', true, NOW(), NOW(), 'admin@bank.com', '{"theme": "corporate", "locale": "en-US"}');
```

---

### 2. **TABLE: modules**

Logical grouping of screens (e.g., Case Management, Reporting)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique identifier | `mod_001` |
| application_id | UUID | FOREIGN KEY → applications(id) | Parent application | `app_001` |
| name | VARCHAR(100) | NOT NULL | Module name | `Case Management` |
| code | VARCHAR(50) | NOT NULL | Module code | `CASE_MGMT` |
| description | TEXT | | Module description | `Manage compliance cases` |
| icon | VARCHAR(50) | | Icon identifier | `briefcase` |
| display_order | INTEGER | DEFAULT 0 | Display order in navigation | `1` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| required_permissions | TEXT[] | | Base permissions required | `{view_cases}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO modules VALUES 
('mod_001', 'app_001', 'Case Management', 'CASE_MGMT', 'Manage compliance cases', 'briefcase', 1, true, ARRAY['view_cases'], NOW(), NOW());
```

---

### 3. **TABLE: screens**

Individual screens/pages in the application

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | VARCHAR(100) | PRIMARY KEY | Unique screen identifier | `case_details_screen` |
| module_id | UUID | FOREIGN KEY → modules(id) | Parent module | `mod_001` |
| name | VARCHAR(100) | NOT NULL | Screen display name | `Case Details` |
| title | VARCHAR(200) | | Screen title | `Case Management Dashboard` |
| route | VARCHAR(200) | | Frontend route | `/cases/:caseId` |
| layout_type | VARCHAR(50) | | Layout template | `two-column` |
| description | TEXT | | Screen description | `Detailed case information view` |
| icon | VARCHAR(50) | | Icon identifier | `file-text` |
| display_order | INTEGER | DEFAULT 0 | Order within module | `2` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| classification | VARCHAR(50) | | Data classification | `confidential` |
| required_roles | TEXT[] | | Minimum roles required | `{viewer}` |
| required_permissions | TEXT[] | | Minimum permissions | `{view_case}` |
| refresh_interval | INTEGER | | Auto-refresh interval (seconds) | `300` |
| cache_ttl | INTEGER | | Cache time-to-live (seconds) | `60` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| metadata | JSONB | | Additional config | `{"showBreadcrumbs": true}` |

**Sample Data:**
```sql
INSERT INTO screens VALUES 
('case_details_screen', 'mod_001', 'Case Details', 'Case Management Dashboard', '/cases/:caseId', 'two-column', 
'Detailed case information view', 'file-text', 2, true, 'confidential', ARRAY['viewer'], ARRAY['view_case'], 
300, 60, NOW(), NOW(), '{"showBreadcrumbs": true, "showHelpButton": true}');
```

---

### 4. **TABLE: sections**

Sections within a screen (grouping of related fields)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | VARCHAR(100) | PRIMARY KEY | Unique section identifier | `case_basic_info` |
| screen_id | VARCHAR(100) | FOREIGN KEY → screens(id) | Parent screen | `case_details_screen` |
| name | VARCHAR(100) | NOT NULL | Section name | `Case Information` |
| label | VARCHAR(200) | NOT NULL | Display label | `Basic Case Information` |
| description | TEXT | | Section description | `Core case details` |
| display_order | INTEGER | DEFAULT 0 | Order within screen | `1` |
| columns | INTEGER | DEFAULT 1 | Number of columns | `2` |
| is_collapsible | BOOLEAN | DEFAULT false | Can be collapsed | `false` |
| is_collapsed_default | BOOLEAN | DEFAULT false | Default collapsed state | `false` |
| is_highlighted | BOOLEAN | DEFAULT false | Highlight section | `false` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| required_permissions | TEXT[] | | Permissions to view section | `{view_case_details}` |
| required_roles | TEXT[] | | Roles to view section | `{viewer}` |
| conditional_display | JSONB | | Conditional display rules | `{"field": "status", "value": "open"}` |
| css_class | VARCHAR(100) | | Custom CSS class | `highlight-section` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO sections VALUES 
('case_basic_info', 'case_details_screen', 'Case Information', 'Basic Case Information', 
'Core case details and status', 1, 2, false, false, false, true, 
ARRAY['view_case_details'], ARRAY['viewer'], NULL, 'standard-section', NOW(), NOW());

INSERT INTO sections VALUES 
('risk_assessment', 'case_details_screen', 'Risk Assessment', 'Risk Analysis', 
'Risk scoring and categorization', 4, 1, true, true, true, true, 
ARRAY['view_risk_assessment'], ARRAY['compliance_officer'], NULL, 'highlight-section', NOW(), NOW());
```

---

### 5. **TABLE: fields**

Individual form fields within sections

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique field identifier | `fld_001` |
| section_id | VARCHAR(100) | FOREIGN KEY → sections(id) | Parent section | `case_basic_info` |
| name | VARCHAR(100) | NOT NULL | Field name (API mapping) | `case_id` |
| label | VARCHAR(200) | NOT NULL | Display label | `Case ID` |
| field_type | VARCHAR(50) | NOT NULL | Input type | `text` |
| data_type | VARCHAR(50) | | Data type | `string` |
| display_order | INTEGER | DEFAULT 0 | Order within section | `1` |
| is_required | BOOLEAN | DEFAULT false | Required field | `false` |
| is_system_field | BOOLEAN | DEFAULT false | System-generated field | `true` |
| is_sensitive | BOOLEAN | DEFAULT false | Contains sensitive data | `false` |
| default_editable | BOOLEAN | DEFAULT true | Default edit permission | `false` |
| default_visible | BOOLEAN | DEFAULT true | Default visibility | `true` |
| placeholder | VARCHAR(200) | | Placeholder text | `Enter case ID` |
| help_text | TEXT | | Help/tooltip text | `System generated identifier` |
| min_length | INTEGER | | Minimum length | `5` |
| max_length | INTEGER | | Maximum length | `20` |
| min_value | DECIMAL | | Minimum value (numbers) | `0` |
| max_value | DECIMAL | | Maximum value (numbers) | `100` |
| pattern | VARCHAR(500) | | Validation regex pattern | `^[A-Z0-9]{10}$` |
| validation_message | TEXT | | Validation error message | `Invalid case ID format` |
| options | JSONB | | Select/radio options | `[{"value": "open", "label": "Open"}]` |
| data_source | VARCHAR(200) | | API endpoint for options | `/api/officers` |
| format | VARCHAR(100) | | Display format | `YYYY-MM-DD` |
| mask_pattern | VARCHAR(100) | | Masking pattern | `XXX-XX-{last4}` |
| conditional_rules | JSONB | | Conditional display/edit rules | `{"showIf": {"field": "status", "equals": "open"}}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| metadata | JSONB | | Additional config | `{"autocomplete": "off"}` |

**Sample Data:**
```sql
INSERT INTO fields VALUES 
('fld_001', 'case_basic_info', 'case_id', 'Case ID', 'text', 'string', 1, false, true, false, false, true, 
NULL, 'System generated case identifier', 10, 10, NULL, NULL, '^CASE[0-9]{6}$', 'Invalid case ID format', 
NULL, NULL, NULL, NULL, NULL, NOW(), NOW(), '{"readonly": true, "copyable": true}');

INSERT INTO fields VALUES 
('fld_002', 'case_basic_info', 'customer_ssn', 'Social Security Number', 'text', 'string', 5, false, false, true, false, true, 
NULL, 'Customer SSN', 11, 11, NULL, NULL, '^[0-9]{3}-[0-9]{2}-[0-9]{4}$', 'Invalid SSN format', 
NULL, NULL, NULL, 'XXX-XX-{last4}', NULL, NOW(), NOW(), '{"sensitive": true, "encrypted": true}');

INSERT INTO fields VALUES 
('fld_003', 'risk_assessment', 'risk_score', 'Risk Score', 'number', 'integer', 1, true, false, false, true, true, 
NULL, 'Enter risk score between 0-100', NULL, NULL, 0, 100, NULL, 'Risk score must be between 0 and 100', 
NULL, NULL, NULL, NULL, '{"editableBy": ["compliance_officer"]}', NOW(), NOW(), '{"step": 1, "showSlider": true}');
```

---

### 6. **TABLE: components**

Special UI components (tables, charts, file uploads, etc.)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique component identifier | `comp_001` |
| section_id | VARCHAR(100) | FOREIGN KEY → sections(id) | Parent section | `audit_trail` |
| component_type | VARCHAR(50) | NOT NULL | Component type | `audit_log_table` |
| name | VARCHAR(100) | NOT NULL | Component name | `Audit Log` |
| display_order | INTEGER | DEFAULT 0 | Order within section | `1` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| config | JSONB | NOT NULL | Component configuration | `{"endpoint": "/api/audit", "columns": [...]}` |
| required_permissions | TEXT[] | | Permissions to view | `{view_audit_trail}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO components VALUES 
('comp_001', 'audit_trail', 'audit_log_table', 'Audit Log', 1, true, 
'{
  "endpoint": "/api/cases/{caseId}/audit",
  "columns": ["timestamp", "user", "action", "changes"],
  "pageSize": 10,
  "sortable": true,
  "filterable": true
}', 
ARRAY['view_audit_trail'], NOW(), NOW());

INSERT INTO components VALUES 
('comp_002', 'documents_section', 'file_upload', 'Document Upload', 1, true, 
'{
  "endpoint": "/api/cases/{caseId}/documents",
  "acceptedFormats": ["pdf", "doc", "docx"],
  "maxSize": "10MB",
  "multiple": true,
  "maxFiles": 5
}', 
ARRAY['upload_documents'], NOW(), NOW());
```

---

### 7. **TABLE: actions**

Actions/buttons available on a screen

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | VARCHAR(100) | PRIMARY KEY | Unique action identifier | `approve_case` |
| screen_id | VARCHAR(100) | FOREIGN KEY → screens(id) | Parent screen | `case_details_screen` |
| name | VARCHAR(100) | NOT NULL | Action name | `Approve Case` |
| label | VARCHAR(200) | NOT NULL | Display label | `Approve` |
| action_type | VARCHAR(50) | NOT NULL | Action type | `api` |
| button_type | VARCHAR(50) | DEFAULT 'secondary' | Button style | `success` |
| icon | VARCHAR(50) | | Icon identifier | `check` |
| display_order | INTEGER | DEFAULT 0 | Display order | `2` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| confirmation_required | BOOLEAN | DEFAULT false | Requires confirmation | `true` |
| confirmation_message | TEXT | | Confirmation dialog text | `Are you sure you want to approve?` |
| endpoint | VARCHAR(500) | | API endpoint | `/api/cases/{caseId}/approve` |
| http_method | VARCHAR(10) | | HTTP method | `POST` |
| route | VARCHAR(200) | | Navigation route | `/cases/{caseId}/edit` |
| modal_id | VARCHAR(100) | | Modal identifier to open | `assign_case_modal` |
| required_permissions | TEXT[] | | Permissions to see action | `{approve_case}` |
| required_roles | TEXT[] | | Roles to see action | `{compliance_officer}` |
| conditional_display | JSONB | | Conditional display rules | `{"field": "status", "notEquals": "closed"}` |
| action_config | JSONB | | Additional action config | `{"format": "pdf", "fileName": "case_report"}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO actions VALUES 
('approve_case', 'case_details_screen', 'Approve Case', 'Approve', 'api', 'success', 'check', 2, true, 
true, 'Are you sure you want to approve this case?', '/api/cases/{caseId}/approve', 'POST', NULL, NULL, 
ARRAY['approve_case'], ARRAY['compliance_officer'], 
'{"field": "status", "in": ["pending_review", "in_progress"]}', 
'{"successMessage": "Case approved successfully", "redirectAfter": true}', NOW(), NOW());

INSERT INTO actions VALUES 
('export_report', 'case_details_screen', 'Export Report', 'Export Report', 'download', 'secondary', 'download', 5, true, 
false, NULL, '/api/cases/{caseId}/export', 'GET', NULL, NULL, 
ARRAY['export_report'], ARRAY['compliance_officer', 'auditor'], NULL, 
'{"format": "pdf", "fileName": "case_{caseId}_report.pdf"}', NOW(), NOW());
```

---

### 8. **TABLE: navigation**

Navigation elements (breadcrumbs, related links, etc.)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique navigation identifier | `nav_001` |
| screen_id | VARCHAR(100) | FOREIGN KEY → screens(id) | Parent screen | `case_details_screen` |
| navigation_type | VARCHAR(50) | NOT NULL | Navigation type | `breadcrumb` |
| label | VARCHAR(200) | NOT NULL | Display label | `Case Details` |
| route | VARCHAR(200) | | Navigation route | `/cases/{caseId}` |
| display_order | INTEGER | DEFAULT 0 | Display order | `3` |
| icon | VARCHAR(50) | | Icon identifier | `home` |
| is_active | BOOLEAN | DEFAULT true | Active status | `true` |
| required_permissions | TEXT[] | | Permissions to view | `{view_cases}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO navigation VALUES 
('nav_001', 'case_details_screen', 'breadcrumb', 'Home', '/', 1, 'home', true, NULL, NOW(), NOW());

INSERT INTO navigation VALUES 
('nav_002', 'case_details_screen', 'breadcrumb', 'Cases', '/cases', 2, 'briefcase', true, ARRAY['view_cases'], NOW(), NOW());

INSERT INTO navigation VALUES 
('nav_003', 'case_details_screen', 'breadcrumb', 'Case Details', '/cases/{caseId}', 3, 'file-text', true, ARRAY['view_case'], NOW(), NOW());

INSERT INTO navigation VALUES 
('nav_004', 'case_details_screen', 'related_link', 'View All Cases', '/cases', 1, 'list', true, ARRAY['view_cases'], NOW(), NOW());
```

---

### 9. **TABLE: field_permissions**

Granular field-level permissions (optional - can be managed in OPA)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique permission identifier | `perm_001` |
| field_id | UUID | FOREIGN KEY → fields(id) | Target field | `fld_002` |
| role | VARCHAR(100) | NOT NULL | Role name | `compliance_officer` |
| can_view | BOOLEAN | DEFAULT false | View permission | `true` |
| can_edit | BOOLEAN | DEFAULT false | Edit permission | `true` |
| requires_masking | BOOLEAN | DEFAULT false | Apply masking | `false` |
| mask_pattern | VARCHAR(100) | | Masking pattern override | `XXX-XX-{last4}` |
| conditions | JSONB | | Additional conditions | `{"clearanceLevel": "L3"}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
-- Compliance officer can view/edit SSN without masking
INSERT INTO field_permissions VALUES 
('perm_001', 'fld_002', 'compliance_officer', true, false, false, NULL, NULL, NOW(), NOW());

-- Analyst can view SSN but with masking
INSERT INTO field_permissions VALUES 
('perm_002', 'fld_002', 'analyst', true, false, true, 'XXX-XX-{last4}', NULL, NOW(), NOW());

-- Viewer cannot see SSN at all
INSERT INTO field_permissions VALUES 
('perm_003', 'fld_002', 'viewer', false, false, true, NULL, NULL, NOW(), NOW());
```

---

### 10. **TABLE: screen_versions**

Version history for screens (audit trail)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique version identifier | `ver_001` |
| screen_id | VARCHAR(100) | FOREIGN KEY → screens(id) | Target screen | `case_details_screen` |
| version | VARCHAR(20) | NOT NULL | Version number | `2.1.0` |
| change_description | TEXT | | Description of changes | `Added risk assessment section` |
| configuration_snapshot | JSONB | NOT NULL | Full config snapshot | `{...}` |
| created_by | VARCHAR(100) | NOT NULL | User who made changes | `admin@bank.com` |
| created_at | TIMESTAMP | DEFAULT NOW() | Version creation timestamp | `2025-12-27 10:00:00` |
| is_active | BOOLEAN | DEFAULT true | Active version | `true` |

**Sample Data:**
```sql
INSERT INTO screen_versions VALUES 
('ver_001', 'case_details_screen', '2.1.0', 'Added risk assessment section with edit capabilities', 
'{
  "sections": [...],
  "fields": [...],
  "actions": [...]
}', 
'admin@bank.com', NOW(), true);
```

---

## Configuration Management Strategy

### **Grouping Logic Overview**

```
┌──────────────────────────────────────────────────────────┐
│                      APPLICATION                          │
│                   (CLM Application)                       │
└────────────────────────┬─────────────────────────────────┘
                         │
          ┌──────────────┼──────────────┐
          │              │              │
    ┌─────▼─────┐  ┌─────▼─────┐  ┌────▼────┐
    │  Module 1 │  │  Module 2 │  │ Module 3│
    │   Case    │  │ Reporting │  │  Admin  │
    │   Mgmt    │  │           │  │         │
    └─────┬─────┘  └───────────┘  └─────────┘
          │
    ┌─────┼─────┬─────────┬─────────┐
    │     │     │         │         │
┌───▼──┐ ┌▼───┐ ┌▼──────┐ ┌▼──────┐ 
│Screen│ │Scrn│ │Screen │ │Screen │
│  1   │ │ 2  │ │   3   │ │   4   │
│ List │ │Dtl │ │Create │ │Search │
└───┬──┘ └┬───┘ └───────┘ └───────┘
    │     │
    │     └─────┬────────┬────────┬─────────┐
    │           │        │        │         │
    │      ┌────▼──┐ ┌───▼───┐ ┌─▼─────┐ ┌─▼──────┐
    │      │Section│ │Section│ │Section│ │Actions │
    │      │   1   │ │   2   │ │   3   │ │  Bar   │
    │      │ Basic │ │ Risk  │ │ Audit │ └────────┘
    │      └───┬───┘ └───┬───┘ └───┬───┘
    │          │         │         │
    │      ┌───┴──┬──────┼─────┐   └──────┐
    │      │      │      │     │          │
    │   ┌──▼─┐ ┌─▼──┐ ┌─▼──┐ ┌▼────┐ ┌───▼──────┐
    │   │Fld1│ │Fld2│ │Fld3│ │Fld4 │ │Component │
    │   │ID  │ │Name│ │SSN │ │Risk │ │AuditTable│
    │   └────┘ └────┘ └────┘ └─────┘ └──────────┘
```

### **Data Retrieval Strategy**

#### **Single Query Approach (Recommended for Performance)**

```sql
-- Get complete screen configuration in one query
WITH screen_data AS (
    SELECT s.* FROM screens s WHERE s.id = 'case_details_screen'
),
sections_data AS (
    SELECT sec.* FROM sections sec 
    WHERE sec.screen_id = 'case_details_screen' 
    AND sec.is_active = true 
    ORDER BY sec.display_order
),
fields_data AS (
    SELECT f.* FROM fields f
    INNER JOIN sections sec ON f.section_id = sec.id
    WHERE sec.screen_id = 'case_details_screen'
    ORDER BY f.display_order
),
components_data AS (
    SELECT c.* FROM components c
    INNER JOIN sections sec ON c.section_id = sec.id
    WHERE sec.screen_id = 'case_details_screen'
    AND c.is_active = true
),
actions_data AS (
    SELECT a.* FROM actions a
    WHERE a.screen_id = 'case_details_screen'
    AND a.is_active = true
    ORDER BY a.display_order
),
navigation_data AS (
    SELECT n.* FROM navigation n
    WHERE n.screen_id = 'case_details_screen'
    AND n.is_active = true
    ORDER BY n.navigation_type, n.display_order
)
SELECT 
    json_build_object(
        'screen', (SELECT row_to_json(screen_data.*) FROM screen_data),
        'sections', (SELECT json_agg(row_to_json(sections_data.*)) FROM sections_data),
        'fields', (SELECT json_agg(row_to_json(fields_data.*)) FROM fields_data),
        'components', (SELECT json_agg(row_to_json(components_data.*)) FROM components_data),
        'actions', (SELECT json_agg(row_to_json(actions_data.*)) FROM actions_data),
        'navigation', (SELECT json_agg(row_to_json(navigation_data.*)) FROM navigation_data)
    ) as complete_config;
```

### **Caching Strategy**

| Cache Level | TTL | Key Pattern | Description |
|-------------|-----|-------------|-------------|
| Screen Base Config | 1 hour | `screen:config:{screenId}` | Raw screen configuration from DB |
| User Permission | 15 minutes | `user:perm:{userId}:{screenId}` | OPA evaluation results for user |
| Final UI Config | 5 minutes | `ui:config:{userId}:{screenId}` | Complete filtered UI config |
| Field Options | 30 minutes | `field:options:{fieldId}` | Dropdown/select options |

---

## Use Case: Compliance Officer Role

### Scenario
**User**: Sarah Johnson  
**Role**: Compliance Officer  
**Department**: Regulatory Compliance  
**Access Level**: Senior  

**Requirements**:
- Can view all case details including sensitive financial information
- Can view and edit risk assessment scores
- Can approve/reject cases
- Can view audit trails
- Cannot delete cases (only senior management)
- Cannot modify historical records

**Screen**: Case Management Dashboard - `case_details_screen`

---

## Complete Request-Response Flow

### Step 1: Frontend Request (After SSO Login)

**Endpoint**: `POST /api/ui/config`

**Headers**:
```http
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**JWT Token Payload** (decoded):
```json
{
  "sub": "sarah.johnson@bank.com",
  "userId": "emp_12345",
  "roles": ["compliance_officer", "senior_staff"],
  "department": "regulatory_compliance",
  "region": "north_america",
  "iat": 1703678400,
  "exp": 1703764800
}
```

**Request Body**:
```json
{
  "requestId": "req_abc123",
  "user": {
    "userId": "emp_12345",
    "email": "sarah.johnson@bank.com",
    "roles": ["compliance_officer", "senior_staff"],
    "department": "regulatory_compliance",
    "region": "north_america",
    "attributes": {
      "clearanceLevel": "L3",
      "yearsOfService": 8
    }
  },
  "resource": {
    "type": "screen",
    "id": "case_details_screen",
    "classification": "confidential"
  },
  "sections": [
    "case_basic_info",
    "customer_details",
    "financial_information",
    "risk_assessment",
    "audit_trail",
    "case_actions"
  ],
  "fields": [
    "case_id",
    "case_status",
    "customer_name",
    "customer_ssn",
    "customer_dob",
    "account_number",
    "account_balance",
    "transaction_amount",
    "risk_score",
    "risk_category",
    "assigned_officer",
    "created_date",
    "last_modified_date",
    "notes",
    "supporting_documents"
  ],
  "actions": [
    "view_case",
    "edit_case",
    "create_case",
    "delete_case",
    "approve_case",
    "reject_case",
    "export_report",
    "assign_case",
    "add_notes",
    "upload_documents"
  ],
  "evaluationContext": {
    "timestamp": "2025-12-27T10:00:00Z",
    "sessionId": "sess_xyz789"
  }
}
```

---

### Step 3: OPA Rego Policies

#### Policy File: `ui_permissions.rego`

```rego
package ui.permissions

import future.keywords.if
import future.keywords.in

# Default deny all
default allow_section = false
default allow_field_view = false
default allow_field_edit = false
default allow_action = false

##############################################
# SECTION VISIBILITY RULES
##############################################

# Everyone can see basic case info
allow_section if {
    input.section == "case_basic_info"
}

# Compliance officers can see customer details
allow_section if {
    input.section == "customer_details"
    has_role(input.user, "compliance_officer")
}

# Compliance officers can see financial info
allow_section if {
    input.section == "financial_information"
    has_role(input.user, "compliance_officer")
}

# Only compliance officers can see risk assessment
allow_section if {
    input.section == "risk_assessment"
    has_role(input.user, "compliance_officer")
}

# Compliance officers can see audit trail
allow_section if {
    input.section == "audit_trail"
    has_role(input.user, "compliance_officer")
}

##############################################
# FIELD VISIBILITY RULES
##############################################

# Public fields - everyone can view
allow_field_view if {
    input.field in public_fields
}

# Sensitive PII - only compliance officers
allow_field_view if {
    input.field in sensitive_pii_fields
    has_role(input.user, "compliance_officer")
}

# Financial data - compliance officers
allow_field_view if {
    input.field in financial_fields
    has_role(input.user, "compliance_officer")
}

# Risk assessment - only compliance officers
allow_field_view if {
    input.field in risk_fields
    has_role(input.user, "compliance_officer")
}

##############################################
# FIELD EDIT RULES
##############################################

# Compliance officers can edit risk scores
allow_field_edit if {
    input.field in ["risk_score", "risk_category"]
    has_role(input.user, "compliance_officer")
}

# Compliance officers can edit notes
allow_field_edit if {
    input.field == "notes"
    has_role(input.user, "compliance_officer")
}

##############################################
# ACTION RULES
##############################################

# Approve/Reject case - compliance officers only
allow_action if {
    input.action in ["approve_case", "reject_case"]
    has_role(input.user, "compliance_officer")
}

# Delete case - only senior management
allow_action if {
    input.action == "delete_case"
    has_role(input.user, "senior_management")
}

##############################################
# HELPER FUNCTIONS
##############################################

has_role(user, role) if {
    user.roles[_] == role
}

public_fields := {
    "case_id",
    "case_status",
    "created_date",
    "assigned_officer"
}

sensitive_pii_fields := {
    "customer_ssn",
    "customer_dob"
}

financial_fields := {
    "account_number",
    "account_balance",
    "transaction_amount"
}

risk_fields := {
    "risk_score",
    "risk_category"
}
```

---

### Step 4: Response from RBAC/OPA Service → UI Config Service

```json
{
  "requestId": "req_abc123",
  "evaluationId": "eval_def456",
  "userId": "emp_12345",
  "evaluatedAt": "2025-12-27T10:00:01.234Z",
  "sectionPermissions": {
    "case_basic_info": {"allowed": true},
    "customer_details": {"allowed": true},
    "financial_information": {"allowed": true},
    "risk_assessment": {"allowed": true},
    "audit_trail": {"allowed": true}
  },
  "fieldPermissions": {
    "case_id": {"canView": true, "canEdit": false},
    "customer_ssn": {"canView": true, "canEdit": false},
    "risk_score": {"canView": true, "canEdit": true},
    "risk_category": {"canView": true, "canEdit": true}
  },
  "actionPermissions": {
    "approve_case": {"allowed": true},
    "reject_case": {"allowed": true},
    "delete_case": {"allowed": false}
  }
}
```

---

### Step 5: Response from UI Config Service → Frontend

```json
{
  "screenId": "case_details_screen",
  "title": "Case Management Dashboard",
  "sections": [
    {
      "id": "case_basic_info",
      "label": "Case Information",
      "visible": true,
      "fields": [
        {
          "name": "case_id",
          "label": "Case ID",
          "type": "text",
          "visible": true,
          "editable": false
        }
      ]
    },
    {
      "id": "risk_assessment",
      "label": "Risk Assessment",
      "visible": true,
      "fields": [
        {
          "name": "risk_score",
          "label": "Risk Score",
          "type": "number",
          "visible": true,
          "editable": true
        }
      ]
    }
  ],
  "actions": [
    {
      "id": "approve_case",
      "label": "Approve",
      "visible": true
    },
    {
      "id": "reject_case",
      "label": "Reject",
      "visible": true
    }
  ]
}
```

---

## Sequence Diagram

```
┌─────────┐     ┌─────────────┐     ┌─────────────┐     ┌─────────┐
│Frontend │     │ UI Config   │     │  RBAC/OPA   │     │   OPA   │
│         │     │  Service    │     │  Service    │     │  Engine │
└────┬────┘     └──────┬──────┘     └──────┬──────┘     └────┬────┘
     │                 │                    │                  │
     │ 1. POST /config │                    │                  │
     ├────────────────>│                    │                  │
     │                 │                    │                  │
     │                 │ 2. Query DB for    │                  │
     │                 │    Screen Config   │                  │
     │                 │                    │                  │
     │                 │ 3. POST /permissions│                 │
     │                 ├───────────────────>│                  │
     │                 │                    │                  │
     │                 │                    │ 4. Evaluate OPA  │
     │                 │                    ├─────────────────>│
     │                 │                    │                  │
     │                 │                    │ 5. Policy Result │
     │                 │                    │<─────────────────┤
     │                 │                    │                  │
     │                 │ 6. Permissions     │                  │
     │                 │<───────────────────┤                  │
     │                 │                    │                  │
     │                 │ 7. Filter Config   │                  │
     │                 │    Based on Perms  │                  │
     │                 │                    │                  │
     │ 8. UI Config    │                    │                  │
     │<────────────────┤                    │                  │
     │                 │                    │                  │
     │ 9. Render UI    │                    │                  │
     │                 │                    │                  │
```

---

---

## Data Security & Field-Level Filtering

### **CRITICAL SECURITY PRINCIPLE**
> **Never send data to frontend that the user cannot see!**
> 
> Even if a field is hidden in the UI, if data is sent to frontend, it can be accessed via browser DevTools. For banking compliance, this is a **SEVERE SECURITY VIOLATION**.

---

### **The Problem**

```
❌ WRONG APPROACH:
Backend sends full case data → Frontend filters UI → Security Risk!

{
  "case_id": "CASE123456",
  "customer_ssn": "123-45-6789",  ← Sent but hidden in UI
  "account_balance": 50000,        ← Still accessible in DevTools
  "risk_score": 85                 ← User can see in Network tab
}

✅ CORRECT APPROACH:
Backend filters data before sending → Frontend receives only allowed data

{
  "case_id": "CASE123456"
  // SSN, balance, risk_score NOT sent at all
}
```

---

### **Solution Architecture**

```
┌──────────────────────────────────────────────────────────────┐
│                    DATA FLOW ARCHITECTURE                     │
└──────────────────────────────────────────────────────────────┘

Frontend                                              Backend
   │                                                     │
   │ 1. GET /api/ui/config?screen=case_details          │
   ├────────────────────────────────────────────────────>│
   │                                                     │
   │ ← Returns: UI Structure (what fields to show)      │
   │<────────────────────────────────────────────────────┤
   │                                                     │
   │ 2. GET /api/cases/123 + Authorization Header       │
   ├────────────────────────────────────────────────────>│
   │                                                     ├─→ Extract user from JWT
   │                                                     ├─→ Call RBAC/OPA for data permissions
   │                                                     ├─→ Query full case data
   │                                                     ├─→ FILTER data based on permissions
   │                                                     │
   │ ← Returns: FILTERED Data (only allowed fields)     │
   │<────────────────────────────────────────────────────┤
   │                                                     │
   │ 3. Render UI with filtered data                    │
   │                                                     │
```

---

### **Implementation Strategy**

#### **Option 1: Generic Data Filtering Service (Recommended)**

Create a reusable service that filters ANY API response based on user permissions.

**New Table: `api_endpoints`**

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique identifier | `endpoint_001` |
| path | VARCHAR(200) | NOT NULL, UNIQUE | API path | `/api/cases/{caseId}` |
| method | VARCHAR(10) | NOT NULL | HTTP method | `GET` |
| resource_type | VARCHAR(100) | NOT NULL | Resource type for OPA | `case` |
| response_fields | JSONB | NOT NULL | Field mapping config | `{"case_id": "public", "customer_ssn": "sensitive_pii"}` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO api_endpoints VALUES 
('endpoint_001', '/api/cases/{caseId}', 'GET', 'case', 
'{
  "case_id": {"classification": "public"},
  "case_status": {"classification": "public"},
  "customer_name": {"classification": "basic"},
  "customer_ssn": {"classification": "sensitive_pii"},
  "customer_dob": {"classification": "sensitive_pii"},
  "account_number": {"classification": "financial"},
  "account_balance": {"classification": "financial"},
  "risk_score": {"classification": "risk_assessment"},
  "risk_category": {"classification": "risk_assessment"},
  "notes": {"classification": "basic"}
}', 
NOW(), NOW());
```

---

#### **Java Implementation - Data Filter Service**

```java
@Service
public class DataFilterService {
    
    @Autowired
    private RBACClient rbacClient;
    
    @Autowired
    private ApiEndpointRepository endpointRepository;
    
    /**
     * Filter any response object based on user permissions
     * This is called by an interceptor after the API response is generated
     */
    public Object filterResponse(
            Object responseData, 
            String endpoint, 
            String method,
            UserContext user) {
        
        // 1. Get endpoint configuration
        ApiEndpoint endpointConfig = endpointRepository
            .findByPathAndMethod(endpoint, method)
            .orElse(null);
        
        if (endpointConfig == null) {
            // No filtering configured, return as-is (or throw error for sensitive APIs)
            log.warn("No filtering config for endpoint: {} {}", method, endpoint);
            return responseData;
        }
        
        // 2. Get field permissions from RBAC/OPA
        Map<String, FieldPermission> fieldPermissions = getFieldPermissions(
            user, 
            endpointConfig.getResourceType(),
            endpointConfig.getResponseFields().keySet()
        );
        
        // 3. Filter the response
        return filterObject(responseData, fieldPermissions);
    }
    
    private Map<String, FieldPermission> getFieldPermissions(
            UserContext user, 
            String resourceType,
            Set<String> fieldNames) {
        
        // Call RBAC service to get permissions for these fields
        FieldPermissionRequest request = FieldPermissionRequest.builder()
            .user(User.from(user))
            .resourceType(resourceType)
            .fields(new ArrayList<>(fieldNames))
            .build();
        
        FieldPermissionResponse response = rbacClient.getFieldPermissions(request);
        return response.getFieldPermissions();
    }
    
    @SuppressWarnings("unchecked")
    private Object filterObject(Object data, Map<String, FieldPermission> permissions) {
        if (data == null) {
            return null;
        }
        
        if (data instanceof Map) {
            Map<String, Object> map = (Map<String, Object>) data;
            Map<String, Object> filtered = new LinkedHashMap<>();
            
            for (Map.Entry<String, Object> entry : map.entrySet()) {
                String fieldName = entry.getKey();
                FieldPermission perm = permissions.get(fieldName);
                
                // Only include field if user has view permission
                if (perm != null && perm.isCanView()) {
                    Object value = entry.getValue();
                    
                    // Apply masking if required
                    if (perm.isMasked() && value != null) {
                        value = applyMasking(value.toString(), perm.getMaskingPattern());
                    }
                    
                    filtered.put(fieldName, value);
                }
                // If no permission, field is completely omitted
            }
            
            return filtered;
        }
        
        if (data instanceof List) {
            List<Object> list = (List<Object>) data;
            return list.stream()
                .map(item -> filterObject(item, permissions))
                .collect(Collectors.toList());
        }
        
        return data;
    }
    
    private String applyMasking(String value, String pattern) {
        if (pattern == null || value == null) {
            return value;
        }
        
        // Example: "XXX-XX-{last4}" for SSN
        if (pattern.contains("{last4}")) {
            if (value.length() >= 4) {
                String last4 = value.substring(value.length() - 4);
                return pattern.replace("{last4}", last4);
            }
        }
        
        // Add more masking patterns as needed
        return "***MASKED***";
    }
}
```

---

#### **Response Filter Interceptor**

```java
@Component
public class DataFilterInterceptor implements HandlerInterceptor {
    
    @Autowired
    private DataFilterService dataFilterService;
    
    @Override
    public void postHandle(
            HttpServletRequest request,
            HttpServletResponse response,
            Object handler,
            ModelAndView modelAndView) throws Exception {
        
        // This interceptor wraps the response and filters it
        // Implementation depends on your framework
    }
}

// Better approach: Use ResponseBodyAdvice
@ControllerAdvice
public class DataFilterResponseAdvice implements ResponseBodyAdvice<Object> {
    
    @Autowired
    private DataFilterService dataFilterService;
    
    @Override
    public boolean supports(
            MethodParameter returnType, 
            Class<? extends HttpMessageConverter<?>> converterType) {
        // Apply to all REST controllers
        return true;
    }
    
    @Override
    public Object beforeBodyWrite(
            Object body,
            MethodParameter returnType,
            MediaType selectedContentType,
            Class<? extends HttpMessageConverter<?>> selectedConverterType,
            ServerHttpRequest request,
            ServerHttpResponse response) {
        
        // Extract user from SecurityContext
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        if (auth == null) {
            return body;
        }
        
        UserContext user = (UserContext) auth.getPrincipal();
        String endpoint = request.getURI().getPath();
        String method = request.getMethod().toString();
        
        // Filter the response body
        return dataFilterService.filterResponse(body, endpoint, method, user);
    }
}
```

---

#### **Updated OPA Policies for Data Access**

```rego
package data.permissions

import future.keywords.if
import future.keywords.in

# Default deny
default allow_field_data = false

##############################################
# DATA ACCESS RULES (for API responses)
##############################################

# Public fields - everyone can access
allow_field_data if {
    input.field in public_fields
}

# Basic fields - authenticated users
allow_field_data if {
    input.field in basic_fields
    input.user.roles[_] != "guest"
}

# Sensitive PII - compliance officers and senior staff
allow_field_data if {
    input.field in sensitive_pii_fields
    has_role(input.user, "compliance_officer")
}

allow_field_data if {
    input.field in sensitive_pii_fields
    has_role(input.user, "senior_staff")
    input.user.attributes.clearanceLevel in ["L2", "L3"]
}

# Financial data - compliance officers and financial analysts
allow_field_data if {
    input.field in financial_fields
    has_role(input.user, "compliance_officer")
}

allow_field_data if {
    input.field in financial_fields
    has_role(input.user, "financial_analyst")
    input.user.attributes.clearanceLevel == "L3"
}

# Risk assessment data - only compliance officers
allow_field_data if {
    input.field in risk_fields
    has_role(input.user, "compliance_officer")
}

##############################################
# DATA MASKING RULES
##############################################

# SSN masking
mask_field_data[field] = pattern if {
    field := input.field
    field == "customer_ssn"
    not has_role(input.user, "compliance_officer")
    pattern := "XXX-XX-{last4}"
}

# Account number masking
mask_field_data[field] = pattern if {
    field := input.field
    field == "account_number"
    has_role(input.user, "viewer")
    pattern := "****-****-****-{last4}"
}

##############################################
# HELPER FUNCTIONS
##############################################

has_role(user, role) if {
    user.roles[_] == role
}

public_fields := {
    "case_id",
    "case_status",
    "created_date"
}

basic_fields := {
    "customer_name",
    "assigned_officer",
    "notes"
}

sensitive_pii_fields := {
    "customer_ssn",
    "customer_dob",
    "customer_email",
    "customer_phone"
}

financial_fields := {
    "account_number",
    "account_balance",
    "transaction_amount",
    "transaction_history"
}

risk_fields := {
    "risk_score",
    "risk_category",
    "risk_indicators"
}
```

---

### **Complete Flow Example**

#### **Scenario: Analyst (Limited Access) vs Compliance Officer (Full Access)**

**1. Analyst Request**

```http
GET /api/cases/CASE123456
Authorization: Bearer <analyst_jwt_token>
```

**Backend Processing:**
```java
// 1. CaseController returns full case data
Case fullCase = caseService.getCase("CASE123456");

// 2. ResponseBodyAdvice intercepts the response
// 3. Extract user from JWT: role = "analyst"
// 4. Call RBAC/OPA to check field permissions
// 5. Filter response based on permissions
```

**RBAC/OPA Evaluation:**
```json
{
  "case_id": true,        // Can view
  "customer_name": true,  // Can view
  "customer_ssn": false,  // CANNOT view
  "account_balance": false, // CANNOT view
  "risk_score": false     // CANNOT view
}
```

**Response to Frontend (Analyst):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "assigned_officer": "Sarah Johnson",
  "created_date": "2025-12-01T10:00:00Z",
  "notes": "Initial review completed"
  // customer_ssn: NOT SENT AT ALL
  // account_balance: NOT SENT AT ALL
  // risk_score: NOT SENT AT ALL
}
```

---

**2. Compliance Officer Request**

```http
GET /api/cases/CASE123456
Authorization: Bearer <compliance_officer_jwt_token>
```

**RBAC/OPA Evaluation:**
```json
{
  "case_id": true,
  "customer_name": true,
  "customer_ssn": true,      // CAN view
  "account_balance": true,   // CAN view
  "risk_score": true         // CAN view
}
```

**Response to Frontend (Compliance Officer):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "123-45-6789",  // FULL DATA
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,    // FULL DATA
  "transaction_amount": 15000.00,
  "risk_score": 85,               // FULL DATA
  "risk_category": "high",
  "assigned_officer": "Sarah Johnson",
  "created_date": "2025-12-01T10:00:00Z",
  "notes": "Initial review completed"
}
```

---

### **Alternative: Field-Level API Endpoints**

If you want more explicit control, create separate endpoints:

```java
@RestController
@RequestMapping("/api/cases")
public class CaseController {
    
    @Autowired
    private CaseService caseService;
    
    @Autowired
    private DataFilterService filterService;
    
    // Standard endpoint - returns filtered data automatically
    @GetMapping("/{caseId}")
    public ResponseEntity<Map<String, Object>> getCaseDetails(
            @PathVariable String caseId,
            @AuthenticationPrincipal UserDetails userDetails) {
        
        // Get full case data
        Case fullCase = caseService.getCase(caseId);
        
        // Convert to map
        Map<String, Object> caseData = objectMapper.convertValue(
            fullCase, 
            new TypeReference<Map<String, Object>>() {}
        );
        
        // Filter will be applied by ResponseBodyAdvice automatically
        return ResponseEntity.ok(caseData);
    }
    
    // Explicit field-specific endpoints (optional, for granular control)
    @GetMapping("/{caseId}/basic-info")
    @RequiresPermission("view_case_basic_info")
    public ResponseEntity<CaseBasicInfo> getBasicInfo(@PathVariable String caseId) {
        return ResponseEntity.ok(caseService.getBasicInfo(caseId));
    }
    
    @GetMapping("/{caseId}/financial-info")
    @RequiresPermission("view_financial_info")
    public ResponseEntity<FinancialInfo> getFinancialInfo(@PathVariable String caseId) {
        return ResponseEntity.ok(caseService.getFinancialInfo(caseId));
    }
    
    @GetMapping("/{caseId}/risk-assessment")
    @RequiresPermission("view_risk_assessment")
    public ResponseEntity<RiskAssessment> getRiskAssessment(@PathVariable String caseId) {
        return ResponseEntity.ok(caseService.getRiskAssessment(caseId));
    }
}
```

---

### **Frontend Implementation**

```javascript
// Frontend makes TWO separate calls

// 1. Get UI Configuration (what to show)
const uiConfig = await fetch('/api/ui/config?screen=case_details', {
  headers: { 'Authorization': `Bearer ${token}` }
});

// 2. Get Case Data (filtered by backend)
const caseData = await fetch('/api/cases/CASE123456', {
  headers: { 'Authorization': `Bearer ${token}` }
});

// 3. Render UI with filtered data
function renderScreen(config, data) {
  return config.sections.map(section => {
    return section.fields.map(field => {
      const value = data[field.name]; // Will be undefined if not allowed
      
      if (value === undefined) {
        return null; // Field not in data = user can't see it
      }
      
      return <Field {...field} value={value} />;
    });
  });
}
```

---

### **Security Audit Log**

**New Table: `data_access_log`**

| Column | Type | Description |
|--------|------|-------------|
| id | UUID | Unique log ID |
| user_id | VARCHAR(100) | User who accessed data |
| resource_type | VARCHAR(100) | Type of resource (case, customer, etc.) |
| resource_id | VARCHAR(100) | Specific resource ID |
| fields_accessed | TEXT[] | List of fields accessed |
| fields_denied | TEXT[] | List of fields denied |
| access_timestamp | TIMESTAMP | When access occurred |
| ip_address | VARCHAR(50) | User's IP address |
| session_id | VARCHAR(200) | Session identifier |

```java
@Aspect
@Component
public class DataAccessAuditAspect {
    
    @Autowired
    private DataAccessLogRepository auditLogRepository;
    
    @AfterReturning(
        pointcut = "execution(* com.bank.api..*Controller.*(..))",
        returning = "result"
    )
    public void logDataAccess(JoinPoint joinPoint, Object result) {
        // Log what data was actually returned to the user
        Authentication auth = SecurityContextHolder.getContext().getAuthentication();
        
        if (auth != null && result != null) {
            UserContext user = (UserContext) auth.getPrincipal();
            
            DataAccessLog log = DataAccessLog.builder()
                .userId(user.getUserId())
                .resourceType(extractResourceType(joinPoint))
                .resourceId(extractResourceId(joinPoint))
                .fieldsAccessed(extractAccessedFields(result))
                .accessTimestamp(Instant.now())
                .ipAddress(getClientIp())
                .sessionId(getSessionId())
                .build();
            
            auditLogRepository.save(log);
        }
    }
}
```

---

## Summary

### **Two-Layer Security Model**

| Layer | Purpose | Implementation |
|-------|---------|----------------|
| **UI Configuration** | Controls what fields/sections are DISPLAYED | UI Config Service + OPA policies |
| **Data Filtering** | Controls what data is actually SENT | Data Filter Service + OPA policies |

### **Configuration Hierarchy**
1. **Application** → Top level container
2. **Module** → Logical grouping (Case Management, Reporting)
3. **Screen** → Individual pages/views
4. **Section** → Groups of related fields
5. **Field** → Individual form inputs
6. **Component** → Special UI elements (tables, charts)
7. **Action** → Buttons and operations
8. **Navigation** → Breadcrumbs and links

### **Key Benefits**
- **Complete Backend Control**: All UI logic in database
- **Zero Frontend Logic**: Frontend is pure renderer
- **Dynamic Updates**: Change UI without deployments
- **Granular Permissions**: Field-level access control
- **Audit Trail**: Version history of all changes
- **Performance**: Optimized queries and caching
- **Compliance Ready**: Full traceability and control
- **Data Security**: Never send data user cannot see
- **Double Protection**: UI + Data filtering layers

"screenId": "case_details_screen",
  "context": {
    "timestamp": "2025-12-27T10:00:00Z",
    "sessionId": "sess_xyz789",
    "deviceInfo": {
      "type": "web",
      "browser": "Chrome"
    }
  }
}
```

---

### Step 2: UI Config Service → RBAC/OPA Service Request

**Endpoint**: `POST /api/rbac/ui/permissions`

**Request Body**:
```json
{
  