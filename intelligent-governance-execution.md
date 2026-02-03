# Intelligence Governance: Complete Implementation Guide v3.0

**Date:** 2026-02-03
**Purpose:** IP-grade conceptual model + execution blueprint for Intelligence Governance
**Audience:** CTO, Technical Leadership, Implementation Teams

---

## Table of Contents

1. [Intelligence Governance: Conceptual Model](#conceptual-model)
2. [Core Principles](#core-principles)
3. [Why This Is Different (Non-Goals)](#non-goals)
4. [Architecture: Three-Tier Defense-in-Depth](#architecture)
5. [Technology Choices & Rationale](#technology-choices)
6. [Implementation Architecture (8 Layers)](#implementation-steps)
7. [Complete End-to-End Flow](#end-to-end-flow)
8. [Agents & MoE: Where They Fit](#agents-moe)
9. [Deployment Architecture & Integration Patterns](#deployment)

---

## 1. Intelligence Governance: Conceptual Model {#conceptual-model}

### **What Is Intelligence Governance?**

**Intelligence Governance is a dedicated control plane that governs what intelligence can see, reason about, and act upon.**

It is **NOT**:

- ❌ Identity & Access Management (IAM) - that's Keycloak
- ❌ Guardrails - that's post-hoc filtering
- ❌ Prompt filtering - that's reactive, not structural
- ❌ Agent autonomy frameworks - that's the opposite of governance

It **IS**:

- ✅ A **control plane** orthogonal to identity, data, and inference
- ✅ **Pre-reasoning** enforcement (not post-hoc filtering)
- ✅ **Structural** access control (not prompt-based)
- ✅ **Graded** decision-making (not binary allow/deny)

---

### **Why Governance Must Precede Reasoning**

**Traditional approach (WRONG):**

```
Retrieve ALL data → Reason → Filter output → Hope nothing leaked
```

**Problems:**

- Unauthorized data already touched by LLM
- Prompt injection can bypass output filters
- No audit trail of what was considered
- Compliance violations (GDPR, HIPAA)

**Intelligence Governance approach (CORRECT):**

```
Governance gates → Retrieve ONLY authorized data → Reason on safe subset → Audit decisions
```

**Benefits:**

- ✅ Unauthorized data NEVER touched
- ✅ Structural enforcement (can't be bypassed by prompts)
- ✅ Complete audit trail
- ✅ Compliance by design

---

### **The Three Planes of HAI Indexer**

```
┌─────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE (Intelligence Governance)                        │
│  ├─ Document Registry (MongoDB)                                 │
│  ├─ ACL Policies (MongoDB)                                      │
│  ├─ Prompt Registry (Spring Cloud Config + Git)                 │
│  ├─ Execution Gate (Python service)                             │
│  └─ Audit Trail (MongoDB)                                       │
├─────────────────────────────────────────────────────────────────┤
│  INTELLIGENCE PLANE (Reasoning & Acting)                        │
│  ├─ Graph Operators (KHop, PPR, Steiner) - ACL-aware            │
│  ├─ Vector Search (Qdrant) - Filter-first                       │
│  ├─ SLM Reasoning (Ollama/Mistral)                              │
│  ├─ MoE Router (Future) - INSIDE SLM, AFTER governance          │
│  └─ Agent Orchestrator (Future) - Proposes, doesn't execute     │
├─────────────────────────────────────────────────────────────────┤
│  DATA PLANE (Storage)                                           │
│  ├─ Qdrant (vectors + retrieval metadata)                       │
│  ├─ Neo4j (graph + ACL nodes)                                   │
│  ├─ MongoDB (governance metadata)                               │
│  ├─ Redis (cache)                                               │
│  └─ Keycloak (RBAC - identity-time authorization)               │
└─────────────────────────────────────────────────────────────────┘
```

**Key insight:** Governance is a **first-class system plane**, not a feature bolted onto IAM or LLM.

---

## 2. Core Principles {#core-principles}

### **Principle 1: Governance Before Intelligence**

**Statement:** Governance decisions happen BEFORE intelligence operations, not after.

**Example:**

```
❌ WRONG: Retrieve all invoices → LLM summarizes → Filter out unauthorized ones
✅ CORRECT: Check ACL → Retrieve ONLY authorized invoices → LLM summarizes
```

**Why:** Unauthorized data must never enter the reasoning context.

---

### **Principle 2: Structure Over Filters**

**Statement:** Access control is structural (graph topology, vector filters), not prompt-based.

**Example:**

```
❌ WRONG: "Don't show HR data to finance users" (in prompt)
✅ CORRECT: ACL nodes in Neo4j prevent graph traversal to HR nodes
```

**Why:** Prompts can be bypassed via injection; graph structure cannot.

---

### **Principle 3: Agents Without Authority**

**Statement:** Agents propose actions; governance approves or denies; execution happens only after approval.

**Example:**

```
❌ WRONG: Agent decides to email invoice → Agent sends email
✅ CORRECT: Agent proposes "email invoice" → Governance checks ACL → Execution gate allows/denies
```

**Why:** Autonomous agents create ungovernable risk.

---

### **Principle 4: Graded Outcomes, Not Binary**

**Statement:** Governance decisions are graded (ALLOW / REDACT / READ_ONLY / REQUIRE_HUMAN / DENY), not binary.

**Example:**

```
❌ WRONG: Alice can access invoice → Yes/No
✅ CORRECT: Alice can READ invoice, can SUMMARIZE, cannot EXPORT, cannot EMAIL
```

**Why:** Real-world authorization is nuanced, not binary.

---

### **Principle 5: Learning Without Drift**

**Statement:** Governance rules are explicit and versioned, not learned implicitly by models.

**Example:**

```
❌ WRONG: Fine-tune LLM to "learn" access patterns
✅ CORRECT: Store ACL policies in MongoDB, version them, audit changes
```

**Why:** Implicit learning creates ungovernable drift and compliance risk.

---

### **Principle 6: MoE Is Not Governance**

**Statement:** Mixture of Experts (MoE) lives INSIDE the SLM layer, AFTER governance gates.

**Example:**

```
Flow: Governance (ACL check) → Governed prompt → MoE Router → Finance Expert → Response
```

**Why:** MoE improves reasoning quality; governance controls what can be reasoned about.

---

## 3. Why This Is Different (Non-Goals) {#non-goals}

### **What We Are NOT Doing**

| Anti-Pattern                         | Why We Reject It                       | What We Do Instead                         |
| ------------------------------------ | -------------------------------------- | ------------------------------------------ |
| **Encoding policy in prompts**       | Prompts can be bypassed via injection  | ACL nodes in graph, structural enforcement |
| **Allowing autonomous agents**       | Creates ungovernable execution paths   | Agents propose, governance approves        |
| **Letting MoE bypass policy**        | MoE would become a governance backdoor | Governance gates BEFORE MoE                |
| **Training governance into weights** | Creates drift, no audit trail          | Explicit policies in MongoDB, versioned    |
| **Post-hoc output filtering**        | Data already leaked to LLM context     | Pre-retrieval filtering (filter-first)     |
| **Binary allow/deny**                | Real-world authorization is nuanced    | Graded outcomes (5 levels)                 |

---

## 4. Architecture: Three-Tier Defense-in-Depth {#architecture}

### **Overview**

Intelligence Governance implements **three tiers** of enforcement:

```
┌─────────────────────────────────────────────────────────────────┐
│  TIER 1: Pre-Retrieval Governance (Graph-Native ACL)            │
│  ├─ ACL as first-class nodes in Neo4j                           │
│  ├─ Graph traversal respects ACL edges                          │
│  ├─ Unauthorized nodes NEVER visited                            │
│  └─ Operators (KHop/PPR/Steiner) are ACL-aware                  │
├─────────────────────────────────────────────────────────────────┤
│  TIER 2: Post-Retrieval Execution Gate                          │
│  ├─ Operability scoring (confidence, freshness, sensitivity)    │
│  ├─ Policy gating (check permissions for requested actions)     │
│  ├─ Graded outcomes (ALLOW / REDACT / READ_ONLY / etc.)         │
│  └─ Action restrictions (can summarize, cannot email)           │
├─────────────────────────────────────────────────────────────────┤
│  TIER 3: SLM-Layer MoE (Future)                                 │
│  ├─ Mixture of Experts routing (INSIDE SLM)                     │
│  ├─ Domain-specific experts (Finance, HR, Legal)                │
│  ├─ Governed prompt selection (from Spring Cloud Config)        │
│  └─ MoE improves reasoning, does NOT bypass governance          │
└─────────────────────────────────────────────────────────────────┘
```

---

### **TIER 1: Pre-Retrieval Governance (Graph-Native ACL)**

**Concept:** ACL is not metadata on documents; ACL is a **first-class graph node** that participates in traversal.

**Example Graph Structure:**

```cypher
// Document node
(doc:Document {doc_id: "invoice-001", domain: "FINANCE"})

// ACL policy node
(acl:ACLPolicy {
  acl_id: "acl-finance-internal",
  allowed_roles: ["FINANCE_USER", "FINANCE_MANAGER"],
  allowed_users: ["alice@company.com"]
})

// Relationship
(doc)-[:GOVERNED_BY]->(acl)
```

**ACL-Aware Traversal (Conceptual):**

```
When doing K-hop traversal:
1. Start from seed nodes
2. For each hop:
   - Check if target node has ACL
   - If yes: Check if user's roles/email match ACL
   - If no match: SKIP this node (structural pruning)
   - If match: Include in result
3. Return only authorized nodes
```

**Key insight:** Unauthorized nodes are **structurally pruned** during traversal, not filtered after retrieval.

---

### **TIER 2: Post-Retrieval Execution Gate**

**Concept:** User can SEE data, but what can they DO with it?

**Graded Outcomes:**

| Decision                 | Meaning                                   | Example                                |
| ------------------------ | ----------------------------------------- | -------------------------------------- |
| **ALLOW**                | Full access, all actions permitted        | Finance manager accessing invoices     |
| **ALLOW_WITH_REDACTION** | Access granted, sensitive fields masked   | Contractor viewing salary (SSN masked) |
| **READ_ONLY**            | Can view/summarize, cannot export/email   | Finance user can summarize, not export |
| **REQUIRE_HUMAN**        | Requires human approval before proceeding | High-risk action on RESTRICTED data    |
| **DENY**                 | No access                                 | HR data accessed by finance user       |

**Example Decision:**

Alice requests: `["SUMMARIZE", "EMAIL"]`

ACL policy says:

```json
{
  "permissions": {
    "read": true,
    "summarize": true,
    "export": false,
    "email": false
  }
}
```

Execution Gate decision:

```json
{
  "decision": "READ_ONLY",
  "allowed_actions": ["read", "summarize"],
  "blocked_actions": ["EMAIL"],
  "reason": "Email permission denied by ACL policy"
}
```

Response to Alice:

```
✅ Summary generated successfully!
❌ Email action blocked: You don't have permission to email finance documents.
   Contact your manager to request access.
```

---

### **TIER 3: SLM-Layer MoE (Future)**

**Concept:** MoE lives INSIDE the SLM, AFTER governance gates.

**Flow:**

```
1. Governance selects prompt (from Spring Cloud Config)
2. Governance filters data (ACL-aware)
3. Governed prompt + authorized data → MoE Router
4. MoE Router selects expert (Finance Expert, Language Expert, etc.)
5. Expert generates response
6. Response returned (no governance bypass)
```

**Key insight:** MoE improves **how** reasoning happens; governance controls **what** can be reasoned about.

---

## 5. Technology Choices & Rationale {#technology-choices}

### **5.1 MongoDB for Governance Control Plane**

**Decision:** Use MongoDB for `document_registry`, `acl_policies`, `audit_log`

**Why MongoDB wins:**

| Criterion               | MongoDB                                | PostgreSQL                          |
| ----------------------- | -------------------------------------- | ----------------------------------- |
| **Data shape**          | ✅ Policy-shaped (nested documents)    | ❌ Forces normalization (5+ tables) |
| **Change velocity**     | ✅ No migrations for schema evolution  | ❌ ALTER TABLE for every change     |
| **Query pattern**       | ✅ Key/context lookup (99% of queries) | ❌ Optimized for relational joins   |
| **Atomic updates**      | ✅ Single-document ACID sufficient     | ⚠️ Multi-table ACID overkill        |
| **Tenant partitioning** | ✅ Native sharding support             | ❌ Manual partition creation        |

**Example policy document:**

```json
{
  "_id": "acl-finance-internal",
  "tenant_id": "tenant_xyz",
  "domain_scope": "FINANCE_*",
  "allowed_roles": ["FINANCE_USER", "FINANCE_MANAGER"],
  "permissions": {
    "read": true,
    "summarize": true,
    "export": false,
    "email": false
  },
  "redaction_profiles": {
    "CONTRACTOR": {
      "fields_to_redact": ["salary", "ssn"],
      "redaction_method": "MASK"
    }
  },
  "version": 2,
  "status": "ACTIVE"
}
```

**When PostgreSQL comes in (Phase 2):**

- Complex relational analytics
- Compliance reports
- BI dashboards
- Flow: MongoDB (operational) → ETL → PostgreSQL (analytics)

---

### **5.2 Spring Cloud Config for Prompt Registry**

**Decision:** Use Spring Cloud Config (backed by Git) for prompt registry, NOT MongoDB

**Why Spring Cloud Config wins:**

| Criterion           | Spring Cloud Config + Git       | MongoDB                    |
| ------------------- | ------------------------------- | -------------------------- |
| **Version control** | ✅ Git history, branches, tags  | ❌ Manual versioning       |
| **Hot refresh**     | ✅ `/actuator/refresh` endpoint | ❌ Requires app restart    |
| **Audit trail**     | ✅ Git commits, blame, diffs    | ⚠️ Custom audit log needed |
| **Rollback**        | ✅ `git revert`                 | ❌ Manual rollback logic   |
| **A/B testing**     | ✅ Feature flags + branches     | ❌ Custom logic needed     |
| **Collaboration**   | ✅ Pull requests, code review   | ❌ Direct DB edits         |

**Example prompt in Git:**

**File:** `prompts/finance/invoice-summary-v1.yaml`

```yaml
id: finance.invoice.summary.v1
name: Finance Invoice Summary
description: Summarize finance invoices focusing on totals and anomalies
version: 1.0.0

# Governance
domain: FINANCE
intent: SUMMARIZE
risk_level: MEDIUM

# Prompt Template
template: |
  You are a financial analyst assistant.

  Summarize the following invoice documents:

  {context}

  Focus on:
  1. Total amounts
  2. Unusual patterns or anomalies
  3. Key vendors

  Do NOT include:
  - Personal data (SSN, addresses)
  - Bank account numbers
  - Internal cost codes

  Format your response as a concise summary with bullet points.

# Constraints
max_tokens: 500
temperature: 0.3
allowed_tools:
  - calculator
  - date_parser
forbidden_tools:
  - web_search
  - email_sender

# Safety
safety_checks:
  pii_detection: true
  toxicity_filter: true
  hallucination_check: true

status: ACTIVE
```

**Hot refresh flow:**

```
1. Developer updates prompt in Git
2. Commits + pushes to main branch
3. Spring Cloud Config detects change
4. HAI Indexer calls /actuator/refresh
5. New prompt loaded WITHOUT restart
6. Git commit = audit trail
```

**Key insight:** Prompts are code, not data. They should be versioned, reviewed, and deployed like code.

---

### **5.3 RBAC vs ACL: Keycloak vs MongoDB**

**Critical distinction:**

| Concern                           | System            | Owns                 | Answers                      |
| --------------------------------- | ----------------- | -------------------- | ---------------------------- |
| **Identity**                      | Keycloak          | users, roles, groups | "Who are you?"               |
| **Authorization (identity-time)** | Keycloak          | RBAC                 | "What role do you have?"     |
| **Authorization (data-time)**     | MongoDB           | ACLs                 | "What does this data allow?" |
| **Policy evaluation**             | Governance Engine | decisions            | "Can you do this?"           |
| **Enforcement**                   | HAI Indexer       | filtering, gating    | "Execute or block"           |

**Example flow:**

Alice queries invoices:

**Step 1 - Authentication (Keycloak):**

```json
{
  "user": "alice@company.com",
  "roles": ["FINANCE_USER", "EMPLOYEE"]
}
```

**Step 2 - Document lookup (MongoDB):**

```json
{
  "document_id": "doc-001",
  "domain": "FINANCE_INVOICE",
  "acl_policy_id": "acl-finance-internal"
}
```

**Step 3 - ACL evaluation (MongoDB):**

```json
{
  "acl_id": "acl-finance-internal",
  "allowed_roles": ["FINANCE_USER", "FINANCE_MANAGER"],
  "permissions": ["READ", "SUMMARIZE"]
}
```

**Step 4 - Combine RBAC + ACL:**

```
User roles (from Keycloak): FINANCE_USER
Allowed roles (from ACL): FINANCE_USER, FINANCE_MANAGER
✅ Match → allow
```

**Key insight:**

- Keycloak never sees the document
- MongoDB never sees the user password
- Clean separation of concerns

---

## 6. Implementation Architecture {#implementation-steps}

### **Overview: Eight Layers of Intelligence Governance**

This section details the complete architecture with examples for each layer, including MoE and Agent orchestration.

---

### **STEP 1: Document Control Plane**

**Problem:** No central registry of what documents exist, their classification, and governance policies.

**Solution:** Create `document_registry` collection in MongoDB as the source of truth for all governance metadata.

**Critical Clarification: What We Store vs What We Don't**

> **We don't store documents.** Documents live in Google Drive, Git, or external systems. What we store is **governance metadata**: document identity, classification, domain, and the ACL policy it maps to. ACLs are not per-document schemas; they are **reusable policy artifacts**. A hundred finance invoices can all reference the same ACL policy. MongoDB is chosen because policies evolve faster than relational schemas, and we need versioned, nested, condition-based rules. PostgreSQL can come later for analytics, but governance needs agility, not joins.

**MongoDB Collection Schema:**

```javascript
// Collection: document_registry
{
  _id: "doc-001",
  tenant_id: "tenant_xyz",
  file_id: "invoice-jan-2024.pdf",

  // Document classification
  domain: "FINANCE_INVOICE",
  classification: "RESTRICTED",  // PUBLIC, INTERNAL, RESTRICTED, CONFIDENTIAL
  sensitivity_score: 0.75,       // 0.0 (public) to 1.0 (highly sensitive)

  // Governance linkage
  acl_policy_id: "acl-finance-internal",

  // Metadata for retrieval
  metadata: {
    vendor: "Acme Corp",
    amount: 45230.00,
    invoice_date: "2024-01-15",
    department: "FINANCE"
  },

  // Audit trail
  indexed_at: "2024-01-15T10:30:00Z",
  indexed_by: "system",
  last_accessed_at: "2024-01-20T14:22:00Z",
  last_accessed_by: "alice@company.com",
  access_count: 12,

  // Status
  status: "ACTIVE"  // ACTIVE, ARCHIVED, QUARANTINED, DELETED
}
```

**Automated Ingestion Lifecycle (Detailed Flow)**

This diagram shows exactly what happens when Google Drive watcher fires and a new document arrives:

```
┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 1: DOCUMENT ARRIVAL (Google Drive Watcher)                       │
└─────────────────────────────────────────────────────────────────────────┘
  Google Drive watcher detects: invoice-jan-2024.pdf
    ↓
  Trigger: HAI Indexer ingestion pipeline
    ↓
  Download document to temporary storage

┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 2: METADATA EXTRACTION (Created vs Inferred)                     │
└─────────────────────────────────────────────────────────────────────────┘
  Step 1: Extract CREATED metadata (from file system)
    - file_id: "invoice-jan-2024.pdf"
    - file_name: "invoice-jan-2024.pdf"
    - file_size: 245KB
    - created_at: "2024-01-15T10:30:00Z"
    - mime_type: "application/pdf"
    ↓
  Step 2: Extract INFERRED metadata (via LLM)
    - domain: "FINANCE_INVOICE" (detected from content)
    - vendor: "Acme Corp" (extracted from invoice header)
    - amount: $45,230 (extracted from invoice total)
    - invoice_date: "2024-01-15" (extracted from invoice)
    - department: "FINANCE" (inferred from content)
    ↓
  Step 3: Classify document (rule-based)
    - classification: "RESTRICTED" (based on domain + amount threshold)
    - sensitivity_score: 0.75 (calculated from classification rules)

┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 3: ACL POLICY ATTACHMENT (Rule-Based vs Manual)                  │
└─────────────────────────────────────────────────────────────────────────┘
  Step 4: Determine ACL policy (AUTOMATED - rule-based)
    - Match domain pattern: FINANCE_* → acl-finance-internal
    - Match classification: RESTRICTED → requires acl-finance-internal
    - Result: acl_policy_id = "acl-finance-internal"
    ↓
  Alternative: Manual override (if needed)
    - Admin can manually assign different ACL policy
    - Example: High-value invoice → acl-finance-executive-only
    ↓
  Step 5: Validate ACL policy exists
    - Query MongoDB: acl_policies.findOne({_id: "acl-finance-internal"})
    - If not found → ERROR: "ACL policy not found"
    - If found → Proceed

┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 4: GOVERNANCE METADATA STORAGE (MongoDB)                         │
└─────────────────────────────────────────────────────────────────────────┘
  Step 6: Insert into document_registry (MongoDB)
    {
      "_id": "doc-001",
      "tenant_id": "tenant_xyz",
      "file_id": "invoice-jan-2024.pdf",
      "domain": "FINANCE_INVOICE",
      "classification": "RESTRICTED",
      "sensitivity_score": 0.75,
      "acl_policy_id": "acl-finance-internal",  ← REFERENCES reusable policy
      "metadata": {
        "vendor": "Acme Corp",
        "amount": 45230.00,
        "invoice_date": "2024-01-15",
        "department": "FINANCE"
      },
      "indexed_at": "2024-01-15T10:30:00Z",
      "status": "ACTIVE"
    }
    ↓
  Result: Governance metadata stored (document content NOT stored)

┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 5: VECTOR + GRAPH SYNC (Qdrant + Neo4j)                          │
└─────────────────────────────────────────────────────────────────────────┘
  Step 7: Chunk document and index in Qdrant
    - Split document into chunks (512 tokens each)
    - Generate embeddings (nomic-embed-text via Ollama)
    - Index each chunk with ACL metadata:
      {
        "id": "chunk-001",
        "vector": [0.123, 0.456, ...],
        "payload": {
          "doc_id": "doc-001",
          "tenant_id": "tenant_xyz",
          "principals": ["FINANCE_USER", "FINANCE_MANAGER", "CFO"],  ← From ACL
          "text": "Invoice from Acme Corp..."
        }
      }
    ↓
  Step 8: Create graph nodes in Neo4j
    - Create document node:
      CREATE (doc:Document {
        doc_id: "doc-001",
        file_id: "invoice-jan-2024.pdf",
        domain: "FINANCE_INVOICE"
      })
    ↓
    - Create ACL policy node (if not exists):
      MERGE (acl:ACLPolicy {acl_id: "acl-finance-internal"})
    ↓
    - Link document to ACL policy:
      CREATE (doc)-[:GOVERNED_BY]->(acl)
    ↓
    - Extract entities and create relationships:
      CREATE (vendor:Vendor {name: "Acme Corp"})
      CREATE (doc)-[:VENDOR]->(vendor)
    ↓
  Step 9: Sync complete
    - MongoDB: Governance metadata ✅
    - Qdrant: Vector embeddings with ACL metadata ✅
    - Neo4j: Graph nodes with ACL relationships ✅

┌─────────────────────────────────────────────────────────────────────────┐
│ PHASE 6: DOCUMENT NOW GOVERNED AND SEARCHABLE                          │
└─────────────────────────────────────────────────────────────────────────┘
  Document lifecycle complete:
    - Governance metadata: MongoDB (source of truth)
    - Vector search: Qdrant (with ACL filtering)
    - Graph traversal: Neo4j (with ACL nodes)
    - Document content: Google Drive (original location)
```

**Key Insights:**

1. **Created vs Inferred Metadata:**
   - **Created**: File system metadata (file_id, size, timestamps) - no LLM needed
   - **Inferred**: Content metadata (domain, vendor, amount) - requires LLM extraction

2. **ACL Attachment:**
   - **Rule-based (automated)**: Domain pattern matching (FINANCE\_\* → acl-finance-internal)
   - **Manual override**: Admin can assign different policy for exceptions

3. **When Sync Happens:**
   - **MongoDB first**: Governance metadata stored (source of truth)
   - **Qdrant second**: Vector embeddings indexed with ACL metadata from MongoDB
   - **Neo4j third**: Graph nodes created with ACL relationships from MongoDB

4. **Single Source of Truth:**
   - MongoDB document_registry is authoritative for governance metadata
   - Qdrant and Neo4j sync FROM MongoDB, not independently

---

### **STEP 2: ACL Policy Store**

**Problem:** Access control rules are hardcoded, scattered, or embedded in application logic.

**Solution:** Externalize all ACL policies to MongoDB with versioning, conditional rules, and audit trail.

**Critical Clarification: Policy Scope vs Document Schema**

> **ACLs are reusable policy artifacts, not per-document schemas.** A hundred finance invoices can all reference the same ACL policy (`acl-finance-internal`). Policies are **domain-scoped** (e.g., `FINANCE_*`), not document-shaped. Different document schemas (invoice vs expense report) do NOT require different ACL schemas. The policy defines WHO can access WHAT actions on documents matching a domain pattern. This is why MongoDB is ideal: policies evolve independently of document structure, without migrations.

**Example: Policy Reuse**

```
Document 1: invoice-jan-2024.pdf → acl_policy_id: "acl-finance-internal"
Document 2: invoice-feb-2024.pdf → acl_policy_id: "acl-finance-internal"
Document 3: invoice-mar-2024.pdf → acl_policy_id: "acl-finance-internal"
...
Document 100: invoice-dec-2024.pdf → acl_policy_id: "acl-finance-internal"

All 100 invoices share ONE policy artifact.
Policy change affects all 100 documents instantly (no per-document updates).
```

**MongoDB Collection Schema:**

```javascript
// Collection: acl_policies
{
  _id: "acl-finance-internal",
  tenant_id: "tenant_xyz",

  // Policy scope
  name: "Finance Internal Documents Policy",
  description: "Access control for internal finance documents",
  domain_scope: "FINANCE_*",  // Applies to all FINANCE domains

  // Access control
  allowed_roles: ["FINANCE_USER", "FINANCE_MANAGER", "CFO"],
  allowed_users: ["alice@company.com", "bob@company.com"],
  denied_users: ["contractor@external.com"],  // Explicit deny overrides allow

  // Permissions (graded, not binary)
  permissions: {
    read: true,
    summarize: true,
    extract: true,
    export: false,      // Cannot export to CSV/PDF
    email: false,       // Cannot email documents
    print: false,       // Cannot print
    share: false        // Cannot share with external users
  },

  // Redaction profiles (role-based)
  redaction_profiles: {
    "CONTRACTOR": {
      fields_to_redact: ["salary", "ssn", "bank_account"],
      redaction_method: "MASK"  // MASK, HASH, REMOVE
    },
    "INTERN": {
      fields_to_redact: ["amount", "vendor_details"],
      redaction_method: "HASH"
    }
  },

  // Conditional rules (advanced)
  conditional_rules: [
    {
      condition: "user.department == 'HR' AND document.amount < 10000",
      action: "ALLOW_FULL_ACCESS"
    },
    {
      condition: "user.role == 'AUDITOR' AND document.age_days > 90",
      action: "ALLOW_READ_ONLY"
    }
  ],

  // Time-based access
  time_restrictions: {
    business_hours_only: true,
    allowed_hours: "09:00-17:00",
    allowed_days: ["MON", "TUE", "WED", "THU", "FRI"],
    timezone: "America/New_York"
  },

  // Versioning
  version: 3,
  created_at: "2024-01-01T00:00:00Z",
  updated_at: "2024-01-15T10:00:00Z",
  updated_by: "admin@company.com",

  // Status
  status: "ACTIVE",  // ACTIVE, DEPRECATED, ARCHIVED

  // Audit
  change_log: [
    {
      version: 1,
      changes: "Initial policy creation",
      changed_by: "admin@company.com",
      changed_at: "2024-01-01T00:00:00Z"
    },
    {
      version: 2,
      changes: "Added redaction profiles for contractors",
      changed_by: "admin@company.com",
      changed_at: "2024-01-10T12:00:00Z"
    },
    {
      version: 3,
      changes: "Disabled email and export permissions",
      changed_by: "admin@company.com",
      changed_at: "2024-01-15T10:00:00Z"
    }
  ]
}
```

**Example: Policy Evolution (No Migrations!)**

```
Week 1: Simple policy
{
  "allowed_roles": ["FINANCE_USER"],
  "permissions": {"read": true}
}

Week 2: Add redaction (no migration!)
{
  "allowed_roles": ["FINANCE_USER"],
  "permissions": {"read": true},
  "redaction_profiles": {
    "CONTRACTOR": {"fields_to_redact": ["salary"]}
  }
}

Week 3: Add time restrictions (no migration!)
{
  "allowed_roles": ["FINANCE_USER"],
  "permissions": {"read": true},
  "redaction_profiles": {...},
  "time_restrictions": {"business_hours_only": true}
}
```

**Key Insight:** MongoDB's schema flexibility allows governance semantics to evolve without migrations.

---

### **STEP 3: Pre-Retrieval Governance (Graph-Native ACL)**

**Problem:** Graph traversal can leak unauthorized data by visiting nodes user shouldn't see.

**Solution:** Make ACL a first-class graph node; modify operators to respect ACL during traversal.

**Graph structure:**

```cypher
// Create ACL node
CREATE (acl:ACLPolicy {
  acl_id: "acl-finance-internal",
  allowed_roles: ["FINANCE_USER", "FINANCE_MANAGER"],
  allowed_users: ["alice@company.com"]
})

// Link document to ACL
MATCH (doc:Document {doc_id: "invoice-001"})
MATCH (acl:ACLPolicy {acl_id: "acl-finance-internal"})
CREATE (doc)-[:GOVERNED_BY]->(acl)
```

**ACL-aware traversal (conceptual):**

```
For each node in traversal path:
  IF node has GOVERNED_BY relationship:
    Get ACL policy
    Check if user's roles/email match allowed_roles/allowed_users
    IF no match: SKIP this node (structural pruning)
  ELSE:
    Include node (no ACL = public)
```

**Integration with HAI Indexer:**

- Modify `KHopOperator`, `PPROperator`, `SteinerOperator` to check ACL
- Add ACL filter to Qdrant vector search
- Result: Unauthorized data NEVER enters retrieval context

---

### **STEP 4: Post-Retrieval Execution Gate**

**Problem:** User can see data, but what actions can they perform?

**Solution:** Create `ExecutionGate` service that evaluates permissions for requested actions.

**Example evaluation:**

Alice requests: `["SUMMARIZE", "EMAIL"]`

```
ExecutionGate logic:
1. Get user context (roles, email) from JWT
2. Get document ACL policy from MongoDB
3. Check permissions for each requested action:
   - SUMMARIZE: allowed_actions contains "summarize" → ✅ ALLOW
   - EMAIL: allowed_actions does NOT contain "email" → ❌ DENY
4. Return graded decision: READ_ONLY
```

**Response structure:**

```json
{
  "decision": "READ_ONLY",
  "allowed_actions": ["read", "summarize"],
  "blocked_actions": ["email"],
  "redaction_required": false,
  "reason": "Email permission denied by ACL policy acl-finance-internal"
}
```

**Integration with HAI Indexer:**

- Call ExecutionGate BEFORE executing any action
- If action blocked, return error to user
- Log decision to audit trail

---

### **STEP 5: Governed Prompt Registry**

**Problem:** Prompts are hardcoded, no version control, no audit trail.

**Solution:** Externalize prompts to Spring Cloud Config backed by Git.

**Git repository structure:**

```
prompts-repo/
├── finance/
│   ├── invoice-summary-v1.yaml
│   ├── expense-analysis-v1.yaml
│   └── budget-forecast-v1.yaml
├── hr/
│   ├── employee-lookup-v1.yaml
│   └── policy-qa-v1.yaml
└── legal/
    └── contract-review-v1.yaml
```

**Prompt selection logic:**

```
PromptSelector logic:
1. Parse user query → extract domain + intent
   Example: "Summarize invoices" → domain=FINANCE, intent=SUMMARIZE
2. Query Spring Cloud Config:
   GET /prompts/finance/invoice-summary-v1.yaml
3. Render template with context:
   template.replace("{context}", authorized_chunks)
4. Return governed prompt to SLM
```

**Hot refresh:**

```
Developer updates prompt → Git commit → Spring Cloud Config refresh →
HAI Indexer reloads → New prompt active (no restart)
```

**Integration with HAI Indexer:**

- Replace hardcoded prompts in `AnswerGenerator`
- Add PromptSelector service
- Configure Spring Cloud Config client

---

### **STEP 6: Agent Orchestration**

**Problem:** Traditional agentic AI allows agents to execute actions autonomously, creating ungovernable execution paths.

**Solution:** Agents propose actions; governance evaluates and approves/denies; execution happens only after approval.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  AGENT LAYER (Proposal Generators)                             │
│  ├─ Task Planning Agent                                         │
│  ├─ Data Retrieval Agent                                        │
│  ├─ Action Execution Agent                                      │
│  └─ Communication Agent                                         │
├─────────────────────────────────────────────────────────────────┤
│  GOVERNANCE LAYER (Approval Authority)                          │
│  ├─ Proposal Evaluator                                          │
│  ├─ ACL Checker                                                 │
│  ├─ Risk Assessor                                               │
│  └─ Audit Logger                                                │
├─────────────────────────────────────────────────────────────────┤
│  EXECUTION LAYER (Controlled Execution)                         │
│  ├─ Approved Action Executor                                    │
│  ├─ Fallback Handler                                            │
│  └─ User Notification                                           │
└─────────────────────────────────────────────────────────────────┘
```

**Example 1: Email Action Proposal**

```
User query: "Summarize January invoices and email to my manager"
  ↓
Agent analyzes query:
  - Intent 1: SUMMARIZE invoices
  - Intent 2: EMAIL summary
  ↓
Agent proposes actions:
{
  "proposals": [
    {
      "action_id": "action-001",
      "action_type": "SUMMARIZE",
      "target": "FINANCE_INVOICE",
      "filter": {"month": "January"},
      "risk_level": "LOW"
    },
    {
      "action_id": "action-002",
      "action_type": "EMAIL",
      "target": "manager@company.com",
      "content": "{{summary_from_action-001}}",
      "risk_level": "MEDIUM"
    }
  ]
}
  ↓
Governance evaluates action-001 (SUMMARIZE):
  - Check ACL: permissions.summarize = true → ✅ APPROVE
  ↓
Governance evaluates action-002 (EMAIL):
  - Check ACL: permissions.email = false → ❌ DENY
  ↓
Governance decision:
{
  "action-001": {
    "decision": "APPROVED",
    "reason": "User has summarize permission"
  },
  "action-002": {
    "decision": "DENIED",
    "reason": "Email permission denied by ACL policy acl-finance-internal"
  }
}
  ↓
Execution:
  - Execute action-001: Generate summary ✅
  - Block action-002: Do not send email ❌
  ↓
Response to user:
"✅ Summary generated successfully!
 ❌ Email action blocked: You don't have permission to email finance documents.
    Contact your manager to request access."
```

**Example 2: Multi-Step Agent Workflow**

```
User query: "Find all invoices from Acme Corp, calculate total, and create a report"
  ↓
Agent proposes workflow:
{
  "workflow_id": "wf-001",
  "steps": [
    {
      "step": 1,
      "action": "SEARCH",
      "query": "vendor:Acme Corp",
      "domain": "FINANCE_INVOICE"
    },
    {
      "step": 2,
      "action": "CALCULATE",
      "operation": "SUM",
      "field": "amount"
    },
    {
      "step": 3,
      "action": "GENERATE_REPORT",
      "format": "PDF",
      "template": "invoice_summary"
    }
  ]
}
  ↓
Governance evaluates each step:
  - Step 1 (SEARCH): Check ACL for FINANCE_INVOICE → ✅ APPROVE
  - Step 2 (CALCULATE): Check permissions.extract → ✅ APPROVE
  - Step 3 (GENERATE_REPORT): Check permissions.export → ❌ DENY (export=false)
  ↓
Governance decision:
{
  "workflow_decision": "PARTIAL_APPROVAL",
  "approved_steps": [1, 2],
  "denied_steps": [3],
  "alternative": "Display results in UI instead of PDF export"
}
  ↓
Execution:
  - Execute steps 1-2: Search and calculate ✅
  - Block step 3: No PDF export ❌
  - Fallback: Display results in UI ✅
```

**Key Principle:** Agents never own execution. Agents propose; governance disposes.

---

### **STEP 7: MoE Integration (Mixture of Experts)**

**Problem:** Single LLM model cannot excel at all tasks (finance, legal, code, language, etc.).

**Solution:** Use Mixture of Experts (MoE) to route queries to domain-specific expert models, but AFTER governance gates.

**Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  GOVERNANCE LAYER (BEFORE MoE)                                  │
│  ├─ ACL filtering (only authorized data)                        │
│  ├─ Execution gating (check permissions)                        │
│  └─ Prompt selection (governed prompts from Git)                │
├─────────────────────────────────────────────────────────────────┤
│  MoE ROUTER (INSIDE SLM LAYER)                                  │
│  ├─ Analyzes: domain, intent, complexity                        │
│  ├─ Selects: appropriate expert model                           │
│  └─ Routes: governed prompt + authorized data → expert          │
├─────────────────────────────────────────────────────────────────┤
│  EXPERT MODELS                                                  │
│  ├─ Finance Expert (mistral:7b-finance)                         │
│  ├─ Legal Expert (mistral:7b-legal)                             │
│  ├─ Code Expert (mistral:7b-code)                               │
│  ├─ Language Expert (mistral:7b-language)                       │
│  └─ Explanation Expert (mistral:7b-explain)                     │
└─────────────────────────────────────────────────────────────────┘
```

**Example 1: Finance Query Routing**

```
User query: "Summarize January invoices and highlight anomalies"
  ↓
1. Governance filters data:
   - ACL check: User has FINANCE_USER role → ✅ allowed
   - Retrieve only authorized invoices
   - Result: 15 invoices (3 blocked by ACL)
  ↓
2. Governance selects prompt:
   - Domain: FINANCE
   - Intent: SUMMARIZE
   - Prompt: finance.invoice.summary.v1 (from Git)
  ↓
3. MoE Router analyzes:
   {
     "domain": "FINANCE",
     "intent": "SUMMARIZE",
     "complexity": "MEDIUM",
     "requires_calculation": true,
     "requires_anomaly_detection": true
   }
  ↓
4. MoE Router selects expert:
   - Domain=FINANCE → Finance Expert (mistral:7b-finance)
   - Reason: Specialized in financial analysis and anomaly detection
  ↓
5. Finance Expert processes:
   Input: Governed prompt + 15 authorized invoices
   Output: Summary with anomalies highlighted
  ↓
6. Response:
   "January 2026 Invoice Summary:
    - Total: $1.2M across 15 invoices
    - Top vendor: Acme Corp ($450K, 37% of total)
    - Anomalies detected:
      * Invoice #003: $180K (3x average, requires review)
      * Invoice #007: Duplicate vendor entry
      * Invoice #012: Payment terms unusual (Net 90 vs standard Net 30)"
```

**Example 2: Multi-Domain Query Routing**

```
User query: "Explain the legal implications of the Q4 budget overrun mentioned in the finance report"
  ↓
1. Governance filters data:
   - ACL check for FINANCE domain → ✅ allowed
   - ACL check for LEGAL domain → ✅ allowed
  ↓
2. MoE Router analyzes:
   {
     "domains": ["FINANCE", "LEGAL"],
     "intent": "EXPLAIN",
     "complexity": "HIGH",
     "requires_multi_domain_reasoning": true
   }
  ↓
3. MoE Router orchestrates multi-expert workflow:
   Step 1: Finance Expert analyzes budget data
     → Output: "Q4 overrun: $2.3M (15% over budget)"

   Step 2: Legal Expert analyzes implications
     → Input: Finance Expert output + legal policies
     → Output: "Legal implications: Requires board notification per policy FIN-001..."

   Step 3: Explanation Expert synthesizes
     → Input: Finance + Legal outputs
     → Output: Comprehensive explanation
  ↓
4. Response:
   "The Q4 budget overrun of $2.3M (15% over budget) triggers the following legal requirements:
    1. Board notification required within 10 business days (Policy FIN-001)
    2. Audit committee review mandatory (SOX compliance)
    3. Revised forecast submission to regulators (if public company)

    Recommended actions:
    - Schedule emergency board meeting
    - Prepare variance analysis report
    - Consult external auditors"
```

**Example 3: MoE Router Decision Logic**

```javascript
// MoE Router logic (conceptual, not production code)
function selectExpert(query, context) {
  // Analyze query characteristics
  const analysis = {
    domain: extractDomain(query), // FINANCE, LEGAL, HR, etc.
    intent: extractIntent(query), // SUMMARIZE, EXPLAIN, EXTRACT, etc.
    complexity: assessComplexity(query), // LOW, MEDIUM, HIGH
    requires_calculation: hasNumbers(query),
    requires_code: hasCodeKeywords(query),
    requires_multi_domain: hasMultipleDomains(query),
  };

  // Expert selection rules
  if (analysis.domain === "FINANCE" && analysis.requires_calculation) {
    return "finance_expert"; // mistral:7b-finance
  }

  if (analysis.domain === "LEGAL") {
    return "legal_expert"; // mistral:7b-legal
  }

  if (analysis.requires_code) {
    return "code_expert"; // mistral:7b-code
  }

  if (analysis.intent === "EXPLAIN" && analysis.complexity === "HIGH") {
    return "explanation_expert"; // mistral:7b-explain
  }

  if (analysis.requires_multi_domain) {
    return "multi_expert_workflow"; // Orchestrate multiple experts
  }

  // Default fallback
  return "language_expert"; // mistral:7b-language (general purpose)
}
```

**Example 4: MoE vs Governance - Critical Distinction**

```
❌ WRONG: MoE does governance
User query: "Show me all invoices"
  ↓
MoE Router decides: "User shouldn't see invoice-003" ← WRONG!
  ↓
MoE filters data based on its own logic ← UNGOVERNABLE!

✅ CORRECT: Governance gates BEFORE MoE
User query: "Show me all invoices"
  ↓
Governance filters: Only authorized invoices (invoice-001, invoice-002)
  ↓
MoE Router receives: Already-filtered authorized data
  ↓
MoE selects expert: Finance Expert
  ↓
Finance Expert processes: Only authorized data (no access to invoice-003)
```

**Key Principle:** MoE improves **how** reasoning happens; governance controls **what** can be reasoned about.

**Critical Distinction:**

- ❌ **WRONG:** MoE decides what data to access (MoE as governance)
- ✅ **CORRECT:** Governance gates BEFORE MoE (MoE only sees authorized data)

---

### **STEP 8: Operators as MoE/Agents**

**Problem:** HAI Indexer has graph operators (KHop, PPR, Steiner, etc.). Can they be MoE? Can they be agents?

**Analysis:**

| Question                          | Answer                     | Rationale                                                   |
| --------------------------------- | -------------------------- | ----------------------------------------------------------- |
| **Can operators be MoE?**         | ❌ NO                      | MoE is for LLM routing; operators are graph algorithms      |
| **Can operators be agents?**      | ✅ YES (with modification) | Operators can propose graph expansions; governance approves |
| **Should operators be governed?** | ✅ YES                     | Operators should respect ACL during traversal               |

**Example 1: Operator as Governed Graph Algorithm (Current State)**

```
User query: "Find all tasks related to Alice"
  ↓
1. Governance checks: User can access TASK domain → ✅ allowed
  ↓
2. KHop operator executes:
   - Start node: (person:Person {name: "Alice"})
   - Traverse: (person)-[:ASSIGNED_TO]->(task:Task)
   - ACL check during traversal: Filter out unauthorized tasks
   - Result: 5 authorized tasks (2 blocked by ACL)
  ↓
3. Return: Only authorized tasks

Cypher query (ACL-aware):
MATCH (p:Person {name: "Alice"})-[:ASSIGNED_TO]->(t:Task)
WHERE t.acl_policy_id IN user.allowed_policies  ← ACL filtering
RETURN t
```

**Example 2: Operator as Agent (Proposal Generator)**

```
User query: "Find all tasks related to Alice and expand to related meetings"
  ↓
1. KHop operator proposes expansion:
   {
     "proposal_id": "prop-001",
     "operator": "KHop",
     "action": "EXPAND_GRAPH",
     "from_nodes": ["task-001", "task-002"],
     "expansion_pattern": "(task)-[:DISCUSSED_IN]->(meeting:Meeting)",
     "estimated_nodes": 15,
     "risk_level": "MEDIUM"
   }
  ↓
2. Governance evaluates proposal:
   - Check ACL: Can user access MEETING domain?
   - Check permissions: Can user expand graph?
   - Check risk: Is expansion within limits?
  ↓
3. Governance decision:
   {
     "decision": "APPROVED_WITH_LIMITS",
     "max_nodes": 10,
     "reason": "User has MEETING access, but limit expansion to 10 nodes"
   }
  ↓
4. Operator executes with limits:
   - Expand graph: (task)-[:DISCUSSED_IN]->(meeting)
   - ACL filter: Only authorized meetings
   - Limit: Max 10 nodes
   - Result: 8 authorized meetings (5 blocked by ACL, 2 blocked by limit)
  ↓
5. Return: Authorized meetings within limits
```

**Example 3: Why Operators Are NOT MoE**

```
❌ WRONG: Operator as MoE
User query: "Find related tasks"
  ↓
Operator Router decides: "Use KHop for this query" ← This is NOT MoE!
  ↓
Reason: Operator selection is deterministic (based on query pattern),
        not domain-specific expert routing

✅ CORRECT: MoE is for LLM expert selection
User query: "Summarize related tasks"
  ↓
Governance filters: Authorized tasks
  ↓
MoE Router decides: "Use Task Management Expert (mistral:7b-tasks)" ← This IS MoE!
  ↓
Reason: MoE routes to domain-specific LLM expert,
        not graph algorithm
```

**Comparison Table:**

| Aspect              | Operator (Graph Algorithm)       | MoE (LLM Router)                    | Agent (Proposal Generator)          |
| ------------------- | -------------------------------- | ----------------------------------- | ----------------------------------- |
| **Purpose**         | Graph traversal/expansion        | Route to domain-specific LLM expert | Propose actions for approval        |
| **Input**           | Graph nodes + traversal pattern  | Query + context                     | User intent + context               |
| **Output**          | Graph nodes                      | LLM expert selection                | Action proposals                    |
| **Governance**      | ACL-aware traversal              | Receives pre-filtered data          | Proposals evaluated by governance   |
| **Autonomy**        | Executes within ACL constraints  | No autonomy (just routing)          | No autonomy (just proposes)         |
| **Can be governed** | ✅ YES (ACL during traversal)    | ✅ YES (governance gates before)    | ✅ YES (governance approves/denies) |
| **Is MoE?**         | ❌ NO (graph algorithm, not LLM) | ✅ YES (LLM expert routing)         | ❌ NO (action proposer, not LLM)    |
| **Is Agent?**       | ✅ YES (if modified to propose)  | ❌ NO (just routing, not proposing) | ✅ YES (by definition)              |

**Recommendation:**

1. **Keep operators as governed graph algorithms** for deterministic traversal
2. **Optionally extend operators to propose expansions** (agent-like behavior) for complex queries
3. **Never treat operators as MoE** (they are not LLM routers)

**Implementation Guidance:**

```javascript
// Operator as governed graph algorithm (current)
class KHopOperator {
  execute(startNode, pattern, depth) {
    // Execute traversal with ACL filtering
    return this.traverseWithACL(startNode, pattern, depth);
  }
}

// Operator as agent (future enhancement)
class KHopOperatorAgent {
  proposeExpansion(startNode, pattern, depth) {
    // Propose expansion, don't execute
    return {
      proposal_id: generateId(),
      action: "EXPAND_GRAPH",
      pattern: pattern,
      estimated_cost: this.estimateCost(pattern, depth),
    };
  }

  executeApprovedExpansion(proposal, governanceDecision) {
    // Execute only if approved
    if (governanceDecision.approved) {
      return this.traverseWithACL(proposal.pattern, governanceDecision.limits);
    }
  }
}
```

---

## 7. Complete End-to-End Flow {#end-to-end-flow}

### **Scenario: Alice Queries Finance Invoices**

**User query:** "Summarize January invoices and email the summary to my manager"

**Step-by-step flow:**

**1. Authentication (Keycloak)**

```json
{
  "user": "alice@company.com",
  "roles": ["FINANCE_USER", "EMPLOYEE"],
  "jwt": "eyJhbGc..."
}
```

**2. Intent Detection**

```json
{
  "domain": "FINANCE",
  "intent": "SUMMARIZE",
  "requested_actions": ["SUMMARIZE", "EMAIL"]
}
```

**3. Document Lookup (MongoDB)**

```json
{
  "documents": [
    {
      "doc_id": "doc-001",
      "file_id": "invoice-jan-2024.pdf",
      "domain": "FINANCE_INVOICE",
      "acl_policy_id": "acl-finance-internal"
    }
  ]
}
```

**4. ACL Policy Lookup (MongoDB)**

```json
{
  "acl_id": "acl-finance-internal",
  "allowed_roles": ["FINANCE_USER", "FINANCE_MANAGER"],
  "permissions": {
    "read": true,
    "summarize": true,
    "email": false
  }
}
```

**5. Pre-Retrieval Governance (Neo4j)**

```
Graph traversal with ACL filtering:
- Start nodes: invoice-001, invoice-002, invoice-003
- ACL check: Alice has FINANCE_USER role → ✅ allowed
- Result: 3 authorized invoice nodes
```

**6. Vector Search (Qdrant)**

```json
{
  "query": "January invoices",
  "filter": {
    "tenant_id": "tenant_xyz",
    "principals": { "$in": ["alice@company.com", "FINANCE_USER"] }
  },
  "limit": 10
}
```

**7. Post-Retrieval Execution Gate**

```json
{
  "decision": "READ_ONLY",
  "allowed_actions": ["read", "summarize"],
  "blocked_actions": ["EMAIL"],
  "reason": "Email permission denied by ACL policy"
}
```

**8. Prompt Selection (Spring Cloud Config)**

```yaml
# Loaded from Git: prompts/finance/invoice-summary-v1.yaml
template: |
  Summarize the following invoice documents:
  {context}
  Focus on: totals, anomalies, key vendors
```

**9. SLM Reasoning (Ollama/Mistral)**

```
Governed prompt + authorized chunks → Mistral 7B → Summary generated
```

**10. Response to Alice**

```
✅ Summary:
   January invoices total: $45,230
   Key vendors: Acme Corp ($20,000), Beta Inc ($15,000)
   Anomaly: Invoice #003 is 3x higher than average

❌ Email action blocked: You don't have permission to email finance documents.
   Contact your manager to request access.
```

**11. Audit Trail (MongoDB)**

```json
{
  "user": "alice@company.com",
  "query": "Summarize January invoices and email...",
  "decision": "READ_ONLY",
  "allowed_actions": ["summarize"],
  "blocked_actions": ["EMAIL"],
  "timestamp": "2024-01-15T10:30:00Z"
}
```

---

## 8. Agents & MoE: Where They Fit {#agents-moe}

### **8.1 Agents: Participants, Not Authorities**

**Philosophical Stance:**

> **Agents never own execution. Agents propose; governance disposes.**

**Traditional Agentic AI (WRONG):**

```
Agent decides → Agent executes → Done
```

**Problems:**

- No governance oversight
- Ungovernable execution paths
- Compliance violations
- No audit trail

**Governed Agentic AI (CORRECT):**

```
Agent proposes → Governance evaluates → Governance approves/denies → Execution happens
```

**Benefits:**

- ✅ All actions governed
- ✅ Complete audit trail
- ✅ Compliance by design
- ✅ Agents can't bypass policy

**Example:**

```
Agent: "I should email this invoice summary to the manager"
  ↓
Governance: "Check ACL policy for EMAIL permission"
  ↓
ACL Policy: email=false
  ↓
Governance: "DENY - Email action blocked"
  ↓
Agent: "Execution denied, inform user"
```

**Key principle:** Agents are constrained by governance, not autonomous.

---

### **8.2 MoE: Inside SLM, After Governance**

**Philosophical Stance:**

> **MoE is not governance. MoE lives INSIDE the SLM, AFTER governance gates.**

**Where MoE Fits:**

```
┌─────────────────────────────────────────────────────────────────┐
│  GOVERNANCE LAYER (BEFORE MoE)                                  │
│  ├─ ACL filtering                                               │
│  ├─ Execution gating                                            │
│  └─ Prompt selection                                            │
├─────────────────────────────────────────────────────────────────┤
│  SLM LAYER (MoE LIVES HERE)                                     │
│  ├─ MoE Router                                                  │
│  ├─ Finance Expert (mistral:7b-finance)                         │
│  ├─ Language Expert (mistral:7b-language)                       │
│  ├─ Explanation Expert (mistral:7b-explain)                     │
│  └─ Code Expert (mistral:7b-code)                               │
└─────────────────────────────────────────────────────────────────┘
```

**Flow:**

```
1. Governance filters data (ACL-aware)
2. Governance selects prompt (from Git)
3. Governed prompt + authorized data → MoE Router
4. MoE Router selects expert based on:
   - Domain (FINANCE → Finance Expert)
   - Intent (SUMMARIZE → Explanation Expert)
   - Complexity (COMPLEX → Code Expert)
5. Expert generates response
6. Response returned (no governance bypass)
```

**Key principle:** MoE improves **how** reasoning happens; governance controls **what** can be reasoned about.

---

### **8.3 Operators: Governed Graph Algorithms**

**Can operators be MoE?** ❌ NO

- MoE is for LLM routing (selecting which expert model to use)
- Operators are graph algorithms (KHop, PPR, Steiner)
- Different concerns

**Can operators be agents?** ✅ YES (with modification)

- Operators can propose graph expansions
- Governance approves or denies expansion
- Example: "KHop proposes expanding to 3-hop neighbors; governance checks ACL; only authorized neighbors included"

**Should operators be governed?** ✅ YES

- Operators should respect ACL during traversal
- Unauthorized nodes should be structurally pruned
- No post-hoc filtering

**Recommendation:** Keep operators as governed graph algorithms, not MoE or autonomous agents.

---

## 9. Deployment Architecture & Integration Patterns {#deployment}

This section outlines the deployment architecture and integration patterns for Intelligence Governance, focusing on architectural decisions rather than delivery timelines.

---

### **9.1 Deployment Topology**

**Three-Tier Architecture:**

```
┌─────────────────────────────────────────────────────────────────┐
│  CONTROL PLANE (Governance)                                     │
│  ├─ MongoDB (Document Registry, ACL Policies, Audit Logs)       │
│  ├─ Spring Cloud Config (Prompt Registry from Git)              │
│  └─ Keycloak (RBAC, Identity Management)                        │
├─────────────────────────────────────────────────────────────────┤
│  INTELLIGENCE PLANE (Reasoning)                                 │
│  ├─ HAI Indexer (Query Pipeline, Governance Integration)        │
│  ├─ Ollama (LLM: Mistral 7B + Domain Experts)                   │
│  ├─ MoE Router (Expert Selection)                               │
│  └─ Agent Orchestrator (Proposal Generator)                     │
├─────────────────────────────────────────────────────────────────┤
│  DATA PLANE (Storage)                                           │
│  ├─ Neo4j (Knowledge Graph + ACL Nodes)                         │
│  ├─ Qdrant (Vector Store with ACL Metadata)                     │
│  └─ Redis (Caching Layer)                                       │
└─────────────────────────────────────────────────────────────────┘
```

**Key Architectural Decisions:**

1. **Governance plane is orthogonal** - Can be deployed independently
2. **MongoDB as governance store** - Policy-shaped data, no migrations
3. **Git as prompt source of truth** - Version control, audit trail
4. **Neo4j for structural ACL** - Graph topology enforces access control
5. **MoE inside SLM layer** - After governance gates, not before

---

### **9.2 Integration Patterns**

**Pattern 1: Document Indexing Flow**

```
Google Drive → HAI Indexer → Governance Control Plane → Data Plane
                    ↓
              1. Extract metadata
              2. Classify document
              3. Determine ACL policy
              4. Register in MongoDB
              5. Index in Qdrant (with ACL metadata)
              6. Create graph nodes in Neo4j (with ACL links)
```

**Pattern 2: Query Execution Flow**

```
User Query → Authentication → Governance Gates → Intelligence Plane → Response
                ↓                    ↓                    ↓
           Keycloak          MongoDB + Neo4j      MoE + Agents
           (RBAC)            (ACL filtering)      (Reasoning)
```

**Pattern 3: Agent Proposal Flow**

```
Agent Proposes → Governance Evaluates → Execution (if approved)
      ↓                   ↓                      ↓
  Action JSON      ACL + Risk Check      Controlled Execution
```

**Pattern 4: MoE Routing Flow**

```
Governed Data → MoE Router → Expert Selection → Response
      ↓              ↓              ↓
  Pre-filtered   Analyze      Route to
  by ACL         domain       domain expert
```

---

### **9.3 Component Integration Matrix**

| Component               | Integrates With               | Integration Type | Purpose                         |
| ----------------------- | ----------------------------- | ---------------- | ------------------------------- |
| **MongoDB**             | HAI Indexer, Neo4j, Qdrant    | REST API         | Governance metadata store       |
| **Spring Cloud Config** | HAI Indexer                   | HTTP + Refresh   | Prompt registry                 |
| **Keycloak**            | HAI Indexer                   | OAuth2/OIDC      | Identity & RBAC                 |
| **Neo4j**               | HAI Indexer, MongoDB          | Cypher + Bolt    | Graph + ACL enforcement         |
| **Qdrant**              | HAI Indexer, MongoDB          | gRPC             | Vector search with ACL metadata |
| **Ollama**              | HAI Indexer, MoE Router       | REST API         | LLM inference                   |
| **MoE Router**          | Ollama, HAI Indexer           | Internal API     | Expert model selection          |
| **Agent Orchestrator**  | HAI Indexer, Governance Layer | Internal API     | Proposal generation             |

---

### **9.4 Scalability Considerations**

**Horizontal Scaling:**

- **MongoDB**: Sharded by `tenant_id` for multi-tenancy
- **Neo4j**: Causal clustering for read replicas
- **Qdrant**: Distributed collections for large-scale vector search
- **HAI Indexer**: Stateless, can scale horizontally
- **Ollama**: Multiple instances for parallel LLM inference

**Caching Strategy:**

- **Redis**: Cache ACL policy lookups (TTL: 5 minutes)
- **Spring Cloud Config**: Cache prompts locally (refresh on Git commit)
- **Neo4j**: Cache frequently accessed ACL nodes

**Performance Targets:**

- ACL policy lookup: < 10ms (cached), < 50ms (uncached)
- Graph ACL traversal: < 100ms for 1000 nodes
- Vector search with ACL filtering: < 200ms for 10K documents
- End-to-end query latency: < 2 seconds (including LLM inference)

---

### **9.5 Security & Compliance**

**Data Isolation:**

- **Tenant isolation**: All collections partitioned by `tenant_id`
- **User isolation**: ACL policies enforce user-level access control
- **Network isolation**: Control plane in separate VPC/subnet

**Audit Trail:**

- **MongoDB audit_log collection**: All governance decisions logged
- **Git commit history**: All prompt changes tracked
- **Keycloak audit logs**: All authentication events logged
- **Neo4j query logs**: All graph traversals logged

**Compliance:**

- **GDPR**: Redaction profiles for PII
- **SOX**: Audit trail for financial data access
- **HIPAA**: Encryption at rest and in transit
- **ISO 27001**: Access control and audit logging

---

### **9.6 Monitoring & Observability**

**Key Metrics:**

- **Governance metrics**: ACL denials, redaction events, policy violations
- **Performance metrics**: Query latency, ACL lookup time, graph traversal time
- **Agent metrics**: Proposal approval rate, denial reasons
- **MoE metrics**: Expert selection distribution, routing accuracy

**Alerting:**

- **High ACL denial rate**: Possible misconfigured policy
- **Slow ACL lookups**: MongoDB performance issue
- **Agent proposal failures**: Governance policy too restrictive
- **MoE routing errors**: Expert model unavailable

**Dashboards:**

- **Governance Dashboard**: ACL denials, redaction events, audit trail
- **Performance Dashboard**: Query latency, component health
- **Agent Dashboard**: Proposal approval/denial rates
- **MoE Dashboard**: Expert selection distribution

---

## 10. Summary & Next Steps

### **What We've Built (Conceptually)**

1. **Intelligence Governance as a first-class control plane** - orthogonal to identity, data, and inference
2. **Three-tier defense-in-depth** - Pre-retrieval (graph ACL) + Post-retrieval (execution gate) + SLM (MoE)
3. **Governance before intelligence** - Unauthorized data never enters reasoning context
4. **Structure over filters** - ACL is graph topology, not prompt-based
5. **Agents without authority** - Agents propose, governance approves
6. **Graded outcomes** - ALLOW / REDACT / READ_ONLY / REQUIRE_HUMAN / DENY
7. **MoE inside SLM** - MoE improves reasoning quality; governance controls what can be reasoned about

### **Technology Decisions**

- **MongoDB** for governance control plane (policy-shaped data, high change velocity)
- **Spring Cloud Config + Git** for prompt registry (version control, hot refresh, audit trail)
- **Keycloak** for RBAC (identity-time authorization)
- **MongoDB** for ACL (data-time authorization)
- **Neo4j** for graph-native ACL enforcement
- **Qdrant** for filter-first vector search

### **Implementation Priorities**

**Phase 1: Foundation (Control Plane)**

1. Deploy MongoDB for governance metadata
2. Create document registry and ACL policy collections
3. Integrate with existing HAI Indexer indexing pipeline
4. Establish audit logging

**Phase 2: Structural Governance (Pre-Retrieval)**

1. Create ACL nodes in Neo4j knowledge graph
2. Modify graph operators (KHop, PPR, Steiner) for ACL-aware traversal
3. Add ACL metadata to Qdrant vector search
4. Test: Unauthorized data never enters retrieval context

**Phase 3: Execution Governance (Post-Retrieval)**

1. Implement ExecutionGate service for graded permissions
2. Set up Spring Cloud Config + Git for prompt registry
3. Migrate hardcoded prompts to version-controlled Git repository
4. Test: Graded outcomes (ALLOW/REDACT/READ_ONLY/DENY)

**Phase 4: Intelligence Enhancement (Agents & MoE)**

1. Implement agent proposal system with governance approval
2. Deploy MoE Router for domain-specific expert selection
3. Integrate domain expert models (finance, legal, code, etc.)
4. Optionally extend operators to propose graph expansions

---

### **Architectural Review Checklist**

Before implementation, validate these architectural decisions:

- [ ] **MongoDB vs PostgreSQL**: Confirmed for policy-shaped data with high change velocity
- [ ] **Spring Cloud Config vs Database**: Confirmed for prompt registry with Git version control
- [ ] **RBAC (Keycloak) vs ACL (MongoDB)**: Separation of identity-time vs data-time authorization
- [ ] **Graph-native ACL**: ACL as first-class Neo4j nodes, not post-hoc filtering
- [ ] **Graded outcomes**: Not binary ALLOW/DENY, but ALLOW/REDACT/READ_ONLY/REQUIRE_HUMAN/DENY
- [ ] **Agents without authority**: Agents propose, governance approves
- [ ] **MoE after governance**: MoE receives pre-filtered data, doesn't do governance

---

**This document serves as the execution blueprint for Intelligence Governance implementation.**

**It is designed for architect review, focusing on conceptual clarity, architectural decisions, and detailed examples rather than delivery timelines.**

