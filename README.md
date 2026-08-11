# Companion assets — Automate email support with Amazon Connect Customer email AI agents

Deployable assets for the AWS Contact Center blog post of the same name.

## Contents

| File | Purpose |
|---|---|
| `connect-email-infrastructure.yaml` | CloudFormation template. Creates two S3 buckets: one for email messages and attachments (with the CORS policy and bucket policy Amazon Connect Customer requires), one for knowledge base content. Both use SSE-S3, block all public access, and deny non-HTTPS requests. Creates nothing else and never modifies your instance. |
| `sample-email-ai-flow.json` | Importable inbound contact flow. Checks the channel, associates the AI agents domain, inspects the Amazon SES spam verdict, and routes to one of two queues. |
| `kb-content/` | Six short hotel policy documents used as grounding content. Deliberately rule-based so you can tell whether an answer came from your documents or the model. |
| `architecture-email-ai-agents.png` | Architecture diagram used in the post. Exported from draw.io with the editable diagram embedded in the PNG, so you can reopen it in [draw.io](https://app.diagrams.net/) to edit. |

## Deploy

### If your instance already stores email in Amazon S3

Deploy with `CreateEmailStorageBucket=false` and keep the bucket you have. Only the
knowledge base bucket is created. Repointing **Data storage** at a new bucket splits
your email history across two buckets and leaves new mail outside any lifecycle or
retention rules on the old one.

Amazon Connect Customer creates a bucket automatically when you enable email, but
that bucket has **no CORS rule** — we verified this — and attachment sharing needs
one. If you keep your existing bucket, add the CORS rule to it yourself:

```json
[{"AllowedHeaders":["*"],"AllowedMethods":["PUT","GET"],
  "AllowedOrigins":["*.my.connect.aws","*.awsapps.com"],"ExposeHeaders":[]}]
```

The email storage bucket in this template exists to save you that step on a clean
instance. It is not otherwise required.

### Parameters

| Parameter | Default | Notes |
|---|---|---|
| `ConnectInstanceArn` | — | Required |
| `ConnectInstanceAlias` | — | Required. Max 27 characters: the alias plus suffix plus your 12-digit account ID must fit S3's 63-character bucket name limit. |
| `CreateEmailStorageBucket` | `true` | Set `false` to skip the email bucket |

Replace the instance ARN and alias with your own:

```bash
aws cloudformation deploy \
  --template-file connect-email-infrastructure.yaml \
  --stack-name connect-email-ai-poc \
  --parameter-overrides \
      ConnectInstanceArn="arn:aws:connect:us-east-1:111122223333:instance/aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee" \
      ConnectInstanceAlias="my-instance-alias"
```

Retrieve both bucket names:

```bash
aws cloudformation describe-stacks \
  --stack-name connect-email-ai-poc \
  --query "Stacks[0].Outputs[*].[OutputKey,OutputValue]" --output table
```

Upload the knowledge base content:

```bash
aws s3 sync kb-content/ s3://{instance-alias}-connect-kb-content-{account-id}/
```

## Before importing the flow

Replace three placeholder ARNs in `sample-email-ai-flow.json`:

| Placeholder | Replace with |
|---|---|
| `...:assistant/00000000-0000-0000-0000-000000000000` | Your AI agents domain ARN |
| `...queue/00000000-0000-0000-0000-000000000001` | Your main email queue ARN |
| `...queue/00000000-0000-0000-0000-000000000002` | Your spam review queue ARN |

## Notes worth reading before you troubleshoot

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

## Clean up

Empty both buckets before deleting the stack, since CloudFormation cannot remove
buckets that still hold objects:

```bash
aws s3 rm s3://{instance-alias}-connect-email-storage-{account-id}/ --recursive
aws s3 rm s3://{instance-alias}-connect-kb-content-{account-id}/ --recursive
aws cloudformation delete-stack --stack-name connect-email-ai-poc
```

## License

MIT-0. See `LICENSE`.
