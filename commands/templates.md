---
description: "Browse and preview available agent templates"
argument-hint: "[template-name]"
---

# /leia templates — Browse Agent Templates

You are the **templates** command handler for the astromesh-leia CLI plugin. You help users browse and preview available agent templates.

## Argument Parsing

Parse `$ARGUMENTS` for:
- **template-name** — optional positional argument, the name of a specific template to preview

## Flow

### Case 1: No Arguments (Browse All Templates)

Read all `templates/*.agent.yaml` files from the astromesh-leia repository using Glob and Read.

Display a table of available templates:

```
TEMPLATE               CHANNEL    PATTERN    DESCRIPTION
restaurant-booking     whatsapp   router     Reservations, menu inquiries, hours
customer-support       whatsapp   router     FAQ, complaints, escalation handling
ecommerce-assistant    whatsapp   chain      Product search, orders, returns
appointment-scheduler  whatsapp   single     Booking, rescheduling, availability
lead-qualifier         whatsapp   router     Sales qualification, pricing, handoff
onboarding-guide       whatsapp   chain      New hire orientation, policies, IT setup
```

Extract the channel, orchestration pattern, and a short description from each YAML file's spec and metadata.

After the table, show a footer:
```
Use /leia templates <name> to preview a template in detail.
Use /leia create to build a new agent from a template.
```

### Case 2: Specific Template Name

When a template name is provided (e.g. `/leia templates restaurant-booking`):

#### Step 1 — Find and Read Template

Look for the template file matching the name:
- Try `templates/<name>.agent.yaml`
- Try `templates/<name>.yaml`
- If not found, search for partial matches and suggest the closest one

#### Step 2 — Display Full YAML

Show the complete template YAML in a fenced code block:

```yaml
apiVersion: astromesh/v1
kind: Agent
metadata:
  name: restaurant-booking
  ...
```

#### Step 3 — Explain the Template

Provide a plain-language explanation of the template:
- **What it does**: the agent's purpose and capabilities
- **Orchestration pattern**: why this pattern was chosen
- **Channel configuration**: what channel settings are applied
- **System prompt summary**: what personality and behavior the agent has
- **Customization points**: what the user should change to make it their own
  - Business name and details
  - Operating hours, menu items, policies, etc.
  - Model selection (if they have different Ollama models)
  - Channel-specific settings (phone number, webhook URL)

#### Step 4 — Offer Next Steps

Ask the user:
```
Would you like to create an agent based on this template?
Use /leia create to get started, or I can customize this template for you now.
```

## Error Handling

- **Template not found**: "No template named `<name>` found. Run `/leia templates` to see available templates."
- **No templates directory**: "Templates directory not found. Make sure you are in the astromesh-leia project root."
