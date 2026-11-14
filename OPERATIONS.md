# Ghost SES proxy: deployment and handover

Last verified: 2026-11-14.

## Backup automation

The proxy is included in `/opt/docker-backups` alongside the four Ghost instances.
The backup uses SQLite's online backup API and preserves source/configuration,
events, suppressions, and send-deduplication records. See
[/opt/docker-backups/README.md](/opt/docker-backups/README.md) for operation and
recovery, and [/opt/docker-backups/SYNOLOGY.md](/opt/docker-backups/SYNOLOGY.md) for
the manual NAS snapshot/offsite tasks. Never start the restored SQS poller or email
sending during an isolated restore drill.

## Current deployment

The proxy translates Ghost's Mailgun newsletter API requests into AWS SES sends.
Delivery events travel from SES through SNS and SQS to the proxy's SQLite database;
Ghost reads them through the proxy's Mailgun-compatible events API.

| Item | Current value |
| --- | --- |
| Proxy directory | `/opt/ghost-ses-proxy` |
| Active Compose file | `/opt/ghost-ses-proxy/docker-compose.yml` |
| Proxy configuration | `/opt/ghost-ses-proxy/.env` |
| Ghost configuration | `/opt/ghost_bok/.env` |
| Ghost Compose file | `/opt/ghost_bok/compose.yml` |
| Proxy service / Docker hostname | `ghost-ses-proxy` |
| Proxy container | `ghost-ses-proxy-ghost-ses-proxy-1` |
| Ghost container | `ghost_bok-ghost-1` |
| Shared external Docker network | `mailgun-ses-proxy_SESproxy` |
| Ghost API base URL | `http://ghost-ses-proxy:3003/v3` |
| Host health endpoint | `http://127.0.0.1:3003/health` |
| Sending domain | `blogokomiksach.pl` |
| AWS region | `eu-north-1` |
| SES configuration set | `ghost-ses-proxy` |
| SQS queue | `ghost-ses-events` (full URL in proxy `.env`) |
| SQLite storage | `/opt/ghost-ses-proxy/data/ses-proxy.db`, mounted at `/data` |
| Send concurrency | `5` (SES account rate cap) |
| Restart policy | `unless-stopped` |
| Deployed application version | `1.1.1` |

The host port binds only to `127.0.0.1`. Ghost reaches the container directly
over the shared Docker network. The external network must already exist when
deploying on a new host. `ghost_bok/compose.yml` already joins this network.

## Configuration saved during this session

- Created the active `docker-compose.yml` from the repository's example, with
  persistent storage, localhost port binding, `.env` loading, and the shared network.
- The user changed the proxy domain from `kumate.ai` to `blogokomiksach.pl`.
- Corrected the Ghost API URL from `http://mailgun-ses-proxy:8080/v3` to
  `http://ghost-ses-proxy:3003/v3` and recreated Ghost to apply it.
- Confirmed the Ghost API key matches the proxy API key without recording secrets.

Relevant Ghost settings:

```dotenv
bulkEmail__mailgun__baseUrl=http://ghost-ses-proxy:3003/v3
bulkEmail__mailgun__domain=blogokomiksach.pl
```

