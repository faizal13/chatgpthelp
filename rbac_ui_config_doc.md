# Backend-Controlled UI Configuration with RBAC/OPA
## Complete Request-Response Flow Documentation

---

## Table of Contents
1. [System Overview](#system-overview)
2. [Use Case: Compliance Officer Role](#use-case-compliance-officer-role)
3. [Complete Request-Response Flow](#complete-request-response-flow)
4. [OPA Rego Policies](#opa-rego-policies)
5. [Data Models](#data-models)
6. [Sequence Diagram](#sequence-diagram)

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
                     └─────────────────┘
```

### Flow Summary
1. User logs in via SSO → JWT token with user info
2. Frontend calls UI Config Service with token
3. UI Config Service extracts user context and screen requirements
4. UI Config Service calls RBAC/OPA Service for permissions
5. RBAC/OPA Service evaluates policies using OPA Engine
6. RBAC/OPA Service returns permissions
7. UI Config Service constructs filtered UI configuration
8. Frontend receives and renders UI dynamically

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

### Step 3: RBAC/OPA Service → OPA Engine

**OPA Evaluation Endpoint**: `POST http://opa:8181/v1/data/ui/permissions/evaluate`

**Input to OPA**:
```json
{
  "input": {
    "user": {
      "userId": "emp_12345",
      "roles": ["compliance_officer", "senior_staff"],
      "department": "regulatory_compliance",
      "attributes": {
        "clearanceLevel": "L3"
      }
    },
    "resource": {
      "type": "screen",
      "id": "case_details_screen",
      "classification": "confidential"
    },
    "sections": ["case_basic_info", "customer_details", "financial_information", ...],
    "fields": ["case_id", "customer_ssn", "account_balance", ...],
    "actions": ["view_case", "edit_case", "delete_case", ...]
  }
}
```

---

### Step 4: OPA Rego Policies

#### Policy File 1: `ui_permissions.rego`

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

# Compliance officers and senior analysts can see financial info
allow_section if {
    input.section == "financial_information"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "financial_information"
    has_role(input.user, "senior_analyst")
    input.user.attributes.clearanceLevel == "L3"
}

# Only compliance officers can see risk assessment
allow_section if {
    input.section == "risk_assessment"
    has_role(input.user, "compliance_officer")
}

# Compliance officers and auditors can see audit trail
allow_section if {
    input.section == "audit_trail"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "audit_trail"
    has_role(input.user, "auditor")
}

##############################################
# FIELD VISIBILITY RULES
##############################################

# Public fields - everyone can view
allow_field_view if {
    input.field in public_fields
}

# Sensitive PII - only compliance officers and senior staff
allow_field_view if {
    input.field in sensitive_pii_fields
    has_role(input.user, "compliance_officer")
}

allow_field_view if {
    input.field in sensitive_pii_fields
    has_role(input.user, "senior_staff")
    input.user.attributes.clearanceLevel in ["L2", "L3"]
}

# Financial data - compliance officers and financial analysts
allow_field_view if {
    input.field in financial_fields
    has_role(input.user, "compliance_officer")
}

allow_field_view if {
    input.field in financial_fields
    has_role(input.user, "financial_analyst")
    input.user.attributes.clearanceLevel == "L3"
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
    input.field == "risk_score"
    has_role(input.user, "compliance_officer")
    input.resource.status != "closed"
}

allow_field_edit if {
    input.field == "risk_category"
    has_role(input.user, "compliance_officer")
    input.resource.status != "closed"
}

# Compliance officers can edit notes
allow_field_edit if {
    input.field == "notes"
    has_role(input.user, "compliance_officer")
}

# Case assignment
allow_field_edit if {
    input.field == "assigned_officer"
    has_role(input.user, "compliance_officer")
}

# Cannot edit historical or closed cases
deny_field_edit if {
    input.resource.status == "closed"
    input.field != "notes"
}

# Cannot edit system-generated fields
deny_field_edit if {
    input.field in system_generated_fields
}

##############################################
# ACTION RULES
##############################################

# View case - anyone with basic access
allow_action if {
    input.action == "view_case"
    has_role(input.user, "viewer")
}

# Edit case - compliance officers and case managers
allow_action if {
    input.action == "edit_case"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "edit_case"
    has_role(input.user, "case_manager")
}

# Create case
allow_action if {
    input.action == "create_case"
    has_role(input.user, "compliance_officer")
}

# Delete case - only senior management
allow_action if {
    input.action == "delete_case"
    has_role(input.user, "senior_management")
    input.user.attributes.clearanceLevel == "L3"
}

# Approve/Reject case - compliance officers only
allow_action if {
    input.action in ["approve_case", "reject_case"]
    has_role(input.user, "compliance_officer")
}

# Export reports
allow_action if {
    input.action == "export_report"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "export_report"
    has_role(input.user, "auditor")
}

# Assign case
allow_action if {
    input.action == "assign_case"
    has_role(input.user, "compliance_officer")
}

# Add notes - most roles can add notes
allow_action if {
    input.action == "add_notes"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "add_notes"
    has_role(input.user, "case_manager")
}

# Upload documents
allow_action if {
    input.action == "upload_documents"
    has_role(input.user, "compliance_officer")
}

##############################################
# HELPER FUNCTIONS
##############################################

# Check if user has a specific role
has_role(user, role) if {
    user.roles[_] == role
}

# Public fields visible to everyone
public_fields := {
    "case_id",
    "case_status",
    "created_date",
    "assigned_officer"
}

# Sensitive PII fields
sensitive_pii_fields := {
    "customer_ssn",
    "customer_dob",
    "customer_email",
    "customer_phone"
}

# Financial fields
financial_fields := {
    "account_number",
    "account_balance",
    "transaction_amount",
    "transaction_history"
}

# Risk assessment fields
risk_fields := {
    "risk_score",
    "risk_category",
    "risk_indicators"
}

# System-generated fields (never editable)
system_generated_fields := {
    "case_id",
    "created_date",
    "last_modified_date",
    "system_audit_log"
}
```

#### Policy File 2: `data_masking.rego`

```rego
package ui.permissions.masking

import future.keywords.if
import future.keywords.in

# Default - no masking
default mask_field = false
default masking_pattern = null

# Mask SSN for non-compliance officers
mask_field if {
    input.field == "customer_ssn"
    not has_role(input.user, "compliance_officer")
    not has_role(input.user, "senior_management")
}

masking_pattern := "XXX-XX-{last4}" if {
    input.field == "customer_ssn"
    mask_field
}

# Mask account numbers for junior staff
mask_field if {
    input.field == "account_number"
    has_role(input.user, "junior_analyst")
}

masking_pattern := "****-****-****-{last4}" if {
    input.field == "account_number"
    mask_field
}

# Partial masking for account balance (show only range)
mask_field if {
    input.field == "account_balance"
    has_role(input.user, "viewer")
}

masking_pattern := "{range}" if {
    input.field == "account_balance"
    mask_field
}

# Helper function
has_role(user, role) if {
    user.roles[_] == role
}
```

---

### Step 5: Response from RBAC/OPA Service → UI Config Service

**Response Body**:
```json
{
  "requestId": "req_abc123",
  "evaluationId": "eval_def456",
  "userId": "emp_12345",
  "evaluatedAt": "2025-12-27T10:00:01.234Z",
  "sectionPermissions": {
    "case_basic_info": {
      "allowed": true,
      "reason": "public_section"
    },
    "customer_details": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "financial_information": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "risk_assessment": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "audit_trail": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "case_actions": {
      "allowed": true,
      "reason": "compliance_officer_access"
    }
  },
  "fieldPermissions": {
    "case_id": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "public_field"
    },
    "case_status": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "public_field"
    },
    "customer_name": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "compliance_officer_access"
    },
    "customer_ssn": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "maskingPattern": null,
      "reason": "compliance_officer_full_access"
    },
    "customer_dob": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "sensitive_pii_access"
    },
    "account_number": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "financial_field_access"
    },
    "account_balance": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "financial_field_access"
    },
    "transaction_amount": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "financial_field_access"
    },
    "risk_score": {
      "canView": true,
      "canEdit": true,
      "masked": false,
      "reason": "compliance_officer_risk_access"
    },
    "risk_category": {
      "canView": true,
      "canEdit": true,
      "masked": false,
      "reason": "compliance_officer_risk_access"
    },
    "assigned_officer": {
      "canView": true,
      "canEdit": true,
      "masked": false,
      "reason": "case_assignment_access"
    },
    "created_date": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "system_field"
    },
    "last_modified_date": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "system_field"
    },
    "notes": {
      "canView": true,
      "canEdit": true,
      "masked": false,
      "reason": "compliance_officer_access"
    },
    "supporting_documents": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "reason": "compliance_officer_access"
    }
  },
  "actionPermissions": {
    "view_case": {
      "allowed": true,
      "reason": "basic_access"
    },
    "edit_case": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "create_case": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "delete_case": {
      "allowed": false,
      "reason": "requires_senior_management"
    },
    "approve_case": {
      "allowed": true,
      "reason": "compliance_officer_authority"
    },
    "reject_case": {
      "allowed": true,
      "reason": "compliance_officer_authority"
    },
    "export_report": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "assign_case": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "add_notes": {
      "allowed": true,
      "reason": "compliance_officer_access"
    },
    "upload_documents": {
      "allowed": true,
      "reason": "compliance_officer_access"
    }
  },
  "auditInfo": {
    "policyVersion": "v2.3.1",
    "evaluationTime": "123ms",
    "policiesEvaluated": [
      "ui.permissions.allow_section",
      "ui.permissions.allow_field_view",
      "ui.permissions.allow_field_edit",
      "ui.permissions.allow_action"
    ]
  }
}
```

---

### Step 6: Response from UI Config Service → Frontend

**Response Body**:
```json
{
  "screenId": "case_details_screen",
  "title": "Case Management Dashboard",
  "layout": "two-column",
  "sections": [
    {
      "id": "case_basic_info",
      "label": "Case Information",
      "visible": true,
      "order": 1,
      "columns": 2,
      "fields": [
        {
          "name": "case_id",
          "type": "text",
          "label": "Case ID",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "required": false,
          "placeholder": "",
          "helpText": "System generated case identifier",
          "validation": {}
        },
        {
          "name": "case_status",
          "type": "select",
          "label": "Status",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "required": false,
          "options": [
            {"value": "open", "label": "Open"},
            {"value": "in_progress", "label": "In Progress"},
            {"value": "closed", "label": "Closed"}
          ]
        },
        {
          "name": "assigned_officer",
          "type": "select",
          "label": "Assigned Officer",
          "visible": true,
          "editable": true,
          "readOnly": false,
          "required": true,
          "dataSource": "/api/officers",
          "helpText": "Select the responsible compliance officer"
        },
        {
          "name": "created_date",
          "type": "date",
          "label": "Created Date",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "format": "YYYY-MM-DD HH:mm:ss"
        }
      ]
    },
    {
      "id": "customer_details",
      "label": "Customer Details",
      "visible": true,
      "order": 2,
      "columns": 2,
      "fields": [
        {
          "name": "customer_name",
          "type": "text",
          "label": "Customer Name",
          "visible": true,
          "editable": false,
          "readOnly": true
        },
        {
          "name": "customer_ssn",
          "type": "text",
          "label": "SSN",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "masked": false,
          "sensitiveData": true,
          "helpText": "Social Security Number"
        },
        {
          "name": "customer_dob",
          "type": "date",
          "label": "Date of Birth",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "format": "MM/DD/YYYY"
        }
      ]
    },
    {
      "id": "financial_information",
      "label": "Financial Information",
      "visible": true,
      "order": 3,
      "columns": 2,
      "collapsed": false,
      "fields": [
        {
          "name": "account_number",
          "type": "text",
          "label": "Account Number",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "masked": false
        },
        {
          "name": "account_balance",
          "type": "currency",
          "label": "Account Balance",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "masked": false,
          "currency": "USD",
          "format": "$#,##0.00"
        },
        {
          "name": "transaction_amount",
          "type": "currency",
          "label": "Transaction Amount",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "currency": "USD",
          "format": "$#,##0.00",
          "helpText": "Amount flagged for review"
        }
      ]
    },
    {
      "id": "risk_assessment",
      "label": "Risk Assessment",
      "visible": true,
      "order": 4,
      "columns": 1,
      "highlighted": true,
      "fields": [
        {
          "name": "risk_score",
          "type": "number",
          "label": "Risk Score",
          "visible": true,
          "editable": true,
          "readOnly": false,
          "required": true,
          "min": 0,
          "max": 100,
          "step": 1,
          "helpText": "Enter risk score (0-100)",
          "validation": {
            "min": 0,
            "max": 100,
            "message": "Risk score must be between 0 and 100"
          }
        },
        {
          "name": "risk_category",
          "type": "select",
          "label": "Risk Category",
          "visible": true,
          "editable": true,
          "readOnly": false,
          "required": true,
          "options": [
            {"value": "low", "label": "Low", "color": "green"},
            {"value": "medium", "label": "Medium", "color": "yellow"},
            {"value": "high", "label": "High", "color": "orange"},
            {"value": "critical", "label": "Critical", "color": "red"}
          ],
          "helpText": "Select appropriate risk category"
        }
      ]
    },
    {
      "id": "audit_trail",
      "label": "Audit Trail",
      "visible": true,
      "order": 5,
      "columns": 1,
      "collapsible": true,
      "collapsed": true,
      "fields": [
        {
          "name": "last_modified_date",
          "type": "datetime",
          "label": "Last Modified",
          "visible": true,
          "editable": false,
          "readOnly": true,
          "format": "YYYY-MM-DD HH:mm:ss"
        }
      ],
      "components": [
        {
          "type": "audit_log_table",
          "config": {
            "endpoint": "/api/cases/{caseId}/audit",
            "columns": ["timestamp", "user", "action", "changes"],
            "pageSize": 10
          }
        }
      ]
    },
    {
      "id": "notes_section",
      "label": "Notes & Comments",
      "visible": true,
      "order": 6,
      "columns": 1,
      "fields": [
        {
          "name": "notes",
          "type": "textarea",
          "label": "Investigation Notes",
          "visible": true,
          "editable": true,
          "readOnly": false,
          "rows": 5,
          "maxLength": 5000,
          "helpText": "Add investigation notes or comments"
        }
      ]
    }
  ],
  "actions": [
    {
      "id": "edit_case",
      "label": "Edit Case",
      "type": "primary",
      "visible": true,
      "enabled": true,
      "icon": "edit",
      "action": {
        "type": "navigate",
        "route": "/cases/{caseId}/edit"
      }
    },
    {
      "id": "approve_case",
      "label": "Approve",
      "type": "success",
      "visible": true,
      "enabled": true,
      "icon": "check",
      "confirmationRequired": true,
      "confirmationMessage": "Are you sure you want to approve this case?",
      "action": {
        "type": "api",
        "method": "POST",
        "endpoint": "/api/cases/{caseId}/approve"
      }
    },
    {
      "id": "reject_case",
      "label": "Reject",
      "type": "danger",
      "visible": true,
      "enabled": true,
      "icon": "times",
      "confirmationRequired": true,
      "confirmationMessage": "Are you sure you want to reject this case?",
      "action": {
        "type": "api",
        "method": "POST",
        "endpoint": "/api/cases/{caseId}/reject"
      }
    },
    {
      "id": "export_report",
      "label": "Export Report",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "download",
      "action": {
        "type": "download",
        "endpoint": "/api/cases/{caseId}/export",
        "format": "pdf"
      }
    },
    {
      "id": "assign_case",
      "label": "Reassign",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "user",
      "action": {
        "type": "modal",
        "modalId": "assign_case_modal"
      }
    },
    {
      "id": "upload_documents",
      "label": "Upload Documents",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "upload",
      "action": {
        "type": "file_upload",
        "endpoint": "/api/cases/{caseId}/documents",
        "acceptedFormats": ["pdf", "doc", "docx", "jpg", "png"],
        "maxSize": "10MB"
      }
    }
  ],
  "navigation": {
    "breadcrumbs": [
      {"label": "Home", "route": "/"},
      {"label": "Cases", "route": "/cases"},
      {"label": "Case Details", "route": "/cases/{caseId}"}
    ],
    "relatedLinks": [
      {"label": "View All Cases", "route": "/cases"},
      {"label": "Create New Case", "route": "/cases/new"}
    ]
  },
  "metadata": {
    "evaluatedAt": "2025-12-27T10:00:01.234Z",
    "evaluationId": "eval_def456",
    "userId": "emp_12345",
    "userRole": "compliance_officer",
    "screenVersion": "2.1.0",
    "configVersion": "1.5.3"
  }
}
```

---

## Sequence Diagram

```
┌─────────┐          ┌─────────────┐          ┌─────────────┐          ┌─────────┐
│ Frontend│          │ UI Config   │          │  RBAC/OPA   │          │   OPA   │
│         │          │  Service    │          │  Service    │          │  Engine │
└────┬────┘          └──────┬──────┘          └──────┬──────┘          └────┬────┘
     │                      │                        │                       │
     │ 1. POST /api/ui/config                       │                       │
     │    (JWT + screenId)  │                        │                       │
     ├─────────────────────>│                        │                       │
     │                      │                        │                       │
     │                      │ 2. Extract user from JWT                       │
     │                      │    Fetch screen def    │                       │
     │                      │    from DB             │                       │
     │                      │                        │                       │
     │                      │ 3. POST /api/rbac/ui/permissions              │
     │                      │    (user + sections +  │                       │
     │                      │     fields + actions)  │                       │
     │                      ├───────────────────────>│                       │
     │                      │