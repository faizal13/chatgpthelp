# Backend-Controlled UI with RBAC/OPA
## Part 2: OPA Policies & Permission Model

---

## Table of Contents
1. [OPA-Only Permission Model](#opa-only-permission-model)
2. [Complete OPA Policies](#complete-opa-policies)
3. [Policy Examples & Use Cases](#policy-examples--use-cases)

---

## OPA-Only Permission Model

### **Design Principle**

```
┌─────────────────────────────────────────────────────────────────┐
│           SINGLE SOURCE OF TRUTH: OPA                            │
│                                                                   │
│  Fields Table          OPA Engine           Services             │
│  (Metadata Only)       (Permissions)        (Enforcement)        │
│                                                                   │
│  ┌──────────────┐     ┌──────────────┐     ┌─────────────┐     │
│  │ customer_ssn │────>│ Evaluate:    │────>│ UI Config   │     │
│  │ type: text   │     │ - Can view?  │     │ Service     │     │
│  │ class: pii   │     │ - Can edit?  │     └─────────────┘     │
│  │ sensitive: Y │     │ - Mask?      │                          │
│  └──────────────┘     └──────────────┘     ┌─────────────┐     │
│                              │              │ Data Filter │     │
│                              └─────────────>│ Service     │     │
│                                             └─────────────┘     │
└─────────────────────────────────────────────────────────────────┘
```

### **Field Classification Values**

| Classification | Description | Security Level | Examples |
|---------------|-------------|----------------|----------|
| `public` | Everyone can view | Public | case_id, case_status, created_date |
| `basic` | Authenticated users | Internal | customer_name, notes, assigned_officer |
| `pii` | Personal identifiable info | Confidential | SSN, DOB, email, phone |
| `financial` | Financial data | Confidential | account_balance, transactions, credit_score |
| `risk_assessment` | Risk scoring data | Restricted | risk_score, risk_category, risk_indicators |

### **Why OPA-Only?**

✅ **Advantages:**
- Single source of truth for all permissions
- No duplicate configuration between database and policies
- Easier to audit and maintain
- Version-controlled policies (GitOps)
- Can evaluate complex conditions (time-based, attribute-based, etc.)
- Dynamic policy updates without database changes

❌ **Avoided Problems:**
- No sync issues between DB and OPA
- No N+1 query problems
- No permission data scattered across multiple tables
- No duplicate field definitions

---

## Complete OPA Policies

### **Main Policy File: `permissions.rego`**

```rego
package permissions

import future.keywords.if
import future.keywords.in

##############################################
# DEFAULT POLICIES
##############################################

# Default deny all operations
default allow = false
default mask_required = false
default mask_pattern = null

##############################################
# VIEW PERMISSIONS (GET requests)
##############################################

# Rule 1: Public fields - everyone can view
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "public"
}

# Rule 2: Basic fields - authenticated users only
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "basic"
    count(input.user.roles) > 0  # Has at least one role (not guest)
}

# Rule 3: PII fields - compliance officers
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "pii"
    has_role(input.user, "compliance_officer")
}

# Rule 4: PII fields - senior staff with L2 or L3 clearance
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "pii"
    has_role(input.user, "senior_staff")
    input.user.attributes.clearanceLevel in ["L2", "L3"]
}

# Rule 5: Financial fields - compliance officers
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "financial"
    has_role(input.user, "compliance_officer")
}

# Rule 6: Financial fields - financial analysts with L3 clearance
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "financial"
    has_role(input.user, "financial_analyst")
    input.user.attributes.clearanceLevel == "L3"
}

# Rule 7: Risk assessment fields - compliance officers only
allow if {
    input.operation == "view"
    input.fieldMetadata.classification == "risk_assessment"
    has_role(input.user, "compliance_officer")
}

##############################################
# EDIT PERMISSIONS (PUT/PATCH requests)
##############################################

# Rule 8: System fields cannot be edited by anyone
deny if {
    input.operation in ["edit", "create"]
    input.fieldMetadata.is_system_field == true
}

# Rule 9: Cannot edit closed or archived resources
deny if {
    input.operation == "edit"
    input.context.resourceStatus in ["closed", "archived"]
}

# Rule 10: Compliance officers can edit risk assessment fields
allow if {
    input.operation == "edit"
    input.fieldMetadata.classification == "risk_assessment"
    has_role(input.user, "compliance_officer")
    input.context.resourceStatus not in ["closed", "archived"]
}

# Rule 11: Case managers can edit basic fields
allow if {
    input.operation == "edit"
    input.fieldMetadata.classification == "basic"
    has_role(input.user, "case_manager")
    input.context.resourceStatus not in ["closed", "archived"]
}

# Rule 12: Compliance officers can always edit notes
allow if {
    input.operation == "edit"
    input.field == "notes"
    has_role(input.user, "compliance_officer")
}

# Rule 13: Case managers can edit notes on open cases
allow if {
    input.operation == "edit"
    input.field == "notes"
    has_role(input.user, "case_manager")
    input.context.resourceStatus == "open"
}

# Rule 14: Cannot edit PII fields (view only)
deny if {
    input.operation == "edit"
    input.fieldMetadata.classification == "pii"
}

##############################################
# CREATE PERMISSIONS (POST requests)
##############################################

# Rule 15: Compliance officers can create cases with all fields
allow if {
    input.operation == "create"
    input.context.resource == "case"
    has_role(input.user, "compliance_officer")
}

# Rule 16: Case managers can create with public/basic fields only
allow if {
    input.operation == "create"
    input.fieldMetadata.classification in ["public", "basic"]
    has_role(input.user, "case_manager")
}

# Rule 17: Senior analysts can create with basic + financial fields
allow if {
    input.operation == "create"
    input.fieldMetadata.classification in ["public", "basic", "financial"]
    has_role(input.user, "senior_analyst")
    input.user.attributes.clearanceLevel == "L3"
}

##############################################
# DELETE PERMISSIONS
##############################################

# Rule 18: Only senior management can delete cases
allow if {
    input.operation == "delete"
    input.context.resource == "case"
    has_role(input.user, "senior_management")
}

##############################################
# MASKING RULES (Data Protection)
##############################################

# Rule 19: Mask SSN for non-compliance officers who can view it
mask_required if {
    input.operation == "view"
    input.field == "customer_ssn"
    allow  # User is allowed to view the field
    not has_role(input.user, "compliance_officer")
    not has_role(input.user, "senior_management")
}

mask_pattern := "XXX-XX-{last4}" if {
    mask_required
    input.field == "customer_ssn"
}

# Rule 20: Mask account numbers for junior analysts
mask_required if {
    input.operation == "view"
    input.field == "account_number"
    has_role(input.user, "junior_analyst")
}

mask_pattern := "****-****-****-{last4}" if {
    mask_required
    input.field == "account_number"
}

# Rule 21: Show account balance as range for viewers
mask_required if {
    input.operation == "view"
    input.field == "account_balance"
    has_role(input.user, "viewer")
    allow
}

mask_pattern := "{range}" if {
    mask_required
    input.field == "account_balance"
}

# Rule 22: Mask email addresses partially
mask_required if {
    input.operation == "view"
    input.field == "customer_email"
    not has_role(input.user, "compliance_officer")
    allow
}

mask_pattern := "{first3}***@{domain}" if {
    mask_required
    input.field == "customer_email"
}

##############################################
# TIME-BASED RULES (Advanced)
##############################################

# Rule 23: Financial data only viewable during business hours
deny if {
    input.operation == "view"
    input.fieldMetadata.classification == "financial"
    not is_business_hours(input.context.timestamp)
    has_role(input.user, "junior_analyst")
}

# Rule 24: Cannot edit cases outside business hours (except compliance)
deny if {
    input.operation == "edit"
    not is_business_hours(input.context.timestamp)
    not has_role(input.user, "compliance_officer")
}

##############################################
# ATTRIBUTE-BASED RULES (ABAC)
##############################################

# Rule 25: Regional data access restriction
deny if {
    input.operation == "view"
    input.context.dataRegion != input.user.attributes.region
    not has_role(input.user, "compliance_officer")  # Global access
}

# Rule 26: High-value cases require higher clearance
deny if {
    input.operation == "view"
    input.fieldMetadata.classification == "financial"
    input.context.caseValue > 100000
    input.user.attributes.clearanceLevel not in ["L2", "L3"]
}

##############################################
# HELPER FUNCTIONS
##############################################

# Check if user has a specific role
has_role(user, role) if {
    user.roles[_] == role
}

# Check if current time is within business hours (9 AM - 6 PM)
is_business_hours(timestamp) if {
    # Parse timestamp and check hour
    hour := time.parse_rfc3339_ns(timestamp)[6]  # Gets hour component
    hour >= 9
    hour < 18
}

# Bulk evaluation for multiple fields at once
# This is used by the Data Filter Service for performance
evaluate_fields[field] := decision if {
    some i
    field := input.fields[i].name
    metadata := input.fields[i].metadata
    
    # Evaluate with current field context
    decision := {
        "allow": allow with input.field as field 
                      with input.fieldMetadata as metadata,
        "mask_required": mask_required with input.field as field 
                                       with input.fieldMetadata as metadata,
        "mask_pattern": mask_pattern with input.field as field 
                                     with input.fieldMetadata as metadata
    }
}

##############################################
# SECTION VISIBILITY RULES (for UI Config)
##############################################

allow_section if {
    input.section == "case_basic_info"
    # Everyone can see basic info
}

allow_section if {
    input.section == "customer_details"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "financial_information"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "financial_information"
    has_role(input.user, "financial_analyst")
    input.user.attributes.clearanceLevel == "L3"
}

allow_section if {
    input.section == "risk_assessment"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "audit_trail"
    has_role(input.user, "compliance_officer")
}

allow_section if {
    input.section == "audit_trail"
    has_role(input.user, "auditor")
}

##############################################
# ACTION PERMISSIONS (for UI Config)
##############################################

allow_action if {
    input.action == "view_case"
    has_role(input.user, "viewer")
}

allow_action if {
    input.action in ["edit_case", "create_case"]
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action in ["edit_case", "create_case"]
    has_role(input.user, "case_manager")
}

allow_action if {
    input.action == "delete_case"
    has_role(input.user, "senior_management")
    input.user.attributes.clearanceLevel == "L3"
}

allow_action if {
    input.action in ["approve_case", "reject_case"]
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "export_report"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "export_report"
    has_role(input.user, "auditor")
}

allow_action if {
    input.action == "assign_case"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "add_notes"
    has_role(input.user, "compliance_officer")
}

allow_action if {
    input.action == "add_notes"
    has_role(input.user, "case_manager")
}

allow_action if {
    input.action == "upload_documents"
    has_role(input.user, "compliance_officer")
}
```

---

## Policy Examples & Use Cases

### **Example 1: Compliance Officer - Full Access**

**Input to OPA:**
```json
{
  "user": {
    "userId": "emp_12345",
    "roles": ["compliance_officer", "senior_staff"],
    "attributes": {
      "clearanceLevel": "L3",
      "region": "north_america"
    }
  },
  "operation": "view",
  "field": "customer_ssn",
  "fieldMetadata": {
    "classification": "pii",
    "is_sensitive": true,
    "is_system_field": false
  },
  "context": {
    "resource": "case",
    "resourceId": "CASE123456",
    "resourceStatus": "open",
    "timestamp": "2025-12-27T14:30:00Z"
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": true,           // ✅ Rule 3 matched
    "mask_required": false,  // ✅ No masking for compliance officers
    "mask_pattern": null
  }
}
```

---

### **Example 2: Analyst - Limited Access with Masking**

**Input to OPA:**
```json
{
  "user": {
    "userId": "emp_67890",
    "roles": ["senior_analyst"],
    "attributes": {
      "clearanceLevel": "L2",
      "region": "north_america"
    }
  },
  "operation": "view",
  "field": "customer_ssn",
  "fieldMetadata": {
    "classification": "pii",
    "is_sensitive": true,
    "is_system_field": false
  },
  "context": {
    "resource": "case",
    "resourceId": "CASE123456",
    "resourceStatus": "open",
    "timestamp": "2025-12-27T14:30:00Z"
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": false,         // ❌ Analyst cannot view PII
    "mask_required": false,
    "mask_pattern": null
  }
}
```

---

### **Example 3: Senior Staff with L3 Clearance - PII Access with Masking**

**Input to OPA:**
```json
{
  "user": {
    "userId": "emp_55555",
    "roles": ["senior_staff"],
    "attributes": {
      "clearanceLevel": "L3",
      "region": "north_america"
    }
  },
  "operation": "view",
  "field": "customer_ssn",
  "fieldMetadata": {
    "classification": "pii",
    "is_sensitive": true,
    "is_system_field": false
  },
  "context": {
    "resource": "case",
    "resourceId": "CASE123456",
    "resourceStatus": "open",
    "timestamp": "2025-12-27T14:30:00Z"
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": true,                   // ✅ Rule 4 matched (senior_staff + L3)
    "mask_required": true,           // ✅ Rule 19 matched (not compliance officer)
    "mask_pattern": "XXX-XX-{last4}" // ✅ Show last 4 digits only
  }
}
```

---

### **Example 4: Batch Evaluation for Multiple Fields**

**Input to OPA (Batch):**
```json
{
  "user": {
    "userId": "emp_12345",
    "roles": ["financial_analyst"],
    "attributes": {
      "clearanceLevel": "L3"
    }
  },
  "operation": "view",
  "context": {
    "resource": "case",
    "resourceId": "CASE123456",
    "resourceStatus": "open",
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
```

**OPA Response (Batch):**
```json
{
  "result": {
    "case_id": {
      "allow": true,          // ✅ Public field
      "mask_required": false,
      "mask_pattern": null
    },
    "customer_ssn": {
      "allow": false,         // ❌ Financial analyst cannot view PII
      "mask_required": false,
      "mask_pattern": null
    },
    "account_balance": {
      "allow": true,          // ✅ Financial analyst with L3 can view
      "mask_required": false,
      "mask_pattern": null
    },
    "risk_score": {
      "allow": false,         // ❌ Only compliance officers can view
      "mask_required": false,
      "mask_pattern": null
    }
  }
}
```

---

### **Example 5: Edit Permission Check**

**Input to OPA:**
```json
{
  "user": {
    "userId": "emp_12345",
    "roles": ["compliance_officer"]
  },
  "operation": "edit",
  "field": "risk_score",
  "fieldMetadata": {
    "classification": "risk_assessment",
    "is_system_field": false
  },
  "context": {
    "resource": "case",
    "resourceId": "CASE123456",
    "resourceStatus": "open"
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": true,          // ✅ Rule 10 matched
    "mask_required": false,
    "mask_pattern": null
  }
}
```

---

### **Example 6: Create Permission Check**

**Input to OPA:**
```json
{
  "user": {
    "userId": "emp_22222",
    "roles": ["case_manager"]
  },
  "operation": "create",
  "field": "customer_ssn",
  "fieldMetadata": {
    "classification": "pii",
    "is_sensitive": true
  },
  "context": {
    "resource": "case"
  }
}
```

**OPA Response:**
```json
{
  "result": {
    "allow": false,  // ❌ Case managers cannot create PII fields
    "mask_required": false,
    "mask_pattern": null
  }
}
```

---

## Summary

### **OPA Policy Coverage**

| Operation | Classification | Roles Allowed | Masking |
|-----------|---------------|---------------|---------|
| **VIEW** | public | Everyone | No |
| **VIEW** | basic | Authenticated | No |
| **VIEW** | pii | Compliance, Senior Staff L2+ | Yes (non-compliance) |
| **VIEW** | financial | Compliance, Analysts L3 | No |
| **VIEW** | risk_assessment | Compliance only | No |
| **EDIT** | basic | Case managers | N/A |
| **EDIT** | risk_assessment | Compliance only | N/A |
| **EDIT** | pii | DENIED | N/A |
| **CREATE** | all | Compliance | N/A |
| **CREATE** | public/basic | Case managers | N/A |
| **DELETE** | case | Senior management only | N/A |

### **Key Features**

✅ **Single source of truth** - All permissions in OPA  
✅ **No database duplication** - Field metadata separate from permissions  
✅ **Batch evaluation** - Performance optimized  
✅ **Dynamic masking** - Based on user role  
✅ **Time-based rules** - Business hours enforcement  
✅ **Attribute-based** - Clearance level, region  
✅ **Audit-friendly** - All decisions logged by OPA  

### **Next Document**

**Part 3**: Java Implementation & Data Filtering Service