`bulkEmail__mailgun__apiKey` must match `PROXY_API_KEY` in the proxy `.env`.
The proxy also needs `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, and
`SQS_QUEUE_URL`. Keep their values out of documentation and command output.

Ghost's SMTP `mail__*` settings are separate from these bulk newsletter settings.
The proxy URL belongs in `bulkEmail__mailgun__baseUrl`.

An earlier read of Ghost's database found the old Mailgun URL and domain settings.
No database settings were changed during this session; the deployed configuration
uses the environment settings above. Recheck database settings if these environment
overrides are removed later.

## Deploy or apply configuration changes

Run in `/opt/ghost-ses-proxy`:

```sh
docker compose config --quiet
docker compose up -d --build --wait --wait-timeout 120
docker compose ps
docker compose logs --tail 50
```

After changing `/opt/ghost_bok/.env`, run in `/opt/ghost_bok`:

```sh
docker compose up -d --no-deps --wait --wait-timeout 90 ghost
```

After changing only the proxy `.env`, run in `/opt/ghost-ses-proxy`:

```sh
docker compose up -d --no-deps --wait --wait-timeout 90 ghost-ses-proxy
```

Compose recreates a container when its configuration changes. A plain container
restart does not load new environment values. Docker access is required.
Preserve the `data` directory: it contains events, suppressions, and deduplication
records. Take a consistent SQLite backup before migrations or destructive changes.

## Validation

Host health check:

```sh
curl --fail http://127.0.0.1:3003/health
```

Validate the actual Ghost environment, DNS, and authenticated API access without
sending mail or printing the API key:

```sh
docker exec -i ghost_bok-ghost-1 node <<'JS'
(async () => {
  const base = process.env.bulkEmail__mailgun__baseUrl;
  const domain = process.env.bulkEmail__mailgun__domain;
  await require('dns').promises.lookup(new URL(base).hostname);
  const health = await fetch(new URL('/health', base));
  if (!health.ok) throw new Error('Health HTTP ' + health.status);
  const response = await fetch(base + '/' + domain + '/events?limit=1', {
    headers: {
      Authorization: 'Basic ' + Buffer.from(
        'api:' + process.env.bulkEmail__mailgun__apiKey
      ).toString('base64')
    }
  });
  if (!response.ok) throw new Error('Events HTTP ' + response.status);
  const body = await response.json();
  if (!Array.isArray(body.items)) throw new Error('Invalid events response');
  for (const url of Object.values(body.paging)) new URL(url);
  console.log('PASS: DNS, health, authentication, and events response');
})().catch(error => {
  console.error(error.message);
  process.exitCode = 1;
});
JS
```

Verified during the session:

- Compose deployment completed and the proxy reported healthy.
- Unauthenticated and incorrect-key API requests returned HTTP 401.
- Authenticated requests and event pagination returned HTTP 200.
- Ghost resolved the corrected hostname and successfully authenticated to the proxy.
- Both deployed environments used `blogokomiksach.pl` after the correction.
- SQS `GetQueueAttributes` succeeded; the queue policy permitted the corresponding
  SNS topic to publish. No SQS polling errors appeared in the inspected logs.

Validation limits:

- The IAM user denied `ses:GetSendQuota`, `ses:GetAccountSendingEnabled`,
  `ses:GetIdentityVerificationAttributes`, and `ses:DescribeConfigurationSet`.
  Therefore SES quota, sending status, identity verification, and the configuration
  set/event destination were not independently verified. These read denials alone
  do not establish whether `ses:SendRawEmail` is allowed.
- No test email was sent by the assistant. End-to-end delivery and the complete
  SES → SNS → SQS event path were not independently verified.
- `/health` checks the application and SQLite, not AWS readiness.
- The build reported three moderate dependency vulnerabilities and the runtime
  emitted an AWS SDK warning about future Node.js 22 requirements. These were
  not remediated during this deployment. The Dockerfile currently uses Node 20.

## Resolved error: EAI_AGAIN mailgun-ses-proxy

Ghost was configured with `http://mailgun-ses-proxy:8080/v3`, but the deployed
service's Docker DNS name is `ghost-ses-proxy` and it listens on port `3003`.
The old hostname was not a network alias. Correcting the URL and recreating Ghost
resolved the DNS failure; authenticated connectivity then passed.

If it recurs, check the running Ghost environment, the service's network aliases,
and membership of both containers in the shared network. Within a Ghost container,
`localhost` refers to Ghost itself, not to the proxy or host.

## Multiple hosted Ghost instances

Use a separate proxy service for each Ghost instance with this implementation.
Although the routes accept different domain names, the proxy does not isolate sites:

- One API key is shared by all requests to a proxy.
- Events are not scoped by domain.
- Suppressions use a shared `(email, type)` key.
- Send deduplication uses `(ghost_email_id, recipient)`, without a site identifier.

For each additional Ghost instance, use a unique proxy service/DNS name, API key,
data directory, and SES configuration set → SNS topic → SQS queue pipeline. Host
ports must be unique if published; each container can still listen on port 3003.
The AWS account can be shared.

Do not point separate proxies at the same SQS queue: they compete for messages,
so events can be consumed by the wrong proxy. Changing only `MAILGUN_DOMAIN` does
not add tenant isolation.

