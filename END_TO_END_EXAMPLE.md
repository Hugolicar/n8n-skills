# End-to-End Workflow Example: Kommo → Z-API

Complete workflow demonstrating **Kommo CRM → Z-API WhatsApp** integration with action buttons.

## Scenario

When a new lead is created in Kommo, automatically send a WhatsApp welcome message with action buttons ("Schedule Appointment", "Talk to Human", "See Prices").

---

## Workflow Architecture

```
Kommo Webhook (leads.add)
    ↓
Get Lead Details (Kommo node)
    ↓
Format Phone (Code node)
    ↓
Send WhatsApp with Buttons (Z-API node)
    ↓
Add Note to Lead (Kommo node)
```

---

## Node 1: Trigger — Kommo Webhook

```json
{
  "id": "trigger-1",
  "name": "Kommo Novo Lead",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [100, 300],
  "webhookId": "kommo-lead-created",
  "parameters": {
    "httpMethod": "POST",
    "path": "kommo-lead-created",
    "responseMode": "responseNode",
    "options": {}
  }
}
```

**Kommo webhook setup**: Register `leads.add` webhook in Kommo with URL `https://your-n8n.com/webhook/kommo-lead-created`.

**Critical**: Kommo expects HTTP 200 response within 2 seconds. Use "Respond to Webhook" node or `responseMode: responseNode`.

---

## Node 2: Get Lead Details

```json
{
  "id": "kommo-get-1",
  "name": "Kommo Buscar Lead",
  "type": "n8n-nodes-kommo.kommo",
  "typeVersion": 1,
  "position": [300, 300],
  "parameters": {
    "resource": "leads",
    "operation": "get",
    "leadId": "={{ $json.body.leads.add[0].id }}"
  },
  "credentials": {
    "kommoOAuth2Api": {
      "id": "YOUR_KOMMO_CRED_ID",
      "name": "Kommo account"
    }
  }
}
```

**Data mapping from webhook**:
- Lead ID: `$json.body.leads.add[0].id`
- Contact phone: extracted from nested `contacts` array in response

---

## Node 3: Format Phone Number

```json
{
  "id": "code-format-1",
  "name": "Formatar Telefone",
  "type": "n8n-nodes-base.code",
  "typeVersion": 2,
  "position": [500, 300],
  "parameters": {
    "jsCode": "const lead = $input.first().json;\nconst contact = lead.contacts?.[0] || {};\nconst phoneField = contact.custom_fields_values?.find(\n  f => f.field_code === 'PHONE'\n);\nconst rawPhone = phoneField?.values?.[0]?.value || '';\nconst phone = rawPhone.replace(/\\D/g, '');\nreturn {\n  phone: phone.length >= 10 ? '55' + phone : phone,\n  leadId: lead.id,\n  contactName: contact.name || 'Cliente'\n};"
  }
}
```

**Rules**:
- Remove all non-digits: `.replace(/\D/g, '')`
- Prepend `55` for Brazilian numbers if needed
- Handle missing phone gracefully

---

## Node 4: Send WhatsApp with Buttons (Z-API)

```json
{
  "id": "zapi-buttons-1",
  "name": "Z-API Botões de Ação",
  "type": "n8n-nodes-zapi.zApi",
  "typeVersion": 1,
  "position": [700, 300],
  "parameters": {
    "resource": "messages",
    "operation": "sendMessage",
    "messageType": "buttonActions",
    "phone": "={{ $json.phone }}",
    "message": "Olá {{ $json.contactName }}! Bem-vindo. Como podemos ajudar?",
    "buttons": [
      {
        "id": "1",
        "label": "Agendar Consulta",
        "action": "quickReply"
      },
      {
        "id": "2",
        "label": "Falar com Humano",
        "action": "quickReply"
      },
      {
        "id": "3",
        "label": "Ver Preços",
        "action": "quickReply"
      }
    ]
  },
  "credentials": {
    "zApi": {
      "id": "YOUR_ZAPI_CRED_ID",
      "name": "Z-API Account"
    }
  }
}
```

**Button types supported**:
- `quickReply` — Triggers webhook with `buttonReply` payload
- `url` — Opens external URL
- `call` — Initiates phone call

