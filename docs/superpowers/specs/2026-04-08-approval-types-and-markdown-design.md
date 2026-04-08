# Approval Types Expansion + Markdown Rendering

**Date:** 2026-04-08
**Scope:** Presentation layer only — no database or API changes.

## Problem

1. The approval type list has only 4 entries, which does not cover the range of decisions a founder needs to gate.
2. Approval body fields are rendered as plain text, even when agents write them in markdown.

## Changes

### 1. Expanded Approval Types

Add 5 new types to `APPROVAL_TYPES` in `packages/shared/src/constants.ts`:

| Type key | Label | When it fires |
|---|---|---|
| `resource_request` | Resource Request | Agent needs a credential, API key, or tool access |
| `external_communication` | External Communication | Agent wants to send something on the founder's behalf |
| `spend_approval` | Spend Approval | One-time purchase or expense above a threshold |
| `client_commitment` | Client Commitment | Agent is about to make a promise or agreement with a client |
| `policy_change` | Policy Change | Agent proposes a change to company operations or SOPs |

Existing types are unchanged:
- `hire_agent`
- `approve_ceo_strategy`
- `budget_override_required`
- `request_board_approval`

### 2. Markdown Rendering

In `ui/src/components/ApprovalPayload.tsx`:

- Replace plain-text `whitespace-pre-wrap` blocks in `CeoStrategyPayload` and `BoardApprovalPayloadContent` with `<MarkdownBody>` (already exists at `ui/src/components/MarkdownBody.tsx`).
- The `summary`, `recommendedAction`, `nextActionOnApproval`, `proposedComment`, and `plan` fields all get markdown rendering.

### 3. New Generic Payload Renderer

Add a `GenericApprovalPayload` component for the 5 new types. It renders:
- `title` as a heading (plain text)
- `description` or `body` as markdown via `<MarkdownBody>`
- Falls back to a JSON dump if neither field is present

Wire the new types in `ApprovalPayloadRenderer`:
- `resource_request` → `GenericApprovalPayload`
- `external_communication` → `GenericApprovalPayload`
- `spend_approval` → `GenericApprovalPayload`
- `client_commitment` → `GenericApprovalPayload`
- `policy_change` → `GenericApprovalPayload`

### 4. Icons

Add icons for each new type from `lucide-react` (already a dependency):

| Type | Icon |
|---|---|
| `resource_request` | `KeyRound` |
| `external_communication` | `Send` |
| `spend_approval` | `CreditCard` |
| `client_commitment` | `Handshake` |
| `policy_change` | `ScrollText` |

## Files Changed

| File | Change |
|---|---|
| `packages/shared/src/constants.ts` | Add 5 values to `APPROVAL_TYPES` |
| `ui/src/components/ApprovalPayload.tsx` | New types, icons, labels, markdown rendering, `GenericApprovalPayload` |

## Out of Scope

- No new routes, schema changes, or migrations
- No changes to how approvals are created or stored
- No changes to approval workflow (approve/reject/revision)
