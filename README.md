# Allied Henna Demo Agent

An AI-powered multi-channel service orchestration system built on Camunda 8. This project builds on the [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker) by adding intelligent orchestration, multi-channel support, and user-facing automation.

## What is Allied Henna?

Allied Henna is a demonstration of an AI-assisted service agent that can:
- Accept user requests via **Camunda forms, Slack, or Email**
- Use an **AI agent** (Claude via AWS Bedrock) to intelligently select and invoke business operations
- Return results through the **same channel** the user initiated from
- Support **follow-up conversations** with context retention

![Company Services Agent Process Diagram](img/Company%20Services%20Agent.png)

## Architecture: Two-Project Design

This project works in tandem with [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker):

| Component | Purpose | Location |
|-----------|---------|----------|
| **Orchestration Layer** | BPMN processes, forms, multi-channel routing, AI task orchestration | This repo |
| **Execution Layer** | CRM connectors (search, CRUD operations on accounts, companies, employees, etc.) | Worker repo |

**This means:** Without the worker project running, this process cannot complete connector tasks.

## Quick Start

1. **Start the worker first** (handles all CRM operations):
   ```bash
   git clone https://github.com/NPDeehan/search-internal-systems-worker
   cd search-internal-systems-worker
   mvn spring-boot:run
   ```

2. **Import this repo to Camunda**:
   - BPMN files from `bpmn/`
   - Forms from `forms/`
   - Connector templates from `Connector Templates/`

3. **Configure secrets** (see Setup Guide section below)

4. **Deploy and test** using the provided forms

## What is in this repo

This repository contains all orchestration and UI assets:

- **`bpmn/Company Services Agent.bpmn`** — Main multi-channel AI orchestration process (forms, Slack, Email input; AI agent task selection; multi-channel response)
- **`Connector Templates/*.json`** — Camunda connector templates for all task types (references worker implementations)
- **`forms/*.form`** — Camunda user task and start event forms

**What is NOT here:**
- Connector implementations (those are in [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker))
- Worker infrastructure or Spring Boot code

## Main process behavior

### 1. Entry channels

The process can start from:
- Camunda form (`Form_048h1nh`) with `questionFromUser`
- Slack inbound message event
- Email inbound message event

For Slack starts, it posts an initial "eyes" reaction to acknowledge the request.

For Email starts, it maps the email body to `questionFromUser`.

### 2. AI orchestration

The `Allied Henna Agent` ad-hoc subprocess uses an Agentic AI worker (`io.camunda.agenticai:aiagent-job-worker:1`) configured for AWS Bedrock (Claude Sonnet model).

The AI agent chooses and invokes tool tasks from the connector set, then loops with tool results as context.

### 3. User response and follow-up

The process can answer through:
- Camunda user task form (`display-answer-to-user-05en478`)
- Slack outbound connector
- Email outbound connector

For Slack and Email follow-up questions, event-based gateways wait for a response and timeout after `PT5M`.

### 4. Completion/compensation

The process includes compensation handlers for communication acknowledgements (for example, a final Slack reaction and a closure email path).

## Custom Task Types

All custom task connectors are defined via templates in this repository. They are organized by function:

**Search/Query Tasks** (provided by worker):
- `query-for-company` — Find companies in CRM
- `search-account` — Search for accounts
- `search-employee` — Search for employees

**CRUD Tasks** (provided by worker):
- `manage-account-record` — Create/update/delete account
- `manage-company-record` — Create/update/delete company
- `manage-customer-record` — Create/update/delete customer
- `manage-employee-record` — Create/update/delete employee
- `manage-customer-account-link` — Link customer to account
- `manage-insurance-policy` — Manage policy records
- `manage-package` — Manage packages
- `manage-product` — Manage products
- `manage-purchase-item` — Manage purchase items
- `manage-purchase-order` — Manage purchase orders
- `match-customer-with-dri` — Match customers to account representatives

**Note:** All these tasks are implemented as workers in the [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker) project. Without that worker running, these tasks will remain pending.

## Prerequisites & Compatibility

**Required:**
- Camunda 8 (SaaS or self-managed)
- Camunda Web Modeler or Desktop Modeler
- [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker) running and connected to the same cluster
- AWS Bedrock access (for Claude AI agent)

**Optional (for multi-channel support):**
- Slack app with webhook and event subscriptions
- Email account with IMAP/SMTP enabled (Gmail or similar)

**Tested with:**
- Camunda 8.x
- Java 11+
- AWS Bedrock (Claude Sonnet model)

## Setup Guide

### Step 1: Start the worker project

