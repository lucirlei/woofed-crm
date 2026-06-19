# Chatwoot Integration Improvement Plan

## Context

This plan addresses the main risks found in the current Chatwoot integration review. The integration already covers setup, contact synchronization, message delivery, webhooks, embedded dashboard views, and MCP tools, but a few areas should be hardened before expanding the feature set.

## Goals

- Keep Chatwoot behavior account-scoped and safe in multi-tenant environments.
- Remove legacy or misleading code paths around integration status.
- Improve observability and error handling without hiding unexpected failures.
- Harden public webhook and attachment handling.
- Make message delivery work consistently across development and production endpoints.
- Preserve current user-facing behavior while adding tests for every changed branch.

## Work plan

### 1. Fix account scoping for conversation links

**Problem:** `Contact::Integrations::Chatwoot::GenerateConversationLink` currently fetches `Apps::Chatwoot.first`, which can generate links using another account's integration in multi-account environments.

**Planned changes:**

- Fetch the Chatwoot integration from the contact's account instead of using the first global integration.
- Prefer an active integration when multiple records exist.
- Return the existing `no_chatwoot_or_id` error when the contact has no account-level Chatwoot integration or no `chatwoot_id`.
- Add request/model specs proving that one account cannot generate a link using another account's Chatwoot integration.

**Suggested files:**

- `app/models/contact/integrations/chatwoot/generate_conversation_link.rb`
- `spec/models/contact/integrations/chatwoot/generate_conversation_link_spec.rb`

### 2. Remove or replace the legacy `actives` scope

**Problem:** `Apps::Chatwoot` defines `scope :actives, -> { where(active: true) }`, but the current schema uses `status` instead of an `active` boolean.

**Planned changes:**

- Search for all usages of `actives`.
- If unused, remove the scope.
- If used, replace it with the enum-generated `active` scope or an explicit status query.
- Add/adjust specs to document status-based filtering.

**Suggested files:**

- `app/models/apps/chatwoot.rb`
- `spec/models/apps/chatwoot_spec.rb`

### 3. Make token validation failures observable

**Problem:** `Apps::Chatwoot#valid_token?` rescues every exception and returns `false`, which can hide programming errors and make production issues hard to diagnose.

**Planned changes:**

- Replace the bare rescue with explicit handling for expected network/API/JSON errors.
- Log enough context to diagnose failures without exposing `chatwoot_user_token`.
- Keep returning `false` for expected token validation failures.
- Let unexpected exceptions raise in test/development or log them clearly before returning according to product expectations.
- Add specs for suspended accounts, API errors, malformed responses, and successful administrator validation.

**Suggested files:**

- `app/models/apps/chatwoot.rb`
- `app/models/apps/chatwoot/api_client.rb`
- `spec/models/apps/chatwoot_spec.rb`
- `spec/models/apps/chatwoot/api_client_spec.rb`

### 4. Harden webhook authentication

**Problem:** Incoming Chatwoot webhooks are currently identified by `embedding_token` in the URL. There is no additional signature or shared-secret verification.

**Planned changes:**

- Check whether the deployed Chatwoot version can send webhook signatures or custom headers.
- If supported, store a webhook secret per integration and validate incoming signatures.
- If signatures are not supported, introduce a separate webhook token distinct from `embedding_token` so embedded UI auth and webhook auth do not share the same secret.
- Ensure invalid webhook authentication returns a non-success status and does not enqueue processing.
- Add request specs for valid token, invalid token, inactive integration, and missing/invalid signature or webhook token.

**Suggested files:**

- `app/models/apps/chatwoot.rb`
- `app/controllers/apps/chatwoots_controller.rb`
- `app/use_cases/accounts/apps/chatwoots/webhooks/process_webhook.rb`
- `db/migrate/*`
- `spec/requests/apps/chatwoots_controller_spec.rb`
- `spec/use_cases/accounts/apps/chatwoots/webhooks/process_webhook_spec.rb`

### 5. Harden attachment download from webhooks

**Problem:** Webhook message import downloads attachment URLs directly with `URI.open`, without visible URL allow-listing, size limits, or timeout controls.

**Planned changes:**

