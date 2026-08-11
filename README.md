# Companion assets — Automate email support with Amazon Connect Customer email AI agents

Deployable assets for the AWS Contact Center blog post of the same name.

**Full walkthrough:** [Automate email support with Amazon Connect Customer email AI agents](https://REPLACE-WITH-BLOG-POST-URL)

This README is the short version: what to deploy, in what order, and the traps
worth knowing. The post explains the why and shows the output.

![Architecture: customer email flows through Amazon SES into Amazon Connect Customer, which stores messages in one Amazon S3 bucket and reads knowledge base content from another. Three AI agents produce an email summary, knowledge base recommendations, and a draft reply in the agent workspace.](architecture-email-ai-agents.png)

## What this sets up

The three system default email AI agents — EmailOverview, EmailGenerativeAnswer,
and EmailResponse — running against your own knowledge base content. No prompt
engineering, no Lambda function, no orchestration code.

**These agents assist a human.** They run when an agent accepts the contact and
display their output for review. They do not reply to customers automatically.

## Contents

| File | Purpose |
|---|---|
| `connect-email-infrastructure.yaml` | CloudFormation template. Creates two S3 buckets: one for email messages and attachments (with the CORS policy and bucket policy Amazon Connect Customer requires), one for knowledge base content. Both use SSE-S3, block all public access, and deny non-HTTPS requests. Creates nothing else and never modifies your instance. |
| `sample-email-ai-flow.json` | Importable inbound contact flow. Checks the channel, associates the AI agents domain, inspects the Amazon SES spam verdict, and routes to one of two queues. |
| `kb-content/` | Six short hotel policy documents used as grounding content. Deliberately rule-based so you can tell whether an answer came from your documents or the model. |
| `architecture-email-ai-agents.png` | Architecture diagram used in the post. Exported from draw.io with the editable diagram embedded in the PNG, so you can reopen it in [draw.io](https://app.diagrams.net/) to edit. |

## Prerequisites

- An Amazon Connect instance, plus its instance ARN and alias
- An administrator security profile on that instance
- Amazon SES set up for email. In the [SES sandbox](https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html)
  inbound email and the agents work, but replies fail because you can only send
  to verified addresses. Request production access, or just
  [verify the address you test from](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html).
- The AWS CLI, configured for the account and Region holding your instance

## Setup

Seven steps. Order matters — later steps consume identifiers from earlier ones.

### 1. Deploy the buckets

```bash
aws cloudformation deploy \
  --template-file connect-email-infrastructure.yaml \
  --stack-name connect-email-ai-poc \
  --parameter-overrides \
      ConnectInstanceArn="arn:aws:connect:us-east-1:111122223333:instance/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" \
      ConnectInstanceAlias="my-instance-alias"
```

| Parameter | Default | Notes |
|---|---|---|
| `ConnectInstanceArn` | — | Required |
| `ConnectInstanceAlias` | — | Required. Max 27 characters: the alias plus suffix plus your 12-digit account ID must fit S3's 63-character bucket name limit. |
| `CreateEmailStorageBucket` | `true` | Set `false` to skip the email bucket. See below. |

Retrieve both bucket names, needed in steps 2 and 3:

```bash
aws cloudformation describe-stacks --stack-name connect-email-ai-poc \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" --output table
```

**If your instance already stores email in Amazon S3,** deploy with
`CreateEmailStorageBucket=false` and keep the bucket you have. Repointing **Data
storage** at a new bucket splits your email history across two buckets and leaves
new mail outside any lifecycle or retention rules on the old one.

Amazon Connect Customer creates a bucket automatically when you enable email, but
that bucket has **no CORS rule** — we verified this — and attachment sharing needs
one. If you keep your existing bucket, add the CORS rule to it yourself:

```json
[{"AllowedHeaders":["*"],"AllowedMethods":["PUT","GET"],
  "AllowedOrigins":["*.my.connect.aws","*.awsapps.com"],"ExposeHeaders":[]}]
```

The email storage bucket in this template exists to save you that step on a clean
instance. It is not otherwise required.

### 2. Enable the email channel

In the Amazon Connect admin website:

- Add the email domain, then under **Data storage** point email message export and
  attachment sharing at the email bucket from step 1. Attachment sharing needs that
  CORS rule; without it the email channel does not work.
- Create an email address. Leave its inbound flow as the default for now — you
  change it in step 4.
- Create **two** queues under **Routing**, **Queues**: one for mail that passes the
  spam check, one for mail flagged as spam. Each needs hours of operation and an
  outbound email configuration pointing at your new address. Note both ARNs from
  **Show additional queue information** — the flow needs them.
- Under **Users**, **Routing profiles**, enable the **Email** channel on your agent
  profile and add both queues.

### 3. Create the AI agents domain and knowledge base

Choose **AI Agents**, **Add domain**, **Create a domain**, and wait for **Active**.
Note the domain ARN (`arn:aws:wisdom:<region>:<account-id>:assistant/<uuid>`).

Upload the grounding content:

```bash
aws s3 sync kb-content/ s3://{instance-alias}-connect-kb-content-{account-id}/
```

Then **Add integration**, **Create a new integration**, **Source** = **Amazon S3**,
and select that bucket. Ingestion takes a few minutes.

Confirm EmailOverview, EmailGenerativeAnswer, and EmailResponse appear on the
**AI agents** tab. They need no setup of their own.

> If you supply your own AWS KMS key for the domain, its policy must grant
> `connect.amazonaws.com` the `kms:Decrypt`, `kms:GenerateDataKey*`, and
> `kms:DescribeKey` permissions, or the email AI agents fail.

### 4. Import the flow and attach it to the address

Replace three placeholder ARNs in `sample-email-ai-flow.json` first:

| Placeholder | Replace with |
|---|---|
| `...:assistant/00000000-0000-0000-0000-000000000000` | Your AI agents domain ARN from step 3 |
| `...queue/00000000-0000-0000-0000-000000000001` | Your main email queue ARN |
| `...queue/00000000-0000-0000-0000-000000000002` | Your spam review queue ARN |

Then **Routing**, **Flows**, **Create flow**, **Inbound flow**, and from the **Save**
menu choose **Import flow**. Open each block to confirm the ARNs resolved, then
**Save** and **Publish**.

**Then set your email address's inbound flow to this flow** under **Channels**,
**Email**. This is mandatory and easy to miss: skip it and email still arrives, but
no session is created and the assistant panel stays empty.

### 5. Grant the assistant permission and create a test user

Under **Users**, **Security profiles**, edit the `Agent` profile: confirm
**Initiate email conversation** under **Contact Control Panel**, then enable
**Connect assistant** with **View** access under **Agent Applications**. Without the
second one the panel never appears at all.

Create a user with that security profile and the routing profile from step 2.

### 6. Customize (optional)

Nothing here is required — the default agents work against your content as soon as
the domain is active. If you do customize, read the gotchas below first, and note
you need **AI agent designer** permissions for **AI agents** and **AI prompts** set
to **Create**; **View** is not sufficient.

### 7. Test

Sign in to `https://{instance-alias}.my.connect.aws/agent-app-v2/` as the test user,
set status to **Available**, and email your new address from an external account.

Nothing happens until you accept the contact. On accept, all three agents run and
populate the panel. The generative answer takes longest, so the panel fills in over
several seconds.

## Gotchas

**The Connect assistant block is two flow actions, not one.** `CreateWisdomSession`
opens the session and `UpdateContactData` writes `$.Wisdom.SessionArn` onto the
contact. Both are in this flow. If you rebuild the flow by hand and omit the
second, no AI agent will ever run: the flow completes, the contact routes
normally, and the assistant panel stays empty with no error logged anywhere.

**Customizations fail silently.** An AI agent that is published and active can
still return nothing. In testing, a custom EmailResponse agent that overrode only
the query reformulation prompt produced no draft at all. Change one agent at a
time and send a test email after each change.

**Publishing an agent does not activate it.** You must also call
`update-assistant-ai-agent` to set it as the domain default. Confirm with:

```bash
aws qconnect get-assistant --assistant-id <YOUR_DOMAIN_ID> \
  --query "assistant.aiAgentConfiguration"
```

Every use case always appears in that output, pre-populated with a system agent,
so a key being present proves nothing. Compare the returned ID against your own.

**`connect:X-SES-SPAM-VERDICT` is not always present.** When absent the flow falls
through to the main queue, which is the safe default. Treat the spam branch as
best-effort and verify against your own mail.

**Flow logging needs a block in the flow.** Enabling contact flow logs at the
instance level is not sufficient; add a **Set logging behavior** block.

**The assistant block bills per contact processed,** which is why the flow checks
the channel first. Keep non-email contacts away from it.

## Troubleshooting

Failures here are usually silent. Two places to look:

- **Connect assistant event logs** — enable delivery with `EVENT_LOGS` as the log
  type. See [Monitor AI agents using CloudWatch](https://docs.aws.amazon.com/connect/latest/adminguide/monitor-ai-agents.html).
- **Session traces** — [`ListSpans`](https://docs.aws.amazon.com/amazon-q-connect/latest/APIReference/API_ListSpans.html)
  returns the per-agent trace, keyed on the session ID in the contact's `WisdomInfo`.

The two sources disagree on generated content. Trust the event logs for what a
model actually returned.

## Clean up

Detach references before deleting their targets: reset the email address inbound
flow to the default, then delete the address, domain, flow, queues, and test user.

Empty both buckets before deleting the stack, since CloudFormation cannot remove
buckets that still hold objects:

```bash
aws s3 rm s3://{instance-alias}-connect-email-storage-{account-id}/ --recursive
aws s3 rm s3://{instance-alias}-connect-kb-content-{account-id}/ --recursive
aws cloudformation delete-stack --stack-name connect-email-ai-poc
```

## License

MIT-0. See `LICENSE`.