## Resolved error: Duplicate header 'Message-ID' (2026-11-14)

Ghost 6.75's mailgun client sends newsletters with a per-recipient Message-ID:
`h:Message-Id: <%recipient.message_id%>` plus a deterministic `message_id` entry
in recipient-variables. The proxy copied every `h:*` header verbatim into the
MIME on top of its own `Message-ID:`, so SES rejected all 290 sends of the
2026-11-14 digest with "Duplicate header 'Message-ID'". Ghost recorded the
email as `submitted` (the proxy ACKed first), so the failure was silent.

Fixed in proxy v1.1.1 (`lib/send-email.js`): `h:Message-Id` is excluded from
the custom headers and substituted per recipient from recipient-variables, with
a fallback to the batch Message-ID when the template does not resolve.

Verification performed without sending mail, all passing: host health endpoint,
unauthenticated/wrong-key requests rejected with 401, authenticated events
request with valid pagination from inside the Ghost container, compose
deployment healthy, SQLite row counts preserved, SQS poller started cleanly.

## Resolved error: SES Maximum sending rate exceeded (2026-11-14)

At send concurrency 10, 25 of 290 recipients in the first retried batch failed
with "Maximum sending rate exceeded" (SES account-level per-second rate cap).
`SEND_CONCURRENCY` was lowered to `5` in the proxy `.env`.

The proxy records a dedup claim before the SES attempt and does not roll it
back on failure, so failed recipients stay claimed. The remaining 25 were
re-delivered by clearing only their claims and re-triggering the email retry:
Ghost resubmitted all 290 recipients, the proxy skipped the 265 already claimed
(`sent=25 skipped=265 failed=0`) and every recipient ended with a delivery
event. No duplicates are possible while claims exist.

## Retrying a silently failed newsletter

Ghost trusts the proxy's immediate "Queued. Thank you." ACK, so a proxy-side
failure records the email as `submitted` with zero deliveries and no automatic
retry. Recovery procedure, in order:

1. Fix the proxy first; a retry against a broken proxy fails silently again.
2. Clear dedup claims for recipients that never reached SES (otherwise the
   retry is skipped entirely):

   ```sh
   # backed up first: /data/ses-proxy.db -> ses-proxy-backup-*.db
   DELETE FROM send_claims  WHERE ghost_email_id = '<email id>';
   DELETE FROM recipient_emails WHERE ghost_email_id = '<email id>';
   DELETE FROM message_map WHERE ghost_email_id = '<email id>';
   ```

3. Ghost's retry endpoint only accepts emails in `failed` status, and its
   batch lock only re-claims batches in `pending`/`failed` — a `submitted`
   batch is skipped as "already submitted". Reset both rows in the Ghost
   database before retrying:

   ```sql
   UPDATE emails SET status = 'failed', error = '<reason>' WHERE id = '<email id>';
   UPDATE email_batches SET status = 'failed', error_message = '<reason>'
     WHERE email_id = '<email id>';
   ```

4. Trigger the retry with `PUT /ghost/api/admin/emails/:id/retry/`:

   - Integration admin API keys always receive 403 here; the `retry` action is
     not in Ghost's permission map. A staff API key belonging to the Owner
     user works.
   - Admin keys authenticate as a signed JWT: header `kid` = key id, HS256,
     secret `Buffer.from(secret, 'hex')`, audience `/admin/`, maxAge 5m,
     header `Authorization: Ghost <token>` (no `api` scheme word).
   - Ghost 6.x requires `Accept-Version: v6.75`-style versions; plain `v5.0`
     or `v6.0` return 406.
   - Call the canonical site URL directly. `http://localhost:2368` returns a
     301 to the site URL and fetch drops the Authorization header on the
     cross-origin redirect.
   - `PUT`, not `POST`.

5. Watch the proxy log for `Batch ... done: sent=N skipped=N failed=N` and
   confirm delivery events appear for every recipient. Do not re-clear claims
   for recipients that already show a delivery event.

## Repository status

The proxy source fix (v1.1.1), the deployment Compose file, and this document
are committed to the repository. The proxy `.env` and `data/` are git-ignored.
The upstream example Compose file was not modified.
