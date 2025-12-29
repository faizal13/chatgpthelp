# Backend-Controlled UI with RBAC/OPA
## Part 3: Complete Request-Response Flow

---

## Table of Contents
1. [Use Case: Compliance Officer](#use-case-compliance-officer)
2. [Flow 1: UI Configuration Request](#flow-1-ui-configuration-request)
3. [Flow 2: Data Retrieval Request (GET)](#flow-2-data-retrieval-request-get)
4. [Flow 3: Data Creation Request (POST)](#flow-3-data-creation-request-post)
5. [Flow 4: Data Update Request (PUT)](#flow-4-data-update-request-put)
6. [Comparison: Different User Roles](#comparison-different-user-roles)

---

## Use Case: Compliance Officer

### **User Profile**
- **Name**: Sarah Johnson
- **User ID**: emp_12345
- **Email**: sarah.johnson@bank.com
- **Roles**: `compliance_officer`, `senior_staff`
- **Department**: Regulatory Compliance
- **Clearance Level**: L3
- **Region**: North America

### **Access Requirements**
✅ Can view all case details including sensitive PII  
✅ Can view financial information without masking  
✅ Can view and edit risk assessment scores  
✅ Can approve/reject cases  
✅ Can view audit trails  
✅ Can export reports  
❌ Cannot delete cases (requires senior management)  
❌ Cannot modify closed/archived cases  

### **Screen**: Case Management Dashboard (`case_details_screen`)

---

## Flow 1: UI Configuration Request

### **Step 1: Frontend Request (After SSO Login)**

```http
POST /api/ui/config
Host: config-service.bank.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**JWT Token Decoded:**
```json
{
  "sub": "sarah.johnson@bank.com",
  "userId": "emp_12345",
  "roles": ["compliance_officer", "senior_staff"],
  "department": "regulatory_compliance",
  "region": "north_america",
  "attributes": {
    "clearanceLevel": "L3",
    "yearsOfService": 8
  },
  "iat": 1703678400,
  "exp": 1703764800
}
```

**Request Body:**
```json
{
  "screenId": "case_details_screen",
  "context": {
    "timestamp": "2025-12-27T14:30:00Z",
    "sessionId": "sess_xyz789",
    "deviceInfo": {
      "type": "web",
      "browser": "Chrome"
    }
  }
}
```

---

### **Step 2: UI Config Service → RBAC/OPA Service**

**Endpoint:** `POST /api/rbac/ui/permissions`

**Request Body:**
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
    "audit_trail"
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
    "notes"
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
    "upload_documents"
  ],
  "evaluationContext": {
    "timestamp": "2025-12-27T14:30:00Z",
    "sessionId": "sess_xyz789"
  }
}
```

---

### **Step 3: RBAC/OPA Service → OPA Engine**

**Endpoint:** `POST http://opa:8181/v1/data/permissions/evaluate_fields`

**Request to OPA:**
```json
{
  "input": {
    "user": {
      "userId": "emp_12345",
      "roles": ["compliance_officer", "senior_staff"],
      "attributes": {
        "clearanceLevel": "L3",
        "region": "north_america"
      }
    },
    "operation": "view",
    "context": {
      "resource": "screen",
      "resourceId": "case_details_screen",
      "timestamp": "2025-12-27T14:30:00Z"
    },
    "fields": [
      {
        "name": "case_id",
        "metadata": {
          "classification": "public",
          "is_sensitive": false,
          "is_system_field": true
        }
      },
      {
        "name": "customer_ssn",
        "metadata": {
          "classification": "pii",
          "is_sensitive": true,
          "is_system_field": false
        }
      },
      {
        "name": "account_balance",
        "metadata": {
          "classification": "financial",
          "is_sensitive": true,
          "is_system_field": false
        }
      },
      {
        "name": "risk_score",
        "metadata": {
          "classification": "risk_assessment",
          "is_sensitive": false,
          "is_system_field": false
        }
      }
    ]
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "case_id": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "case_status": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "customer_name": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "customer_ssn": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "customer_dob": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "account_number": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "account_balance": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "transaction_amount": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "risk_score": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "risk_category": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "assigned_officer": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "created_date": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    },
    "notes": {
      "allow": true,
      "mask_required": false,
      "mask_pattern": null
    }
  }
}
```

---

### **Step 4: RBAC/OPA Service → UI Config Service**

**Response Body:**
```json
{
  "requestId": "req_abc123",
  "evaluationId": "eval_def456",
  "userId": "emp_12345",
  "evaluatedAt": "2025-12-27T14:30:01.234Z",
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
    }
  },
  "fieldPermissions": {
    "case_id": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "case_status": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "customer_name": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "customer_ssn": {
      "canView": true,
      "canEdit": false,
      "masked": false,
      "maskingPattern": null
    },
    "customer_dob": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "account_number": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "account_balance": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "transaction_amount": {
      "canView": true,
      "canEdit": false,
      "masked": false
    },
    "risk_score": {
      "canView": true,
      "canEdit": true,
      "masked": false
    },
    "risk_category": {
      "canView": true,
      "canEdit": true,
      "masked": false
    },
    "assigned_officer": {
      "canView": true,
      "canEdit": true,
      "masked": false
    },
    "notes": {
      "canView": true,
      "canEdit": true,
      "masked": false
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
    "upload_documents": {
      "allowed": true,
      "reason": "compliance_officer_access"
    }
  },
  "auditInfo": {
    "policyVersion": "v2.3.1",
    "evaluationTime": "123ms",
    "policiesEvaluated": [
      "permissions.allow",
      "permissions.allow_section",
      "permissions.allow_action"
    ]
  }
}
```

---

### **Step 5: UI Config Service → Frontend**

**Response Body:**
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
          "placeholder": "",
          "helpText": "System generated case identifier"
        },
        {
          "name": "case_status",
          "type": "select",
          "label": "Status",
          "visible": true,
          "editable": false,
          "readOnly": true,
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
          "dataSource": "/api/officers"
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
          "sensitiveData": true
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
          "currency": "USD"
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
          "helpText": "Enter risk score (0-100)",
          "validation": {
            "min": 0,
            "max": 100,
            "message": "Must be between 0 and 100"
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
          ]
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
      "fields": [
        {
          "name": "notes",
          "type": "textarea",
          "label": "Investigation Notes",
          "visible": true,
          "editable": true,
          "readOnly": false,
          "rows": 5,
          "maxLength": 5000
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
      "icon": "edit"
    },
    {
      "id": "approve_case",
      "label": "Approve",
      "type": "success",
      "visible": true,
      "enabled": true,
      "icon": "check",
      "confirmationRequired": true,
      "confirmationMessage": "Are you sure you want to approve this case?"
    },
    {
      "id": "reject_case",
      "label": "Reject",
      "type": "danger",
      "visible": true,
      "enabled": true,
      "icon": "times",
      "confirmationRequired": true
    },
    {
      "id": "export_report",
      "label": "Export Report",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "download"
    },
    {
      "id": "assign_case",
      "label": "Reassign",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "user"
    },
    {
      "id": "upload_documents",
      "label": "Upload Documents",
      "type": "secondary",
      "visible": true,
      "enabled": true,
      "icon": "upload"
    }
  ],
  "navigation": {
    "breadcrumbs": [
      {"label": "Home", "route": "/"},
      {"label": "Cases", "route": "/cases"},
      {"label": "Case Details", "route": "/cases/{caseId}"}
    ]
  },
  "metadata": {
    "evaluatedAt": "2025-12-27T14:30:01.234Z",
    "evaluationId": "eval_def456",
    "userId": "emp_12345",
    "userRole": "compliance_officer",
    "screenVersion": "2.1.0"
  }
}
```

---

## Flow 2: Data Retrieval Request (GET)

### **Step 1: Frontend Request**

```http
GET /api/cases/CASE123456
Host: api.bank.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

### **Step 2: Backend Fetches Full Data**

Backend controller retrieves complete case data from database:

```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "123-45-6789",
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,
  "transaction_amount": 15000.00,
  "risk_score": 85,
  "risk_category": "high",
  "assigned_officer": "Sarah Johnson",
  "created_date": "2025-12-01T10:00:00Z",
  "last_modified_date": "2025-12-27T14:00:00Z",
  "notes": "Initial review completed"
}
```

---

### **Step 3: Data Filter Intercepts Response**

**Request to OPA:**
```json
{
  "input": {
    "user": {
      "userId": "emp_12345",
      "roles": ["compliance_officer"],
      "attributes": {"clearanceLevel": "L3"}
    },
    "operation": "view",
    "context": {
      "resource": "case",
      "resourceId": "CASE123456",
      "resourceStatus": "open"
    },
    "fields": [
      {"name": "case_id", "metadata": {"classification": "public"}},
      {"name": "customer_ssn", "metadata": {"classification": "pii"}},
      {"name": "account_balance", "metadata": {"classification": "financial"}},
      {"name": "risk_score", "metadata": {"classification": "risk_assessment"}}
    ]
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "case_id": {"allow": true, "mask_required": false},
    "customer_ssn": {"allow": true, "mask_required": false},
    "account_balance": {"allow": true, "mask_required": false},
    "risk_score": {"allow": true, "mask_required": false}
  }
}
```

---

### **Step 4: Filtered Response to Frontend**

```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "123-45-6789",
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,
  "transaction_amount": 15000.00,
  "risk_score": 85,
  "risk_category": "high",
  "assigned_officer": "Sarah Johnson",
  "created_date": "2025-12-01T10:00:00Z",
  "last_modified_date": "2025-12-27T14:00:00Z",
  "notes": "Initial review completed"
}
```

**Note:** Compliance officer receives ALL fields without masking.

---

## Flow 3: Data Creation Request (POST)

### **Step 1: Frontend Request**

```http
POST /api/cases
Host: api.bank.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body:**
```json
{
  "customer_name": "Jane Smith",
  "customer_ssn": "987-65-4321",
  "customer_dob": "1990-05-20",
  "account_number": "1234567890",
  "account_balance": 75000.00,
  "transaction_amount": 25000.00,
  "risk_score": 65,
  "risk_category": "medium",
  "notes": "New compliance case for review"
}
```

---

### **Step 2: Request Validator Intercepts**

For each field, validate with OPA:

**OPA Request (for customer_ssn):**
```json
{
  "input": {
    "user": {
      "userId": "emp_12345",
      "roles": ["compliance_officer"]
    },
    "operation": "create",
    "field": "customer_ssn",
    "fieldMetadata": {
      "classification": "pii",
      "is_sensitive": true,
      "is_system_field": false
    },
    "context": {
      "resource": "case"
    }
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": true  // ✅ Compliance officer can create with PII fields
  }
}
```

---

### **Step 3: All Fields Validated - Request Proceeds**

Since all fields are allowed, the request proceeds to create the case.

**Response:**
```json
{
  "case_id": "CASE123457",
  "status": "created",
  "message": "Case created successfully",
  "created_at": "2025-12-27T14:35:00Z"
}
```

---

## Flow 4: Data Update Request (PUT)

### **Step 1: Frontend Request**

```http
PUT /api/cases/CASE123456
Host: api.bank.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

**Request Body:**
```json
{
  "risk_score": 90,
  "risk_category": "critical",
  "notes": "Risk level escalated based on new evidence"
}
```

---

### **Step 2: Validate Each Field with OPA**

**For risk_score:**
```json
{
  "input": {
    "user": {"userId": "emp_12345", "roles": ["compliance_officer"]},
    "operation": "edit",
    "field": "risk_score",
    "fieldMetadata": {"classification": "risk_assessment"},
    "context": {
      "resource": "case",
      "resourceId": "CASE123456",
      "resourceStatus": "open"
    }
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": true  // ✅ Compliance officer can edit risk fields
  }
}
```

---

### **Step 3: Update Proceeds**

**Response:**
```json
{
  "case_id": "CASE123456",
  "status": "updated",
  "message": "Case updated successfully",
  "updated_fields": ["risk_score", "risk_category", "notes"],
  "updated_at": "2025-12-27T14:40:00Z"
}
```

---

## Comparison: Different User Roles

### **Scenario: Same Case, Different Users**

**Case ID:** CASE123456  
**Full Data Available:**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "123-45-6789",
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,
  "risk_score": 85,
  "risk_category": "high",
  "notes": "Initial review completed"
}
```

---

### **User 1: Compliance Officer (Sarah Johnson)**

**Roles:** `compliance_officer`, `senior_staff`  
**Clearance:** L3

**What Sarah Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "123-45-6789",        // ✅ FULL SSN
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,          // ✅ FULL BALANCE
  "risk_score": 85,                     // ✅ CAN SEE
  "risk_category": "high",
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ All sections visible
- ✅ All fields visible
- ✅ Can edit: risk_score, risk_category, notes, assigned_officer
- ✅ Actions: Edit, Approve, Reject, Export, Assign, Upload
- ❌ Cannot delete case

---

### **User 2: Financial Analyst (Mike Chen)**

**Roles:** `financial_analyst`  
**Clearance:** L3

**What Mike Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  // customer_ssn: NOT SENT - No PII access
  // customer_dob: NOT SENT - No PII access
  "account_number": "9876543210",
  "account_balance": 50000.00,          // ✅ Can see financial data (L3)
  // risk_score: NOT SENT - No risk assessment access
  // risk_category: NOT SENT
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ Sections visible: case_basic_info, financial_information
- ❌ Sections hidden: customer_details, risk_assessment, audit_trail
- ✅ Fields visible: case_id, case_status, account_number, account_balance, notes
- ❌ Fields hidden: customer_ssn, customer_dob, risk_score, risk_category
- ✅ All fields read-only (cannot edit)
- ✅ Actions: View, Export
- ❌ Actions hidden: Edit, Approve, Reject, Delete, Assign

---

### **User 3: Junior Analyst (Lisa Wang)**

**Roles:** `junior_analyst`  
**Clearance:** L1

**What Lisa Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  // customer_ssn: NOT SENT
  // customer_dob: NOT SENT
  "account_number": "****-****-****-3210",  // ✅ MASKED for junior analyst
  // account_balance: NOT SENT - No financial access at L1
  // risk_score: NOT SENT
  // risk_category: NOT SENT
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ Sections visible: case_basic_info
- ❌ Sections hidden: customer_details, financial_information, risk_assessment, audit_trail
- ✅ Fields visible: case_id, case_status, customer_name, notes
- ✅ account_number visible but MASKED
- ❌ Fields hidden: All PII, financial, and risk fields
- ✅ All fields read-only
- ✅ Actions: View only
- ❌ Actions hidden: Edit, Approve, Reject, Export, Delete, Assign

---

### **User 4: Senior Staff (David Kumar)**

**Roles:** `senior_staff`  
**Clearance:** L3

**What David Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "XXX-XX-6789",        // ✅ MASKED (last 4 shown)
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,          // ✅ Can see (L3 clearance)
  // risk_score: NOT SENT - No risk assessment access
  // risk_category: NOT SENT
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ Sections visible: case_basic_info, customer_details, financial_information
- ❌ Sections hidden: risk_assessment, audit_trail
- ✅ Fields visible: Most fields except risk assessment
- ⚠️ customer_ssn visible but MASKED
- ✅ All fields read-only (cannot edit anything)
- ✅ Actions: View, Export
- ❌ Actions hidden: Edit, Approve, Reject, Delete, Assign

---

### **User 5: Case Manager (Emily Brown)**

**Roles:** `case_manager`

**What Emily Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  // customer_ssn: NOT SENT - No PII access
  // customer_dob: NOT SENT
  // account_number: NOT SENT - No financial access
  // account_balance: NOT SENT
  // risk_score: NOT SENT - No risk assessment access
  // risk_category: NOT SENT
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ Sections visible: case_basic_info
- ❌ Sections hidden: customer_details, financial_information, risk_assessment, audit_trail
- ✅ Fields visible: case_id, case_status, customer_name, notes, assigned_officer
- ✅ Can edit: notes, assigned_officer (on open cases)
- ✅ Actions: View, Edit (basic fields only), Create Case
- ❌ Actions hidden: Approve, Reject, Delete, Export

---

### **User 6: Auditor (Robert Chen)**

**Roles:** `auditor`  
**Clearance:** L2

**What Robert Sees (GET /api/cases/CASE123456):**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "XXX-XX-6789",        // ✅ MASKED (auditor sees partial)
  "customer_dob": "1985-03-15",
  "account_number": "9876543210",
  "account_balance": 50000.00,
  "risk_score": 85,                     // ❌ Actually NOT visible
  "risk_category": "high",              // ❌ Actually NOT visible
  "notes": "Initial review completed"
}
```

**Correction - What Robert ACTUALLY Sees:**
```json
{
  "case_id": "CASE123456",
  "case_status": "open",
  "customer_name": "John Doe",
  "customer_ssn": "XXX-XX-6789",        // ✅ MASKED
  "customer_dob": "1985-03-15",
  // account_number: NOT SENT - Auditors don't see financial details
  // account_balance: NOT SENT
  // risk_score: NOT SENT
  // risk_category: NOT SENT
  "notes": "Initial review completed"
}
```

**UI Configuration:**
- ✅ Sections visible: case_basic_info, customer_details, audit_trail
- ❌ Sections hidden: financial_information, risk_assessment
- ✅ Fields visible: case_id, case_status, customer_name, customer_ssn (masked), notes
- ⚠️ Special access: Full audit trail (can see all historical changes)
- ✅ All fields read-only (cannot edit)
- ✅ Actions: View, Export Report
- ❌ Actions hidden: Edit, Approve, Reject, Delete, Assign

---

## Side-by-Side Comparison Table

| Field | Compliance Officer | Financial Analyst (L3) | Junior Analyst (L1) | Senior Staff (L3) | Case Manager | Auditor |
|-------|-------------------|----------------------|---------------------|------------------|--------------|---------|
| **case_id** | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View |
| **case_status** | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View |
| **customer_name** | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View | ✅ View |
| **customer_ssn** | ✅ Full | ❌ Hidden | ❌ Hidden | ⚠️ Masked | ❌ Hidden | ⚠️ Masked |
| **customer_dob** | ✅ View | ❌ Hidden | ❌ Hidden | ✅ View | ❌ Hidden | ✅ View |
| **account_number** | ✅ View | ✅ View | ⚠️ Masked | ✅ View | ❌ Hidden | ❌ Hidden |
| **account_balance** | ✅ View | ✅ View | ❌ Hidden | ✅ View | ❌ Hidden | ❌ Hidden |
| **risk_score** | ✅ Edit | ❌ Hidden | ❌ Hidden | ❌ Hidden | ❌ Hidden | ❌ Hidden |
| **risk_category** | ✅ Edit | ❌ Hidden | ❌ Hidden | ❌ Hidden | ❌ Hidden | ❌ Hidden |
| **notes** | ✅ Edit | ✅ View | ✅ View | ✅ View | ✅ Edit | ✅ View |
| **assigned_officer** | ✅ Edit | ✅ View | ✅ View | ✅ View | ✅ Edit | ✅ View |
| **Actions** | | | | | | |
| View | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Edit | ✅ | ❌ | ❌ | ❌ | ⚠️ Limited | ❌ |
| Create | ✅ | ❌ | ❌ | ❌ | ⚠️ Limited | ❌ |
| Delete | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Approve/Reject | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Export | ✅ | ✅ | ❌ | ✅ | ❌ | ✅ |
| Assign | ✅ | ❌ | ❌ | ❌ | ✅ | ❌ |

**Legend:**
- ✅ Full Access
- ⚠️ Partial/Masked Access
- ❌ No Access (field not sent to frontend)

---

## Sequence Diagrams

### **Complete Flow: UI Config + Data Retrieval**

```
┌─────────┐    ┌───────────┐    ┌──────────┐    ┌─────┐
│Frontend │    │UI Config  │    │RBAC/OPA  │    │ OPA │
│         │    │Service    │    │Service   │    │     │
└────┬────┘    └─────┬─────┘    └────┬─────┘    └──┬──┘
     │               │               │             │
     │ 1. Login via SSO              │             │
     │ (Get JWT Token)               │             │
     │               │               │             │
     │ 2. POST /ui/config            │             │
     │   screenId=case_details       │             │
     ├──────────────>│               │             │
     │               │               │             │
     │               │ 3. Query DB   │             │
     │               │   Get Screen  │             │
     │               │   Metadata    │             │
     │               │               │             │
     │               │ 4. POST /rbac/│             │
     │               │    permissions│             │
     │               ├──────────────>│             │
     │               │               │             │
     │               │               │ 5. Evaluate │
     │               │               │    Policies │
     │               │               ├────────────>│
     │               │               │             │
     │               │               │ 6. Results  │
     │               │               │<────────────┤
     │               │               │             │
     │               │ 7. Permission │             │
     │               │    Response   │             │
     │               │<──────────────┤             │
     │               │               │             │
     │               │ 8. Filter &   │             │
     │               │    Construct  │             │
     │               │    UI Config  │             │
     │               │               │             │
     │ 9. UI Config  │               │             │
     │    Response   │               │             │
     │<──────────────┤               │             │
     │               │               │             │
     │ 10. Render UI │               │             │
     │               │               │             │
     │               │               │             │
     ├───────────────┴───────────────┴─────────────┤
     │                                              │
     │        NOW USER INTERACTS WITH UI           │
     │                                              │
     ├───────────────┬───────────────┬─────────────┤
     │               │               │             │
     │ 11. GET /api/cases/CASE123456 │             │
     ├──────────────────────────────>│             │
     │               │               │             │
     │               │ 12. Fetch     │             │
     │               │     Full Data │             │
     │               │     from DB   │             │
     │               │               │             │
     │               │ 13. Intercept │             │
     │               │     Response  │             │
     │               │               │             │
     │               │ 14. POST OPA  │             │
     │               │     Field     │             │
     │               │     Evaluation│             │
     │               ├──────────────────────────────>
     │               │               │             │
     │               │               │ 15. Field   │
     │               │               │     Decisions
     │               │<──────────────────────────────┤
     │               │               │             │
     │               │ 16. Filter    │             │
     │               │     Response  │             │
     │               │     Data      │             │
     │               │               │             │
     │ 17. Filtered  │               │             │
     │     Data      │               │             │
     │<──────────────┤               │             │
     │               │               │             │
     │ 18. Display   │               │             │
     │     Data in   │               │             │
     │     UI        │               │             │
     │               │               │             │
```

---

## Error Scenarios

### **Scenario 1: Unauthorized Field Access (POST)**

**Request:**
```http
POST /api/cases
Authorization: Bearer <case_manager_token>
```

```json
{
  "customer_name": "Jane Smith",
  "customer_ssn": "987-65-4321",  // ❌ Case manager cannot set PII
  "risk_score": 75,               // ❌ Case manager cannot set risk
  "notes": "New case"
}
```

**OPA Evaluation:**
- `customer_name`: ✅ Allowed
- `customer_ssn`: ❌ Denied (classification: pii)
- `risk_score`: ❌ Denied (classification: risk_assessment)
- `notes`: ✅ Allowed

**Response:**
```http
HTTP 403 Forbidden
```

```json
{
  "error": "PermissionDenied",
  "message": "You don't have permission to set the following fields",
  "deniedFields": ["customer_ssn", "risk_score"],
  "reason": "case_manager role cannot create PII or risk assessment fields",
  "timestamp": "2025-12-27T14:45:00Z"
}
```

---

### **Scenario 2: Edit Closed Case**

**Request:**
```http
PUT /api/cases/CASE123456
Authorization: Bearer <compliance_officer_token>
```

```json
{
  "risk_score": 95,
  "notes": "Updated risk assessment"
}
```

**Backend checks case status: "closed"**

**OPA Evaluation:**
```json
{
  "input": {
    "user": {"roles": ["compliance_officer"]},
    "operation": "edit",
    "field": "risk_score",
    "fieldMetadata": {"classification": "risk_assessment"},
    "context": {
      "resourceStatus": "closed"  // ❌ This triggers deny rule
    }
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": false  // ❌ Rule 9: Cannot edit closed resources
  }
}
```

**Response:**
```http
HTTP 403 Forbidden
```

```json
{
  "error": "PermissionDenied",
  "message": "Cannot edit fields on closed cases",
  "caseStatus": "closed",
  "deniedFields": ["risk_score"],
  "hint": "Only notes can be added to closed cases by compliance officers",
  "timestamp": "2025-12-27T14:50:00Z"
}
```

---

### **Scenario 3: Time-Based Access Restriction**

**Request:**
```http
GET /api/cases/CASE123456
Authorization: Bearer <junior_analyst_token>
Time: 2025-12-27T22:30:00Z (10:30 PM - Outside business hours)
```

**OPA Evaluation:**
```json
{
  "input": {
    "user": {"roles": ["junior_analyst"]},
    "operation": "view",
    "fieldMetadata": {"classification": "financial"},
    "context": {
      "timestamp": "2025-12-27T22:30:00Z"
    }
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": false  // ❌ Rule 23: Outside business hours
  }
}
```

**Response:**
```http
HTTP 403 Forbidden
```

```json
{
  "error": "TimeBasedRestriction",
  "message": "Financial data access is restricted to business hours (9 AM - 6 PM)",
  "currentTime": "2025-12-27T22:30:00Z",
  "allowedHours": "09:00 - 18:00",
  "timestamp": "2025-12-27T22:30:00Z"
}
```

---

## Performance Considerations

### **Optimization 1: Batch OPA Evaluation**

Instead of calling OPA for each field individually:

❌ **Bad: 10 fields = 10 OPA calls**
```
Call 1: Check case_id
Call 2: Check customer_ssn
Call 3: Check account_balance
...
Call 10: Check notes
```

✅ **Good: 10 fields = 1 OPA call**
```
Single call with all 10 fields → OPA evaluates in parallel
```

**Performance Improvement:** ~90% reduction in network calls

---

### **Optimization 2: Caching Strategy**

| Cache Type | Duration | Key | Invalidation |
|-----------|----------|-----|--------------|
| UI Config (Base) | 1 hour | `screen:{screenId}` | On screen update |
| User Permissions | 15 min | `perm:{userId}:{screenId}` | On role change |
| Final UI Config | 5 min | `ui:{userId}:{screenId}` | On permission change |
| Field Options | 30 min | `options:{fieldId}` | On data change |
| Case Data | 2 min | `case:{caseId}` | On case update |

---

### **Optimization 3: Lazy Loading**

**Initial Load (Essential):**
1. Screen structure
2. Field metadata
3. Permission evaluation
4. Basic case data

**Lazy Load (On Demand):**
1. Audit trail data (when section expanded)
2. Document list (when tab opened)
3. Historical data (when requested)
4. Related cases (when needed)

---

## Summary

### **Key Takeaways**

1. **Two API Calls for Complete Experience:**
   - Call 1: `/api/ui/config` → Get UI structure
   - Call 2: `/api/cases/{id}` → Get filtered data

2. **Complete Security:**
   - UI controls WHAT to display
   - Data filter controls WHAT to send
   - OPA is single source of truth

3. **Role-Based Experience:**
   - Different users see different data
   - Same API, different responses
   - No data leakage to frontend

4. **Performance Optimized:**
   - Batch OPA evaluations
   - Multi-level caching
   - Lazy loading for heavy data

5. **Audit Ready:**
   - Every OPA decision logged
   - Complete trail of who saw what
   - Version history of UI changes

### **Data Flow Summary**

```
User Login → JWT Token → Frontend → UI Config Service
                                   ↓
                            Query Database (Screen Metadata)
                                   ↓
                            Call RBAC/OPA Service
                                   ↓
                            OPA Evaluates Policies
                                   ↓
                            Build Filtered UI Config
                                   ↓
                            Return to Frontend
                                   ↓
                            Render Dynamic UI
                                   ↓
User Requests Data → API Call → Backend Fetches Full Data
                                   ↓
                            Intercept Response
                                   ↓
                            Call OPA for Field Permissions
                                   ↓
                            Filter Response Data
                                   ↓
                            Return Only Allowed Fields
                                   ↓
                            Frontend Displays Data
```

---

## Next Steps

You now have complete documentation for:

✅ **Part 1:** Database Schema & Architecture  
✅ **Part 2:** OPA Policies & Permission Model  
✅ **Part 3:** Complete Request-Response Flow  

**Implementation Checklist:**

1. ✅ Create database tables (Part 1)
2. ✅ Deploy OPA with policies (Part 2)
3. ✅ Implement UI Config Service (Java - optional)
4. ✅ Implement Data Filter Service (Java - optional)
5. ✅ Build frontend rendering engine
6. ✅ Test with different user roles
7. ✅ Set up caching layer
8. ✅ Configure audit logging
9. ✅ Performance testing
10. ✅ Security audit

**You're ready to build your banking compliance CLM system!** 🎉