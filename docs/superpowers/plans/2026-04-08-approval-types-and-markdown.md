# Approval Types Expansion + Markdown Rendering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add 5 new approval types to the constants and render all approval body fields as markdown instead of plain text.

**Architecture:** Two files change — the shared constants package gets 5 new type string literals, and the UI ApprovalPayload component gets new labels, icons, a generic renderer for the new types, and `<MarkdownBody>` swapped in for plain-text fields.

**Tech Stack:** TypeScript, React, lucide-react (icons), `MarkdownBody` component (already in `ui/src/components/MarkdownBody.tsx`), Vitest + jsdom for tests.

---

### Task 1: Add new approval types to shared constants

**Files:**
- Modify: `packages/shared/src/constants.ts`

- [ ] **Step 1: Update `APPROVAL_TYPES`**

Open `packages/shared/src/constants.ts` and replace the existing `APPROVAL_TYPES` array:

```ts
export const APPROVAL_TYPES = [
  "hire_agent",
  "approve_ceo_strategy",
  "budget_override_required",
  "request_board_approval",
  "resource_request",
  "external_communication",
  "spend_approval",
  "client_commitment",
  "policy_change",
] as const;
export type ApprovalType = (typeof APPROVAL_TYPES)[number];
```

- [ ] **Step 2: Verify typecheck passes**

```bash
pnpm -r typecheck
```

Expected: all packages pass, no errors.

- [ ] **Step 3: Commit**

```bash
git add packages/shared/src/constants.ts
git commit -m "feat(shared): add resource_request, external_communication, spend_approval, client_commitment, policy_change approval types"
```

---

### Task 2: Add labels, icons, and generic renderer in ApprovalPayload

**Files:**
- Modify: `ui/src/components/ApprovalPayload.tsx`

- [ ] **Step 1: Write a failing test for a new type**

Open `ui/src/components/ApprovalPayload.test.tsx` and add this test inside the existing `describe("ApprovalPayloadRenderer")` block:

```ts
it("renders resource_request payload without falling back to raw JSON", () => {
  const root = createRoot(container);

  act(() => {
    root.render(
      <ApprovalPayloadRenderer
        type="resource_request"
        payload={{
          title: "Access to SendGrid API",
          description: "Need a **SendGrid** API key to send transactional emails.",
        }}
      />,
    );
  });

  expect(container.textContent).toContain("Access to SendGrid API");
  expect(container.textContent).toContain("SendGrid");
  expect(container.textContent).not.toContain("\"description\"");

  act(() => {
    root.unmount();
  });
});
```

- [ ] **Step 2: Run the test to verify it fails**

```bash
pnpm test:run -- --reporter=verbose ui/src/components/ApprovalPayload.test.tsx
```

Expected: FAIL — the new type falls through to `CeoStrategyPayload` which may render raw JSON.

- [ ] **Step 3: Update `ApprovalPayload.tsx` — imports, labels, icons, generic renderer, and markdown**

Replace the entire contents of `ui/src/components/ApprovalPayload.tsx` with:

```tsx
import {
  UserPlus,
  Lightbulb,
  ShieldAlert,
  ShieldCheck,
  KeyRound,
  Send,
  CreditCard,
  Handshake,
  ScrollText,
} from "lucide-react";
import { formatCents } from "../lib/utils";
import { MarkdownBody } from "./MarkdownBody";

export const typeLabel: Record<string, string> = {
  hire_agent: "Hire Agent",
  approve_ceo_strategy: "CEO Strategy",
  budget_override_required: "Budget Override",
  request_board_approval: "Board Approval",
  resource_request: "Resource Request",
  external_communication: "External Communication",
  spend_approval: "Spend Approval",
  client_commitment: "Client Commitment",
  policy_change: "Policy Change",
};

function firstNonEmptyString(...values: unknown[]): string | null {
  for (const value of values) {
    if (typeof value === "string" && value.trim().length > 0) {
      return value.trim();
    }
  }
  return null;
}

export function approvalSubject(payload?: Record<string, unknown> | null): string | null {
  return firstNonEmptyString(
    payload?.title,
    payload?.name,
    payload?.summary,
    payload?.recommendedAction,
  );
}

/** Build a contextual label for an approval, e.g. "Hire Agent: Designer" */
export function approvalLabel(type: string, payload?: Record<string, unknown> | null): string {
  const base = typeLabel[type] ?? type;
  const subject = approvalSubject(payload);
  if (subject) {
    return `${base}: ${subject}`;
  }
  return base;
}

export const typeIcon: Record<string, typeof UserPlus> = {
  hire_agent: UserPlus,
  approve_ceo_strategy: Lightbulb,
  budget_override_required: ShieldAlert,
  request_board_approval: ShieldCheck,
  resource_request: KeyRound,
  external_communication: Send,
  spend_approval: CreditCard,
  client_commitment: Handshake,
  policy_change: ScrollText,
};

export const defaultTypeIcon = ShieldCheck;

function PayloadField({ label, value }: { label: string; value: unknown }) {
  if (!value) return null;
  return (
    <div className="flex items-center gap-2">
      <span className="text-muted-foreground w-20 sm:w-24 shrink-0 text-xs">{label}</span>
      <span>{String(value)}</span>
    </div>
  );
}

function SkillList({ values }: { values: unknown }) {
  if (!Array.isArray(values)) return null;
  const items = values
    .filter((value): value is string => typeof value === "string")
    .map((value) => value.trim())
    .filter(Boolean);
  if (items.length === 0) return null;

  return (
    <div className="flex items-start gap-2">
      <span className="text-muted-foreground w-20 sm:w-24 shrink-0 text-xs pt-0.5">Skills</span>
      <div className="flex flex-wrap gap-1.5">
        {items.map((item) => (
          <span
            key={item}
            className="rounded bg-muted px-1.5 py-0.5 font-mono text-[11px] text-muted-foreground"
          >
            {item}
          </span>
        ))}
      </div>
    </div>
  );
}

export function HireAgentPayload({ payload }: { payload: Record<string, unknown> }) {
  return (
    <div className="mt-3 space-y-1.5 text-sm">
      <div className="flex items-center gap-2">
        <span className="text-muted-foreground w-20 sm:w-24 shrink-0 text-xs">Name</span>
        <span className="font-medium">{String(payload.name ?? "—")}</span>
      </div>
      <PayloadField label="Role" value={payload.role} />
      <PayloadField label="Title" value={payload.title} />
      <PayloadField label="Icon" value={payload.icon} />
      {!!payload.capabilities && (
        <div className="flex items-start gap-2">
          <span className="text-muted-foreground w-20 sm:w-24 shrink-0 text-xs pt-0.5">Capabilities</span>
          <span className="text-muted-foreground">{String(payload.capabilities)}</span>
        </div>
      )}
      {!!payload.adapterType && (
        <div className="flex items-center gap-2">
          <span className="text-muted-foreground w-20 sm:w-24 shrink-0 text-xs">Adapter</span>
          <span className="font-mono text-xs bg-muted px-1.5 py-0.5 rounded">
            {String(payload.adapterType)}
          </span>
        </div>
      )}
      <SkillList values={payload.desiredSkills} />
    </div>
  );
}

export function CeoStrategyPayload({ payload }: { payload: Record<string, unknown> }) {
  const plan = payload.plan ?? payload.description ?? payload.strategy ?? payload.text;
  return (
    <div className="mt-3 space-y-1.5 text-sm">
      <PayloadField label="Title" value={payload.title} />
      {!!plan && (
        <div className="mt-2 rounded-md bg-muted/40 px-3 py-2 max-h-48 overflow-y-auto">
          <MarkdownBody className="text-xs text-muted-foreground">{String(plan)}</MarkdownBody>
        </div>
      )}
      {!plan && (
        <pre className="mt-2 rounded-md bg-muted/40 px-3 py-2 text-xs text-muted-foreground overflow-x-auto max-h-48">
          {JSON.stringify(payload, null, 2)}
        </pre>
      )}
    </div>
  );
}

export function BudgetOverridePayload({ payload }: { payload: Record<string, unknown> }) {
  const budgetAmount = typeof payload.budgetAmount === "number" ? payload.budgetAmount : null;
  const observedAmount = typeof payload.observedAmount === "number" ? payload.observedAmount : null;
  return (
    <div className="mt-3 space-y-1.5 text-sm">
      <PayloadField label="Scope" value={payload.scopeName ?? payload.scopeType} />
      <PayloadField label="Window" value={payload.windowKind} />
      <PayloadField label="Metric" value={payload.metric} />
      {(budgetAmount !== null || observedAmount !== null) ? (
        <div className="rounded-md bg-muted/40 px-3 py-2 text-xs text-muted-foreground">
          Limit {budgetAmount !== null ? formatCents(budgetAmount) : "—"} · Observed {observedAmount !== null ? formatCents(observedAmount) : "—"}
        </div>
      ) : null}
      {!!payload.guidance && (
        <div className="mt-2 rounded-md bg-muted/40 px-3 py-2 max-h-48 overflow-y-auto">
          <MarkdownBody className="text-xs text-muted-foreground">{String(payload.guidance)}</MarkdownBody>
        </div>
      )}
    </div>
  );
}

export function BoardApprovalPayload({
  payload,
  hideTitle = false,
}: {
  payload: Record<string, unknown>;
  hideTitle?: boolean;
}) {
  const nextPayload = hideTitle ? { ...payload, title: undefined } : payload;
  return (
    <BoardApprovalPayloadContent payload={nextPayload} />
  );
}

function BoardApprovalPayloadContent({ payload }: { payload: Record<string, unknown> }) {
  const risks = Array.isArray(payload.risks)
    ? payload.risks
        .filter((value): value is string => typeof value === "string")
        .map((value) => value.trim())
        .filter(Boolean)
    : [];
  const title = firstNonEmptyString(payload.title);
  const summary = firstNonEmptyString(payload.summary);
  const recommendedAction = firstNonEmptyString(payload.recommendedAction);
  const nextActionOnApproval = firstNonEmptyString(payload.nextActionOnApproval);
  const proposedComment = firstNonEmptyString(payload.proposedComment);

  return (
    <div className="mt-4 space-y-3.5 text-sm">
      {title && (
        <div className="space-y-1">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-muted-foreground">Title</p>
          <p className="font-medium leading-6 text-foreground">{title}</p>
        </div>
      )}
      {summary && (
        <div className="space-y-1">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-muted-foreground">Summary</p>
          <MarkdownBody className="leading-6 text-foreground/90">{summary}</MarkdownBody>
        </div>
      )}
      {recommendedAction && (
        <div className="rounded-lg border border-amber-500/20 bg-amber-500/10 px-3.5 py-3">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-amber-700 dark:text-amber-300">
            Recommended action
          </p>
          <MarkdownBody className="mt-1 leading-6 text-foreground">{recommendedAction}</MarkdownBody>
        </div>
      )}
      {nextActionOnApproval && (
        <div className="rounded-lg border border-border/60 bg-background/60 px-3.5 py-3">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-muted-foreground">On approval</p>
          <MarkdownBody className="mt-1 leading-6 text-foreground">{nextActionOnApproval}</MarkdownBody>
        </div>
      )}
      {risks.length > 0 && (
        <div className="space-y-1.5">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-muted-foreground">Risks</p>
          <ul className="space-y-1 text-sm text-muted-foreground">
            {risks.map((risk) => (
              <li key={risk} className="flex items-start gap-2">
                <span className="mt-2 h-1.5 w-1.5 rounded-full bg-muted-foreground/60" />
                <span className="leading-6">{risk}</span>
              </li>
            ))}
          </ul>
        </div>
      )}
      {proposedComment && (
        <div className="space-y-1.5">
          <p className="text-[11px] font-medium uppercase tracking-[0.08em] text-muted-foreground">
            Proposed comment
          </p>
          <div className="max-h-48 overflow-auto rounded-lg border border-border/60 bg-muted/50 px-3.5 py-3">
            <MarkdownBody className="text-xs leading-5 text-muted-foreground">{proposedComment}</MarkdownBody>
          </div>
        </div>
      )}
    </div>
  );
}

export function GenericApprovalPayload({ payload }: { payload: Record<string, unknown> }) {
  const title = firstNonEmptyString(payload.title);
  const body = firstNonEmptyString(payload.description, payload.body, payload.summary);

  if (!title && !body) {
    return (
      <pre className="mt-2 rounded-md bg-muted/40 px-3 py-2 text-xs text-muted-foreground overflow-x-auto max-h-48">
        {JSON.stringify(payload, null, 2)}
      </pre>
    );
  }

  return (
    <div className="mt-3 space-y-2 text-sm">
      {title && (
        <p className="font-medium text-foreground">{title}</p>
      )}
      {body && (
        <div className="rounded-md bg-muted/40 px-3 py-2 max-h-64 overflow-y-auto">
          <MarkdownBody className="text-xs text-muted-foreground">{body}</MarkdownBody>
        </div>
      )}
    </div>
  );
}

export function ApprovalPayloadRenderer({
  type,
  payload,
  hidePrimaryTitle = false,
}: {
  type: string;
  payload: Record<string, unknown>;
  hidePrimaryTitle?: boolean;
}) {
  if (type === "hire_agent") return <HireAgentPayload payload={payload} />;
  if (type === "budget_override_required") return <BudgetOverridePayload payload={payload} />;
  if (type === "request_board_approval") {
    return <BoardApprovalPayload payload={payload} hideTitle={hidePrimaryTitle} />;
  }
  if (
    type === "resource_request" ||
    type === "external_communication" ||
    type === "spend_approval" ||
    type === "client_commitment" ||
    type === "policy_change"
  ) {
    return <GenericApprovalPayload payload={payload} />;
  }
  return <CeoStrategyPayload payload={payload} />;
}
```

- [ ] **Step 4: Run the new test to verify it passes**

```bash
pnpm test:run -- --reporter=verbose ui/src/components/ApprovalPayload.test.tsx
```

Expected: all tests pass including the new `resource_request` test.

- [ ] **Step 5: Run full typecheck**

```bash
pnpm -r typecheck
```

Expected: no errors.

- [ ] **Step 6: Commit**

```bash
git add ui/src/components/ApprovalPayload.tsx ui/src/components/ApprovalPayload.test.tsx
git commit -m "feat(ui): expand approval types, add GenericApprovalPayload, render body fields as markdown"
```

---

### Task 3: Final verification

- [ ] **Step 1: Run full test suite**

```bash
pnpm test:run
```

Expected: all test files pass, 0 failures.

- [ ] **Step 2: Run full typecheck**

```bash
pnpm -r typecheck
```

Expected: no errors.