**Webhook reply payload** (when user clicks a button):
```json
{
  "type": "buttonReply",
  "phone": "5511999999999",
  "messageId": "msg-id-here",
  "buttonId": "1",
  "buttonLabel": "Agendar Consulta"
}
```

---

## Node 5: Add Note to Lead (Kommo)

```json
{
  "id": "kommo-note-1",
  "name": "Kommo Adicionar Nota",
  "type": "n8n-nodes-kommo.kommo",
  "typeVersion": 1,
  "position": [900, 300],
  "parameters": {
    "resource": "notes",
    "operation": "create",
    "entityType": "leads",
    "entityId": "={{ $json.leadId }}",
    "noteType": "common",
    "text": "Mensagem de boas-vindas enviada via WhatsApp em {{ $now.format('DD/MM/YYYY HH:mm') }}"
  },
  "credentials": {
    "kommoOAuth2Api": {
      "id": "YOUR_KOMMO_CRED_ID",
      "name": "Kommo account"
    }
  }
}
```

---

## HTTP Request Fallback

If community nodes are unavailable, use `n8n-nodes-base.httpRequest`:

### Z-API Send Button Actions (HTTP)

```json
{
  "id": "http-zapi-1",
  "name": "HTTP Z-API Botões",
  "type": "n8n-nodes-base.httpRequest",
  "typeVersion": 4.2,
  "position": [700, 500],
  "parameters": {
    "method": "POST",
    "url": "https://api.z-api.io/instances/YOUR_INSTANCE_ID/token/YOUR_TOKEN/send-button-actions",
    "sendBody": true,
    "bodyParameters": {
      "parameters": [
        {
          "name": "=phone",
          "value": "={{ $json.phone }}"
        },
        {
          "name": "message",
          "value": "Olá! Bem-vindo."
        },
        {
          "name": "buttons",
          "value": "=[{\"id\":\"1\",\"label\":\"Agendar\"}]"
        }
      ]
    }
  }
}
```

**Critical**: Use `=phone` (with `=` prefix) in `bodyParameters.name` to prevent n8n from escaping the value.

---

## Key Integration Points

| From | To | Data Transform |
|---|---|---|
| Kommo webhook | Kommo Get Lead | `$json.body.leads.add[0].id` |
| Kommo Lead | Code node | Extract `contacts[0].custom_fields_values` (PHONE) |
| Code node | Z-API | `phone` (E.164), `contactName`, `leadId` |
| Z-API | Kommo Note | `leadId` for `entityId` |

---

## Button Reply Handler (Separate Workflow)

Create a second workflow triggered by Z-API `ONMESSAGESTATUS` or `ONMESSAGE` webhook:

```json
{
  "id": "button-handler-1",
  "name": "Z-API Resposta Botão",
  "type": "n8n-nodes-base.webhook",
  "typeVersion": 2,
  "position": [100, 300],
  "parameters": {
    "httpMethod": "POST",
    "path": "zapi-button-reply",
    "responseMode": "onReceived"
  }
}
```

**IF node** routing by `buttonId`:
```json
{
  "id": "if-route-1",
  "name": "Roteamento por Botão",
  "type": "n8n-nodes-base.if",
  "typeVersion": 2.2,
  "parameters": {
    "conditions": {
      "options": {
        "caseSensitive": true,
        "leftValue": "",
        "typeValidation": "strict"
      },
      "conditions": [
        {
          "id": "cond-1",
          "leftValue": "={{ $json.buttonId }}",
          "rightValue": "1",
          "operator": {
            "type": "string",
            "operation": "equals"
          }
        }
      ]
    }
  }
}
```

---

## Credentials Required

| Service | Credential Type | Key Fields |
|---|---|---|
| Kommo | `kommoOAuth2Api` | Client ID, Client Secret, OAuth callback |
| Z-API | `zApi` | ID Instance, Token, Client Token |

---

## Files Referenced

- `n8n-stack-kommo/SKILL.md` — Kommo operations and custom fields
- `n8n-stack-kommo/NODE_EXAMPLES.md` — Full Kommo node configurations
- `n8n-stack-zapi/SKILL.md` — Z-API message types and webhooks
- `n8n-stack-zapi/NODE_EXAMPLES.md` — Full Z-API node configurations