Follow the [search-internal-systems-worker](https://github.com/NPDeehan/search-internal-systems-worker) README. At a high level:

```bash
git clone https://github.com/NPDeehan/search-internal-systems-worker
cd search-internal-systems-worker
mvn clean compile
```

Configure Camunda connection credentials (for the same cluster this BPMN will use), then start:

```bash
mvn spring-boot:run
```

Confirm worker health and that workers are polling your Camunda cluster. If workers are not polling, service tasks in this process will remain pending.

### Step 2: Import assets to Camunda

1. Open Camunda Web Modeler
2. Import BPMN files from `bpmn/`
3. Import forms from `forms/`
4. Import connector templates from `Connector Templates/`

### Step 3: Configure Camunda secrets

The BPMN process references secrets by name. You must create each one in your Camunda 8 cluster's secret store.

**To add secrets in Camunda:**
1. Navigate to **Cluster → Secrets** (or **Settings → Secrets** depending on your version)
2. Click **Create new secret** and enter each name and value exactly as shown below

#### AWS Bedrock (Required)

These enable the AI agent task to call Claude via AWS Bedrock.

| Secret Name | Purpose | How to find it |
|-------------|---------|----------------|
| `AWS_REGION` | AWS region where Bedrock is available | E.g., `us-east-1`, `us-west-2`. Check [AWS Bedrock regions](https://docs.aws.amazon.com/bedrock/latest/userguide/what-is-bedrock.html) |
| `AWS_ACCESS_KEY` | AWS IAM access key ID | AWS Console → IAM → Users → Your user → Security credentials → Access keys → Copy "Access key ID" |
| `AWS_SECRET_KEY` | AWS IAM secret access key | AWS Console → IAM → Users → Your user → Security credentials → Access keys → Copy "Secret access key" (only visible once on creation) |

**IAM Permission Required:** Your AWS user/role must have `bedrock:InvokeModel` permission for the Claude Sonnet model.

#### Slack Integration (Optional)

Only needed if you want to start/respond via Slack.

| Secret Name | Purpose | How to find it |
|-------------|---------|----------------|
| `SLACK_ALLIED_HENNA_OATH_TOKEN` | Slack bot token (note: typo in name is intentional) | Slack workspace → API → Apps → Create app → OAuth & Permissions → Bot token (starts with `xoxb-`) |
| `SLACK_ALLIEDHENNA_WEBHOOK_ID` | Webhook ID for sending Slack replies | Slack workspace → API → Apps → Your app → Interactivity & Shortcuts → Request URL (extract ID from webhook setup) |
| `SLACK_ALLIED_HENNA_SIGINING_SECRET` | Signing secret to verify Slack requests (note: typo is intentional) | Slack workspace → API → Apps → Your app → Basic Information → Signing Secret (scroll down) |

**Slack App Permissions Required:**
- `chat:write` — Send messages
- `reactions:write` — Add emoji reactions
- `commands` — Receive slash commands (if using them)
- Event subscriptions for `message.channels`, `app_mention`

#### Email Integration (Optional)

Only needed if you want to start/respond via Email. Currently configured for Gmail.

| Secret Name | Purpose | How to find it |
|-------------|---------|----------------|
| `EMAIL_USER_NAME_DMN` | Gmail address | Your Gmail address (e.g., `your-email@gmail.com`) |
| `EMAIL_PASSWORD_DMN` | Gmail app password, NOT your regular password | Gmail → Account → Security → App passwords (requires 2FA enabled). Generate a 16-char password for "Mail" and "Windows" |

**Gmail Setup Required:**
1. Enable 2-factor authentication on your Gmail account
2. Generate an [app-specific password](https://support.google.com/accounts/answer/185833) for IMAP/SMTP
3. Use that app password in the secret (NOT your regular Gmail password)

### Step 4: Deploy and test

1. Deploy `Company Services Agent.bpmn`
2. Start an instance via the Camunda form start event
3. Verify that connector tasks are consumed by the running worker
4. Optionally test Slack and Email channels

## Troubleshooting

- **Jobs stuck in "Activated" or "Pending"**:
  - Verify `search-internal-systems-worker` is running and connected to the same cluster.
  - Verify job type names match templates and BPMN task definitions.

- **No Slack start/follow-up events**:
  - Check Slack webhook context and signing secret values in Camunda secrets.
  - Verify Slack app event subscriptions and permissions.

- **No Email start/follow-up events**:
  - Verify IMAP/SMTP credentials and mailbox/folder configuration.
  - Check connector logs and correlation (`In-Reply-To`) behavior.

- **AI agent task fails early**:
  - Verify AWS Bedrock secret values and model availability for configured region.

## Recommended local workflow

1. Start the worker project first
2. Deploy `Company Services Agent.bpmn`
3. Test via Camunda form start event
4. Enable Slack/Email channels once base flow is confirmed
