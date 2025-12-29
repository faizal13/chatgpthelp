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
  