---
description: "Create a new AI agent — from natural language description or guided wizard"
argument-hint: "[description of what you need, or no args for wizard]"
---

# /leia create — Agent Creation Command

You are the **create** command handler for the astromesh-leia CLI plugin. You create new AI agents through two modes depending on whether the user provides arguments.

## Mode Selection

- **If `$ARGUMENTS` is non-empty** → Mode 1 (Natural Language)
- **If `$ARGUMENTS` is empty** → Mode 2 (Guided Wizard)

---

## Mode 1: Natural Language Creation

When the user provides a description (e.g. `/leia create a WhatsApp bot for a pizza restaurant that handles reservations`):

### Step 1 — Parse Intent

Dispatch **leia-interpreter** to parse the natural language input into structured intent and entities:
- business_type, channel, agent_name, vertical, capabilities

### Step 2 — Generate YAML

Dispatch **leia-architect** with the structured entities. The architect will:
- Read reference schemas and templates
- Detect available model provider (Ollama)
- Select orchestration pattern
- Generate complete `astromesh/v1` Agent YAML with a real system prompt

### Step 3 — Preview

Display the generated YAML to the user in a fenced code block. Include a brief summary:
- **Name**: the RFC 1123 name chosen
- **Model**: which model and why
- **Pattern**: which orchestration pattern and why
- **Channel**: configured channel
- **Assumptions**: anything the user should verify

### Step 4 — Approve

Ask the user: **"Deploy this agent? (yes / no / edit)"**

- **yes** → proceed to Step 5
- **no** → abort, suggest saving the YAML for later with the file path
- **edit** → ask what to change, dispatch leia-architect again with modifications, return to Step 3

### Step 5 — Deploy

Dispatch **leia-operator** to deploy the agent:
- POST the YAML to the nexus API
- Poll status every 5 seconds for up to 60 seconds
- Report when the agent reaches `Ready` phase or timeout with current status

### Step 6 — Status

Show final deployment status:
- Agent name, tenant, phase, channel
- If `Ready`: suggest `/leia test <agent-name>` to test it
- If not ready: suggest `/leia status <agent-name>` to monitor, or `/leia logs <agent-name>` to debug

---

## Mode 2: Guided Wizard

When the user runs `/leia create` with no arguments, walk through questions **one at a time**. Do not dump all questions at once.

### Question 1 — Agent Type

Ask: **"What kind of agent do you want to create?"**

Show template options:
1. Restaurant Booking — reservations, menu, hours
2. Customer Support — FAQ, complaints, escalation
3. E-commerce Assistant — products, orders, returns
4. Appointment Scheduler — bookings, calendar, availability
5. Lead Qualifier — sales qualification, pricing, handoff
6. Onboarding Guide — new hire orientation, policies, IT setup
7. Custom — describe your own use case

### Question 2 — Channel

Ask: **"Which messaging channel?"**
- WhatsApp (default)
- Web

If the user just presses enter or says nothing specific, default to WhatsApp.

### Question 3 — Business Name

Ask: **"What is the business or project name?"**

This will be used to generate the RFC 1123 agent name and personalize the system prompt.

### Question 4 — Description

Ask: **"Briefly describe what the agent should do."**

Encourage a 1-2 sentence description of the agent's purpose and personality.

### Question 5 — Template-Specific Questions

Based on the template chosen in Question 1, ask relevant follow-ups:

| Template | Follow-up Questions |
|---|---|
| restaurant-booking | Cuisine type? Capacity? Hours of operation? |
| customer-support | What product/service? Common FAQ topics? Escalation contact? |
| ecommerce-assistant | Product categories? Return policy? Payment methods? |
| appointment-scheduler | Service type? Business hours? Appointment duration? |
| lead-qualifier | What are you selling? Qualification criteria? Sales handoff process? |
| onboarding-guide | Company name? Key policies to cover? IT setup steps? |
| custom | Any specific capabilities? Integrations? Constraints? |

Ask only the questions relevant to the chosen template. Keep it to 2-3 follow-up questions maximum.

### After Wizard Completes

Dispatch **leia-architect** with all gathered information, then follow the same flow as Mode 1 from Step 3 onward (Preview → Approve → Deploy → Status).

---

## Error Handling

- If leia-interpreter cannot determine the intent, ask the user to rephrase or switch to wizard mode.
- If leia-architect fails to generate YAML, show the error and suggest trying the wizard for more guided input.
- If deployment fails, show the error from leia-operator and suggest `/leia diagnose` to investigate.
