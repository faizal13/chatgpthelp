# Backend-Controlled UI with RBAC/OPA
## Part 1: Database Schema & System Architecture

---

## Table of Contents
1. [System Overview](#system-overview)
2. [Complete Database Schema](#complete-database-schema)
3. [Configuration Management Strategy](#configuration-management-strategy)

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

### Entity Hierarchy

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

### Flow Summary
1. User logs in via SSO → JWT token with user info
2. Frontend calls UI Config Service
3. UI Config Service extracts screen requirements from database
4. UI Config Service calls RBAC/OPA Service for permissions
5. RBAC/OPA Service evaluates policies using OPA Engine
6. UI Config Service constructs filtered UI configuration
7. Frontend receives and renders UI dynamically

---

## Complete Database Schema

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
| metadata | JSONB | | Additional metadata | `{"theme": "corporate"}` |

**Sample Data:**
```sql
INSERT INTO applications VALUES 
('app_001', 'CLM Application', 'CLM_APP', 'Compliance Lifecycle Management', 
'2.1.0', true, NOW(), NOW(), '{"theme": "corporate", "locale": "en-US"}');
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
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO modules VALUES 
('mod_001', 'app_001', 'Case Management', 'CASE_MGMT', 
'Manage compliance cases', 'briefcase', 1, true, NOW(), NOW());
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
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| metadata | JSONB | | Additional config | `{"showBreadcrumbs": true}` |

**Sample Data:**
```sql
INSERT INTO screens VALUES 
('case_details_screen', 'mod_001', 'Case Details', 'Case Management Dashboard', 
'/cases/:caseId', 'two-column', 'Detailed case information view', 'file-text', 
2, true, 'confidential', NOW(), NOW(), '{"showBreadcrumbs": true}');
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
| css_class | VARCHAR(100) | | Custom CSS class | `highlight-section` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO sections VALUES 
('case_basic_info', 'case_details_screen', 'Case Information', 
'Basic Case Information', 'Core case details', 1, 2, false, false, false, 
true, 'standard-section', NOW(), NOW());

INSERT INTO sections VALUES 
('risk_assessment', 'case_details_screen', 'Risk Assessment', 
'Risk Analysis', 'Risk scoring and categorization', 4, 1, true, true, true, 
true, 'highlight-section', NOW(), NOW());
```

---

### 5. **TABLE: fields** (Metadata Only - Permissions in OPA)

Individual form fields within sections

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique field identifier | `fld_001` |
| section_id | VARCHAR(100) | FOREIGN KEY → sections(id) | Parent section | `case_basic_info` |
| name | VARCHAR(100) | NOT NULL | Field name (API mapping) | `customer_ssn` |
| label | VARCHAR(200) | NOT NULL | Display label | `Social Security Number` |
| field_type | VARCHAR(50) | NOT NULL | Input type | `text` |
| **classification** | **VARCHAR(50)** | **NOT NULL** | **Data classification for OPA** | **`pii`** |
| data_type | VARCHAR(50) | | Data type | `string` |
| display_order | INTEGER | DEFAULT 0 | Order within section | `5` |
| is_required | BOOLEAN | DEFAULT false | Required field | `false` |
| is_system_field | BOOLEAN | DEFAULT false | System-generated field | `false` |
| is_sensitive | BOOLEAN | DEFAULT false | Contains sensitive data | `true` |
| placeholder | VARCHAR(200) | | Placeholder text | `XXX-XX-XXXX` |
| help_text | TEXT | | Help/tooltip text | `Customer SSN` |
| validation | JSONB | | Validation rules | `{"pattern": "^[0-9]{3}-[0-9]{2}-[0-9]{4}$"}` |
| options | JSONB | | Select/radio options | `[{"value": "open", "label": "Open"}]` |
| data_source | VARCHAR(200) | | API endpoint for options | `/api/officers` |
| format | VARCHAR(100) | | Display format | `YYYY-MM-DD` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| metadata | JSONB | | Additional config | `{"encrypted": true}` |

**IMPORTANT:** 
- `classification` field is the key for OPA permission evaluation
- Classification values: `public`, `basic`, `pii`, `financial`, `risk_assessment`
- NO permission data stored here - all permissions managed by OPA policies

**Sample Data:**
```sql
-- Public field
INSERT INTO fields VALUES 
('fld_001', 'case_basic_info', 'case_id', 'Case ID', 'text', 'public', 
'string', 1, false, true, false, NULL, 'System generated case identifier', 
'{"readonly": true}', NULL, NULL, NULL, NOW(), NOW(), '{"copyable": true}');

-- PII field (sensitive)
INSERT INTO fields VALUES 
('fld_002', 'customer_details', 'customer_ssn', 'Social Security Number', 
'text', 'pii', 'string', 5, false, false, true, 'XXX-XX-XXXX', 
'Customer SSN', '{"pattern": "^[0-9]{3}-[0-9]{2}-[0-9]{4}$"}', 
NULL, NULL, NULL, NOW(), NOW(), '{"sensitive": true, "encrypted": true}');

-- Financial field
INSERT INTO fields VALUES 
('fld_003', 'financial_information', 'account_balance', 'Account Balance', 
'currency', 'financial', 'decimal', 2, false, false, true, NULL, 
'Current account balance', '{"min": 0}', NULL, NULL, '$#,##0.00', 
NOW(), NOW(), '{"currency": "USD"}');

-- Risk assessment field (editable by compliance officers)
INSERT INTO fields VALUES 
('fld_004', 'risk_assessment', 'risk_score', 'Risk Score', 'number', 
'risk_assessment', 'integer', 1, true, false, false, NULL, 
'Enter risk score between 0-100', 
'{"min": 0, "max": 100, "message": "Must be between 0 and 100"}', 
NULL, NULL, NULL, NOW(), NOW(), '{"step": 1, "showSlider": true}');
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
| config | JSONB | NOT NULL | Component configuration | `{"endpoint": "/api/audit"}` |
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
}', NOW(), NOW());

INSERT INTO components VALUES 
('comp_002', 'documents_section', 'file_upload', 'Document Upload', 1, true, 
'{
  "endpoint": "/api/cases/{caseId}/documents",
  "acceptedFormats": ["pdf", "doc", "docx"],
  "maxSize": "10MB",
  "multiple": true,
  "maxFiles": 5
}', NOW(), NOW());
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
| confirmation_message | TEXT | | Confirmation dialog text | `Are you sure?` |
| endpoint | VARCHAR(500) | | API endpoint | `/api/cases/{caseId}/approve` |
| http_method | VARCHAR(10) | | HTTP method | `POST` |
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |
| action_config | JSONB | | Additional action config | `{"successMessage": "Approved"}` |

**Sample Data:**
```sql
INSERT INTO actions VALUES 
('approve_case', 'case_details_screen', 'Approve Case', 'Approve', 'api', 
'success', 'check', 2, true, true, 'Are you sure you want to approve?', 
'/api/cases/{caseId}/approve', 'POST', NOW(), NOW(), 
'{"successMessage": "Case approved", "redirectAfter": true}');

INSERT INTO actions VALUES 
('export_report', 'case_details_screen', 'Export Report', 'Export Report', 
'download', 'secondary', 'download', 5, true, false, NULL, 
'/api/cases/{caseId}/export', 'GET', NOW(), NOW(), 
'{"format": "pdf", "fileName": "case_{caseId}_report.pdf"}');
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
| created_at | TIMESTAMP | DEFAULT NOW() | Creation timestamp | `2025-01-15 10:00:00` |
| updated_at | TIMESTAMP | DEFAULT NOW() | Last update timestamp | `2025-12-27 10:00:00` |

**Sample Data:**
```sql
INSERT INTO navigation VALUES 
('nav_001', 'case_details_screen', 'breadcrumb', 'Home', '/', 1, 'home', true, NOW(), NOW());

INSERT INTO navigation VALUES 
('nav_002', 'case_details_screen', 'breadcrumb', 'Cases', '/cases', 2, 'briefcase', true, NOW(), NOW());

INSERT INTO navigation VALUES 
('nav_003', 'case_details_screen', 'breadcrumb', 'Case Details', '/cases/{caseId}', 3, 'file-text', true, NOW(), NOW());
```

---

### 9. **TABLE: screen_versions**

Version history for screens (audit trail)

| Column | Type | Constraints | Description | Example |
|--------|------|-------------|-------------|---------|
| id | UUID | PRIMARY KEY | Unique version identifier | `ver_001` |
| screen_id | VARCHAR(100) | FOREIGN KEY → screens(id) | Target screen | `case_details_screen` |
| version | VARCHAR(20) | NOT NULL | Version number | `2.1.0` |
| change_description | TEXT | | Description of changes | `Added risk section` |
| configuration_snapshot | JSONB | NOT NULL | Full config snapshot | `{...}` |
| created_by | VARCHAR(100) | NOT NULL | User who made changes | `admin@bank.com` |
| created_at | TIMESTAMP | DEFAULT NOW() | Version creation timestamp | `2025-12-27 10:00:00` |
| is_active | BOOLEAN | DEFAULT true | Active version | `true` |

---

## Configuration Management Strategy

### **Hierarchical Grouping**

```
Application (CLM System)
    │
    ├── Module: Case Management
    │   ├── Screen: Case List
    │   ├── Screen: Case Details ────┐
    │   │                            │
    │   └── Screen: Create Case      │
    │                                 │
    └── Module: Reporting             │
        └── ...                       │
                                      │
        ┌─────────────────────────────┘
        │
        ├── Section: Basic Info
        │   ├── Field: case_id (public)
        │   ├── Field: case_status (public)
        │   └── Field: customer_name (basic)
        │
        ├── Section: Customer Details
        │   ├── Field: customer_ssn (pii) ← OPA controls access
        │   └── Field: customer_dob (pii)
        │
        ├── Section: Financial Info
        │   ├── Field: account_balance (financial)
        │   └── Field: transaction_amount (financial)
        │
        ├── Section: Risk Assessment
        │   ├── Field: risk_score (risk_assessment)
        │   └── Field: risk_category (risk_assessment)
        │
        └── Actions
            ├── Action: Approve (requires compliance_officer)
            ├── Action: Reject (requires compliance_officer)
            └── Action: Export (requires compliance_officer)
```

### **Field Classification Guide**

| Classification | Who Can View | Who Can Edit | Examples |
|---------------|--------------|--------------|----------|
| `public` | Everyone | System only | case_id, created_date |
| `basic` | Authenticated users | Case managers | customer_name, notes |
| `pii` | Compliance officers, Senior staff L2+ | Compliance officers | SSN, DOB, email |
| `financial` | Compliance officers, Financial analysts L3 | Compliance officers | account_balance, transactions |
| `risk_assessment` | Compliance officers only | Compliance officers | risk_score, risk_category |

### **Single Query to Fetch Complete Screen**

```sql
WITH screen_data AS (
    SELECT * FROM screens WHERE id = 'case_details_screen'
),
sections_data AS (
    SELECT * FROM sections 
    WHERE screen_id = 'case_details_screen' AND is_active = true 
    ORDER BY display_order
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
    WHERE sec.screen_id = 'case_details_screen' AND c.is_active = true
),
actions_data AS (
    SELECT * FROM actions
    WHERE screen_id = 'case_details_screen' AND is_active = true
    ORDER BY display_order
),
navigation_data AS (
    SELECT * FROM navigation
    WHERE screen_id = 'case_details_screen' AND is_active = true
    ORDER BY navigation_type, display_order
)
SELECT json_build_object(
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
| Screen Base Config | 1 hour | `screen:config:{screenId}` | Raw screen from DB |
| User Permissions | 15 min | `user:perm:{userId}:{screenId}` | OPA evaluation results |
| Final UI Config | 5 min | `ui:config:{userId}:{screenId}` | Complete filtered UI |
| Field Options | 30 min | `field:options:{fieldId}` | Dropdown options |

---

## Summary

### **Key Design Principles**

1. **Single Source of Truth**: Field metadata in `fields` table, permissions in OPA
2. **No Duplicate Configuration**: Classification drives OPA, no separate permission tables
3. **Hierarchical Structure**: Clear parent-child relationships
4. **Audit Trail**: Version history tracks all changes
5. **Performance**: Optimized queries and multi-level caching
6. **Flexibility**: JSONB fields for extensibility

### **Next Steps**

- **Part 2**: OPA Policies & Permission Model
- **Part 3**: Java Implementation & Data Filtering
- **Part 4**: Complete Request-Response Flow Examples