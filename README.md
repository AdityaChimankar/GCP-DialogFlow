# IBM Watson → GCP Conversational Agents Migration Guide

> Migrate IBM Watson Assistant flows to Google Cloud Conversational Agents (Dialogflow CX) using **Generative Playbooks** powered by Gemini.

---

## Table of Contents

- [Overview](#overview)
- [Architecture Comparison](#architecture-comparison)
- [Concept Mapping](#concept-mapping)
- [Migration Approach](#migration-approach)
  - [Phase 1: Audit & Categorize Watson Flows](#phase-1-audit--categorize-watson-flows)
  - [Phase 2: Build the GCP Agent Structure](#phase-2-build-the-gcp-agent-structure)
  - [Phase 3: Migrate Intents & Entities](#phase-3-migrate-intents--entities)
- [Playbook Design Guide](#playbook-design-guide)
  - [Playbook Types](#playbook-types)
  - [Key Parameters](#key-parameters)
  - [When NOT to Use Playbooks](#when-not-to-use-playbooks)
- [Intent & Entity Migration Script](#intent--entity-migration-script)
- [Webhook → OpenAPI Tool Migration](#webhook--openapi-tool-migration)
- [Agent Structure Template](#agent-structure-template)
- [Pre-Migration Checklist](#pre-migration-checklist)
- [Known Limitations](#known-limitations)
- [References](#references)

---

## Overview

This guide covers migrating IBM Watson Assistant workspaces to **Google Cloud Conversational Agents** (formerly Dialogflow CX), specifically using the **Generative Playbook** approach for AI-native agent creation.

### Why Migrate to GCP Conversational Agents?

| Feature | IBM Watson Assistant | GCP Conversational Agents |
|---|---|---|
| Gen AI model | Various LLM providers (via RAG) | Gemini 2.5 Flash/Pro (native) |
| Flow control | Dialog nodes + context | Flows, Pages, Playbooks |
| State management | Context variables (expire by turns) | Session parameters (persistent) |
| Search/RAG | Search Skill | Data Store Tool (AlloyDB, GCS, etc.) |
| Voice support | Watson Speech | Chirp 3 HD (40+ languages) |
| Pricing | Per MAU / API call | Per request (edition-based) |

---

## Architecture Comparison

```
IBM Watson Assistant                GCP Conversational Agents
─────────────────────               ─────────────────────────────
Workspace / Skill           ──►     Agent
  └── Dialog Flow           ──►       └── Flow
        └── Dialog Node     ──►             └── Page
              ├── Condition  ──►               ├── Route (Intent/Condition)
              ├── Response   ──►               ├── Fulfillment
              └── Jump-to    ──►               └── Page Transition

Context Variables           ──►     Session Parameters
Slot Filling (in node)      ──►     Form (on Page)
Webhook (action)            ──►     Webhook / OpenAPI Tool
Search Skill                ──►     Data Store Tool
Intent                      ──►     Intent (same concept)
Entity                      ──►     Entity Type
Assistant (multi-skill)     ──►     Default Start Playbook
```

---

## Concept Mapping

| IBM Watson | GCP Conversational Agents | Notes |
|---|---|---|
| Workspace | Agent | Top-level container |
| Skill | Flow | Grouping of related dialog |
| Dialog Node | Page | State in conversation |
| Dialog Condition | Route Condition | Evaluated on each turn |
| Context Variable | Session Parameter | Persists across turns |
| Slot / Action Variable | Form Parameter | Collected on a Page |
| Intent | Intent | Training phrases map 1:1 |
| Entity | Entity Type | System + custom entities |
| Webhook | Webhook / Tool | HTTP fulfillment call |
| Search Skill | Data Store Tool | RAG / knowledge base |
| Jump to node | Page Transition | Deterministic routing |
| Digression | Route with scope | Handler at flow level |
| Anything else | Default Fallback Route | Catch-all handler |

---

## Migration Approach

### Phase 1: Audit & Categorize Watson Flows

Export your Watson workspace JSON and categorize each dialog node:

| Watson Dialog Node Type | Recommended GCP Approach |
|---|---|
| Simple FAQ / Q&A | **Generative Playbook** |
| Slot-filling (collect structured data) | **Flow + Page with Form** |
| Complex branching + backend calls | **Flow + Webhook** |
| Search / knowledge retrieval | **Data Store Tool** |
| IVR / DTMF (voice keypad input) | **Flow only** (Playbooks don't support DTMF) |
| Compliance / exact-script responses | **Flow** (deterministic) |

---

### Phase 2: Build the GCP Agent Structure

Recommended agent structure after migration:

```
Agent
 ├── Default Generative Playbook      ← Entry point; routes to sub-playbooks
 │     └── Tools: [Routine Playbooks, Flows]
 │
 ├── Routine Playbook: "Order Status"  ← Replaces Watson dialog branch
 ├── Routine Playbook: "Returns"       ← Replaces Watson dialog branch
 ├── Routine Playbook: "FAQ"           ← Replaces Watson Search Skill nodes
 │
 ├── Task Playbook: "Collect Order ID" ← Replaces Watson slot-filling node
 ├── Task Playbook: "Verify Customer"  ← Reusable sub-task
 │
 ├── Flow: "Escalation to Human"       ← Deterministic; must be exact
 ├── Flow: "IVR / DTMF Menu"           ← Voice-only flows
 │
 └── Data Store Tool                   ← Replaces Watson Search Skill
       └── Source: GCS / AlloyDB / Cloud SQL
```

> **Note:** The Default Playbook is auto-created when you create a generative agent.  
> It does **not** receive a summary of preceding turns and cannot define input parameters.

---

### Phase 3: Migrate Intents & Entities

**Watson Export Format:**
```json
{
  "intents": [
    {
      "intent": "order_status",
      "examples": [
        { "text": "where is my order" },
        { "text": "track my package" }
      ]
    }
  ]
}
```

**Dialogflow CX Format:**
```json
{
  "displayName": "order_status",
  "trainingPhrases": [
    { "parts": [{ "text": "where is my order" }], "repeatCount": 1 },
    { "parts": [{ "text": "track my package" }], "repeatCount": 1 }
  ]
}
```

Use the migration script below to automate the conversion.

---

## Playbook Design Guide

### Playbook Types

| Type | Purpose | Supports Input/Output Params | Can Call |
|---|---|---|---|
| **Default Playbook** | Entry point only | ❌ No | Routines, Tasks, Flows |
| **Routine Playbook** | Sequential, independent conversation stages | ❌ No | Task Playbooks, Flows |
| **Task Playbook** | Reusable sub-tasks, compositional stages | ✅ Yes | Other Task Playbooks |

> **Rule:** Any routine or task playbook can call a task playbook,  
> but a task playbook **cannot** call a routine playbook.

---

### Key Parameters

When creating a playbook, configure the following:

#### 1. Goal / Instructions (most critical)
Write like a system prompt. Be specific about scope.

```
# Example: Order Status Routine Playbook

Goal: Help the user check the status of their order.

Instructions:
- Ask for order ID if not provided.
- Call the `get_order_status` tool with the order ID.
- Present the status clearly.
- If order not found, apologize and offer to connect to support.
- Do NOT handle returns or refunds — route to the Returns playbook.
```

#### 2. Input / Output Parameters (Task Playbooks)
Map your Watson context variables here:

| Watson Context Variable | CX Task Playbook Parameter | Type |
|---|---|---|
| `$order_id` | `order_id` (input) | String |
| `$customer_verified` | `is_verified` (input) | Boolean |
| `$order_status` | `status_result` (output) | String |

#### 3. Attached Tools
Replace Watson webhooks with Tools:

| Watson Webhook Action | CX Tool Type |
|---|---|
| REST API call | OpenAPI Tool |
| Knowledge base search | Data Store Tool |
| Inline logic / transform | Function Tool |

#### 4. LLM Model Selection

| Use Case | Recommended Model |
|---|---|
| High-volume FAQ / routing | `gemini-2.5-flash` (default) |
| Complex reasoning / analysis | `gemini-2.5-pro` |
| Cost-sensitive, simple responses | `gemini-2.5-flash-lite` |

#### 5. Fallback Behavior
Define explicitly:
- Route to another playbook
- Transition to a deterministic Flow
- Trigger `sys.no-match` event → human escalation

#### 6. Language / Locale
Verify your language is tested for Playbook quality with Gemini models before migrating. Check the [Playbook language reference](https://cloud.google.com/dialogflow/cx/docs/reference/language).

---

### When NOT to Use Playbooks

Use **deterministic Flows** instead when:

- ☎️ IVR / DTMF input from telephone systems
- 📋 Regulatory / compliance scripts (exact wording required)
- 🔒 High-stakes form collection with strict field validation
- 🔁 Loops with complex condition branching
- 📞 Call companion SMS from Default Welcome Intent

---

## Intent & Entity Migration Script

Save as `migrate_watson_to_cx.py` and run locally:

```python
import json
import sys

def migrate_intents(watson_file: str, output_file: str):
    """Convert Watson Assistant intents to Dialogflow CX format."""
    with open(watson_file, 'r') as f:
        watson_data = json.load(f)

    cx_intents = []

    for intent in watson_data.get('intents', []):
        cx_intent = {
            "displayName": intent['intent'],
            "trainingPhrases": [
                {
                    "parts": [{"text": example['text']}],
                    "repeatCount": 1
                }
                for example in intent.get('examples', [])
            ]
        }
        cx_intents.append(cx_intent)

    with open(output_file, 'w') as f:
        json.dump({"intents": cx_intents}, f, indent=2)

    print(f"✅ Migrated {len(cx_intents)} intents → {output_file}")


def migrate_entities(watson_file: str, output_file: str):
    """Convert Watson Assistant entities to Dialogflow CX entity types."""
    with open(watson_file, 'r') as f:
        watson_data = json.load(f)

    cx_entities = []

    for entity in watson_data.get('entities', []):
        cx_entity = {
            "displayName": entity['entity'],
            "kind": "KIND_MAP",
            "entities": [
                {
                    "value": value['value'],
                    "synonyms": value.get('synonyms', [value['value']])
                }
                for value in entity.get('values', [])
            ]
        }
        cx_entities.append(cx_entity)

    with open(output_file, 'w') as f:
        json.dump({"entityTypes": cx_entities}, f, indent=2)

    print(f"✅ Migrated {len(cx_entities)} entity types → {output_file}")


if __name__ == "__main__":
    # Usage: python migrate_watson_to_cx.py workspace.json
    watson_file = sys.argv[1] if len(sys.argv) > 1 else "workspace.json"
    migrate_intents(watson_file, "cx_intents.json")
    migrate_entities(watson_file, "cx_entities.json")
```

**Run:**
```bash
# Export workspace from Watson Console first, then:
python migrate_watson_to_cx.py workspace.json
```

**Import to CX:**
```bash
# Using gcloud CLI
gcloud dialogflow agent import \
  --location=us-central1 \
  --project=YOUR_PROJECT_ID \
  --agent-file=cx_intents.json
```

---

## Webhook → OpenAPI Tool Migration

Watson webhooks map to **OpenAPI Tools** in CX Playbooks.

**Watson Webhook config:**
```json
{
  "url": "https://your-api.com/order-status",
  "headers": { "Authorization": "Bearer TOKEN" }
}
```

**CX OpenAPI Tool definition (YAML):**
```yaml
openapi: "3.0.0"
info:
  title: Order Status API
  version: "1.0"
servers:
  - url: https://your-api.com
paths:
  /order-status:
    get:
      operationId: getOrderStatus
      summary: Get the status of an order
      parameters:
        - name: order_id
          in: query
          required: true
          schema:
            type: string
      responses:
        "200":
          description: Order status response
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                  estimated_delivery:
                    type: string
```

Upload this YAML in the **Conversational Agents Console → Tools → Create Tool → OpenAPI**.

---

## Pre-Migration Checklist

### Discovery
- [ ] Export Watson workspace JSON (intents + entities + dialog)
- [ ] List all Watson webhooks and their API contracts
- [ ] Identify all Watson context variables used
- [ ] Document slot-filling nodes and required fields
- [ ] Note any Watson Search Skill knowledge base sources

### GCP Setup
- [ ] Create GCP project and enable Dialogflow CX API
- [ ] Create Conversational Agents agent (generative type)
- [ ] Set up Service Account with `Dialogflow API Admin` role
- [ ] Provision data store sources (GCS bucket, AlloyDB, etc.) if using Search Skill replacement

### Migration
- [ ] Run intent/entity migration script
- [ ] Import intents and entities into CX agent
- [ ] Recreate Watson dialog flows as CX Flows or Playbooks (per categorization)
- [ ] Recreate webhooks as OpenAPI Tools
- [ ] Map all Watson context variables to Session Parameters
- [ ] Attach tools to appropriate playbooks

### Validation
- [ ] Test each migrated flow/playbook in the Conversational Agents simulator
- [ ] Validate parameter passing between playbooks and flows
- [ ] Test fallback and escalation paths
- [ ] Verify language/locale support for your region
- [ ] Load test with expected concurrent session volume

### Deployment
- [ ] Create agent Version from Draft
- [ ] Set up Environments (staging → production)
- [ ] Configure channel integrations (web, telephony, etc.)
- [ ] Set up monitoring (Cloud Logging + Cloud Monitoring)

---

## Known Limitations

| Limitation | Detail |
|---|---|
| DTMF input | Playbooks do not support DTMF from telephone systems — use Flows |
| Default Playbook | Cannot define or receive input parameters |
| Task → Routine | Task playbooks cannot call routine playbooks |
| Playbook SLA | The Playbook feature is excluded from the Dialogflow CX SLA |
| Call companion SMS | Not supported from Default Welcome Intent when using Playbooks |
| Context summary | Default Playbook does not receive a summary of preceding turns |

---

## References

- [GCP Conversational Agents Documentation](https://cloud.google.com/dialogflow/cx/docs)
- [Playbooks Concept Guide](https://cloud.google.com/dialogflow/cx/docs/concept/playbook)
- [Dialogflow CX Release Notes](https://cloud.google.com/dialogflow/docs/release-notes)
- [Migrating from Dialogflow ES to CX](https://cloud.google.com/dialogflow/cx/docs/how/migrate)
- [OpenAPI Tool Reference](https://cloud.google.com/dialogflow/cx/docs/concept/tools)
- [Data Store Tools](https://cloud.google.com/dialogflow/cx/docs/concept/data-store-tool)
- [Supported Languages for Playbooks](https://cloud.google.com/dialogflow/cx/docs/reference/language)

---

## Contributing

1. Fork the repository
2. Create a feature branch: `git checkout -b feature/add-flow-template`
3. Commit changes: `git commit -m "Add Flow template for slot-filling"`
4. Push: `git push origin feature/add-flow-template`
5. Open a Pull Request

---

*Last updated: May 2026 | GCP Conversational Agents with Gemini 2.5*