- Replace direct `URI.open` usage with a small downloader service that validates URL scheme and host expectations.
- Add open/read timeouts and a maximum byte-size guard.
- Treat failed downloads as an observable failed attachment import while preserving the message event when appropriate.
- Avoid leaking full signed URLs in logs.
- Add specs for successful attachment import, HTTP failure, unsupported scheme, oversized file, and timeout behavior.

**Suggested files:**

- `app/use_cases/accounts/apps/chatwoots/webhooks/import_message.rb`
- `app/services` or an existing service namespace for a downloader object
- `spec/integration/use_cases/accounts/apps/chatwoots/webhooks/events/message_spec.rb`

### 6. Make attachment message delivery respect endpoint scheme

**Problem:** Message delivery with attachments always sets `use_ssl = true`, which can break local or internal HTTP Chatwoot endpoints.

**Planned changes:**

- Derive SSL usage from the parsed endpoint scheme.
- Keep HTTPS as the expected production path, but support HTTP endpoints in development/test when configured.
- Consider using Faraday multipart for consistency with the no-attachment path.
- Add specs for HTTP and HTTPS endpoints with attachments.

**Suggested files:**

- `app/use_cases/accounts/apps/chatwoots/send_message.rb`
- `spec/integration/use_cases/accounts/apps/chatwoots/send_message_spec.rb`

### 7. Review account scoping across all Chatwoot flows

**Problem:** Several flows use the first Chatwoot integration on an account, and at least one flow uses the first integration globally. The product currently appears to allow one integration per account, but the code should make this explicit and safe.

**Planned changes:**

- Document whether the product supports exactly one Chatwoot integration per account or multiple integrations.
- If exactly one is supported, enforce this with validation and account-scoped lookup helpers.
- If multiple are supported, add explicit selection rules for export, message delivery, conversation links, and MCP tool usage.
- Add specs around cross-account isolation.

**Suggested files:**

- `app/controllers/accounts/apps/chatwoots_controller.rb`
- `app/models/apps/chatwoot.rb`
- `app/models/contact.rb`
- `app/tools/apps/chatwoots/list_tool.rb`
- `app/tools/events/send_chatwoot_message_tool.rb`

### 8. Improve sync and delivery error reporting

**Problem:** Some sync and send flows return `{ error: ... }` but do not consistently surface failures, retry strategy, or structured logs.

**Planned changes:**

- Standardize return objects for Chatwoot use cases.
- Log structured failure context without sensitive tokens.
- Decide which jobs should retry and which failures should mark integration/contact/event state.
- Add specs for API failure paths in import, export, conversation creation, and message delivery.

**Suggested files:**

- `app/use_cases/accounts/apps/chatwoots/sync_import_contacts.rb`
- `app/use_cases/accounts/apps/chatwoots/export_contact.rb`
- `app/use_cases/accounts/apps/chatwoots/get_conversations.rb`
- `app/use_cases/accounts/apps/chatwoots/create_conversation.rb`
- `app/use_cases/accounts/apps/chatwoots/messages/delivery_job.rb`

## Recommended implementation order

1. Fix account-scoped conversation links because it is the clearest multi-tenant correctness risk.
2. Remove the legacy `actives` scope to avoid future runtime errors.
3. Fix attachment delivery SSL handling because it is small and easy to test.
4. Improve token validation observability.
5. Harden webhook authentication.
6. Harden attachment downloads.
7. Review broader account scoping and sync error reporting.

## Testing strategy

- Prefer request specs or integrated specs where possible, using WebMock for Chatwoot HTTP calls.
- Keep examples behavior-focused and group related assertions in one example.
- Add explicit cross-account isolation coverage for account-scoping changes.
- Add failure-path coverage for each new guard or branch.
- Ensure every changed branch has patch coverage before merging.

## Rollout notes

- Webhook authentication changes may require a migration and a deployment step to update existing Chatwoot webhook URLs or secrets.
- If a separate webhook token is introduced, existing integrations should receive a backfilled token before the controller starts requiring it.
- Attachment download limits should be configurable so production can tune limits without code changes.
- Any change to integration selection should be communicated to MCP tool consumers if it changes required parameters.
