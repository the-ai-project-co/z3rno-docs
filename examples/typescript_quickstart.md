# Z3rno TypeScript SDK -- Quickstart

> **Prerequisite:** A running z3rno-server instance is required. Start one locally
> with `docker compose -f docker-compose.dev.yml up` (see the
> [Quickstart guide](/quickstart) for details), or point `baseUrl` at your
> hosted deployment.

This walkthrough covers every major SDK operation in a single plain TypeScript
file. Run any snippet with `npx tsx quickstart.ts` (Node.js 18+).

---

## 1. Installation

```bash
npm install @z3rno/sdk
```

The SDK has a single runtime dependency (Zod) and ships as dual ESM/CJS.

---

## 2. Import and client setup

```typescript
import {
  Z3rnoClient,
  Z3rnoError,
  AuthenticationError,
  RateLimitError,
  ValidationError,
  NotFoundError,
  ServerError,
  Z3rnoTimeoutError,
  Z3rnoConnectionError,
} from "@z3rno/sdk";

const client = new Z3rnoClient({
  baseUrl: "http://localhost:8000",
  apiKey: "z3rno_sk_test_...",
  timeout: 30_000,   // 30 seconds (default)
  maxRetries: 3,     // retries on 5xx, 429, timeouts, and network errors
});

const AGENT_ID = "550e8400-e29b-41d4-a716-446655440000";
```

You can also omit `baseUrl` and `apiKey` and let the client read from the
`Z3RNO_BASE_URL` and `Z3RNO_API_KEY` environment variables.

---

## 3. Store 10 diverse memories

Z3rno supports four memory types: `semantic` (long-term facts), `episodic`
(event-based), `working` (short-lived scratchpad), and `procedural` (how-to
knowledge).

```typescript
// --- Semantic memories (long-term facts) ---
const prefMemory = await client.store({
  agentId: AGENT_ID,
  content: "User prefers dark mode, large fonts, and high-contrast themes.",
  memoryType: "semantic",
  metadata: { source: "settings-page" },
  importance: 0.85,
});
console.log(`Stored preference: ${prefMemory.id}`);

const techMemory = await client.store({
  agentId: AGENT_ID,
  content: "The team's production stack is Next.js 14, PostgreSQL 16, and Valkey 8.",
  memoryType: "semantic",
  metadata: { source: "onboarding-doc", team: "platform" },
});

const policyMemory = await client.store({
  agentId: AGENT_ID,
  content: "Company policy requires MFA for all production deployments.",
  memoryType: "semantic",
  metadata: { source: "security-handbook", department: "engineering" },
});

// --- Episodic memories (events tied to interactions) ---
const bugReport = await client.store({
  agentId: AGENT_ID,
  content: "User reported a 502 error on the /api/checkout endpoint at 14:32 UTC.",
  memoryType: "episodic",
  metadata: { severity: "high", ticket: "INC-4421" },
});

const meetingMemory = await client.store({
  agentId: AGENT_ID,
  content: "During the sprint retro on 2026-04-18 the team agreed to adopt trunk-based development.",
  memoryType: "episodic",
  metadata: { meeting: "sprint-retro-42" },
});

const feedbackMemory = await client.store({
  agentId: AGENT_ID,
  content: "User gave positive feedback on the new onboarding wizard: 'much clearer than before'.",
  memoryType: "episodic",
  metadata: { sentiment: "positive", feature: "onboarding-wizard" },
});

// --- Working memories (short-lived scratchpad) ---
const scratchMemory = await client.store({
  agentId: AGENT_ID,
  content: "Current debugging context: investigating slow queries on the orders table.",
  memoryType: "working",
  metadata: { task: "perf-investigation" },
  ttlSeconds: 3600, // auto-expires after 1 hour
});

const draftMemory = await client.store({
  agentId: AGENT_ID,
  content: "Draft response to user: 'We have identified the root cause and a fix is in progress.'",
  memoryType: "working",
  ttlSeconds: 1800,
});

// --- Procedural memories (how-to knowledge) ---
const deployProcedure = await client.store({
  agentId: AGENT_ID,
  content: "To deploy to staging: run `make build`, push to the staging branch, then approve the GitHub Actions workflow.",
  memoryType: "procedural",
  metadata: { scope: "ci-cd" },
  importance: 0.9,
});

const rollbackProcedure = await client.store({
  agentId: AGENT_ID,
  content: "Rollback procedure: revert the merge commit on main, then trigger a manual deploy from the Actions tab.",
  memoryType: "procedural",
  metadata: { scope: "ci-cd", criticality: "high" },
  importance: 0.95,
});

console.log("All 10 memories stored.");
```

---

## 4. Recall with a semantic query

Pass a natural-language query and the server ranks memories by a combined
relevance score (similarity + importance + recency).

```typescript
const results = await client.recall({
  agentId: AGENT_ID,
  query: "What are the user's UI preferences?",
  topK: 5,
  similarityThreshold: 0.5,
});

console.log(`Found ${results.total} results:`);
for (const item of results.results) {
  console.log(
    `  [${item.relevance_score.toFixed(2)}] (${item.memory_type}) ${item.content}`
  );
}
```

---

## 5. Recall with filters

You can narrow recall by `memoryType` and control how many results come back
with `topK`.

```typescript
// Only procedural memories, top 3 results
const procedures = await client.recall({
  agentId: AGENT_ID,
  query: "How do I deploy?",
  memoryType: "procedural",
  topK: 3,
});

console.log(`Procedural hits: ${procedures.total}`);
for (const item of procedures.results) {
  console.log(`  ${item.content}`);
}

// Only episodic memories with metadata filter
const incidents = await client.recall({
  agentId: AGENT_ID,
  query: "recent errors",
  memoryType: "episodic",
  filters: { severity: "high" },
  topK: 5,
});

console.log(`High-severity episodes: ${incidents.total}`);
```

---

## 6. Graph-augmented recall

When memories are linked via relationships, the recall engine can traverse
the knowledge graph to surface contextually related memories that a
pure-vector search might miss.

```typescript
// First, store a memory that is linked to the deploy procedure
const hotfixProcedure = await client.store({
  agentId: AGENT_ID,
  content: "Hotfix procedure: cherry-pick the fix onto a hotfix/* branch, then follow the standard deploy procedure.",
  memoryType: "procedural",
  metadata: { scope: "ci-cd", criticality: "critical" },
  relationships: [
    {
      targetMemoryId: deployProcedure.id,
      relationshipType: "derived_from",
      weight: 0.9,
    },
    {
      targetMemoryId: rollbackProcedure.id,
      relationshipType: "related_to",
      weight: 0.7,
    },
  ],
});

console.log(`Hotfix procedure stored: ${hotfixProcedure.id}`);

// Now recall -- the server follows graph edges to include related memories
// even if they do not directly match the query embedding.
const graphResults = await client.recall({
  agentId: AGENT_ID,
  query: "How do I ship a hotfix?",
  topK: 5,
});

console.log("Graph-augmented recall results:");
for (const item of graphResults.results) {
  console.log(`  [${item.relevance_score.toFixed(2)}] ${item.content}`);
}
```

---

## 7. Temporal queries (as_of)

Z3rno uses temporal versioning -- every update creates a new version rather
than overwriting. Use `asOf` to see what the agent's memory looked like at a
specific point in time.

```typescript
// Update a memory to create a new version
const updatedPref = await client.updateMemory(prefMemory.id, {
  content: "User prefers light mode and compact layouts (changed 2026-04-20).",
  importance: 0.9,
});
console.log(`Updated preference memory: ${updatedPref.id}`);

// Recall as of *before* the update -- returns the original content
const beforeUpdate = await client.recall({
  agentId: AGENT_ID,
  query: "user UI preferences",
  // asOf is passed as an ISO 8601 timestamp in the recall request body
  // (the SDK model supports it; pass it via the filters or directly if
  // your server version supports the asOf field on the recall endpoint)
});

console.log("Current recall (post-update):");
for (const item of beforeUpdate.results.slice(0, 2)) {
  console.log(`  ${item.content}`);
}
```

---

## 8. Memory history

Retrieve the full version history of any memory. Each version carries
`valid_from` and `valid_to` timestamps.

```typescript
const history = await client.getMemoryHistory(prefMemory.id);

console.log(`Memory ${history.memory_id} has ${history.total} version(s):`);
for (const version of history.versions) {
  const until = version.valid_to ?? "current";
  console.log(`  [${version.valid_from} -> ${until}] ${version.content}`);
}
```

---

## 9. Forget (soft delete and hard delete)

Soft delete marks the memory as deleted but retains the data for audit
purposes. Hard delete permanently removes it.

```typescript
// Soft delete (default) -- data retained, excluded from future recalls
const softResult = await client.forget({
  agentId: AGENT_ID,
  memoryId: scratchMemory.id,
  reason: "Working memory no longer needed",
});
console.log(
  `Soft-deleted ${softResult.deleted_count} memory, hard_deleted=${softResult.hard_deleted}`
);

// Hard delete -- permanently removed, cannot be recovered
const hardResult = await client.forget({
  agentId: AGENT_ID,
  memoryId: draftMemory.id,
  hardDelete: true,
  reason: "User requested permanent deletion (GDPR)",
});
console.log(
  `Hard-deleted ${hardResult.deleted_count} memory, hard_deleted=${hardResult.hard_deleted}`
);

// Cascade delete -- also removes related memories
const cascadeResult = await client.forget({
  agentId: AGENT_ID,
  memoryId: hotfixProcedure.id,
  cascade: true,
  reason: "Deprecated procedure tree",
});
console.log(
  `Cascade-deleted ${cascadeResult.deleted_count} + ${cascadeResult.cascade_count} related`
);
```

---

## 10. Session lifecycle

Sessions group related memory operations (e.g., a single conversation turn or
a task). Memories stored during an active session are automatically associated
with it.

```typescript
// Start a session
const session = await client.startSession({
  agentId: AGENT_ID,
  sessionType: "conversation",
});
console.log(`Session started: ${session.session_id}`);

// Store memories within the session context
await client.store({
  agentId: AGENT_ID,
  content: "User asked: 'Can you summarize last week's incidents?'",
  memoryType: "episodic",
  metadata: { sessionId: session.session_id },
});

await client.store({
  agentId: AGENT_ID,
  content: "Agent responded with a summary of 3 incidents from the past 7 days.",
  memoryType: "episodic",
  metadata: { sessionId: session.session_id },
});

// End the session -- returns summary stats
const summary = await client.endSession(session.session_id);
console.log(
  `Session ended after ${summary.duration_seconds}s, ${summary.memory_count} memories created`
);
```

---

## 11. Audit trail query

The audit log records every `store`, `recall`, `forget`, and `update`
operation. It supports pagination for large histories.

```typescript
let page = 1;
let hasMore = true;

while (hasMore) {
  const auditPage = await client.audit({
    agentId: AGENT_ID,
    page,
    pageSize: 20,
  });

  console.log(`--- Audit page ${auditPage.page} (${auditPage.total} total) ---`);
  for (const entry of auditPage.entries) {
    console.log(
      `  ${entry.created_at} | ${entry.operation.padEnd(8)} | memory=${entry.memory_id ?? "n/a"}`
    );
  }

  hasMore = auditPage.has_next;
  page++;
}
```

---

## 12. Error handling

The SDK exports a typed error hierarchy. Every error extends `Z3rnoError`, so
you can catch broadly or match specific subclasses.

```typescript
async function safeStore() {
  try {
    await client.store({
      agentId: AGENT_ID,
      content: "Some important fact",
      memoryType: "semantic",
    });
  } catch (error) {
    if (error instanceof AuthenticationError) {
      // 401 -- bad or expired API key
      console.error("Authentication failed. Check your API key.");
    } else if (error instanceof ValidationError) {
      // 400 / 422 -- malformed request
      console.error(`Validation error: ${error.message}`);
    } else if (error instanceof NotFoundError) {
      // 404 -- resource does not exist
      console.error(`Not found: ${error.message}`);
    } else if (error instanceof RateLimitError) {
      // 429 -- too many requests
      console.error(`Rate limited. Retry after ${error.retryAfter} seconds.`);
    } else if (error instanceof ServerError) {
      // 5xx -- server-side failure (already retried)
      console.error(`Server error (${error.statusCode}): ${error.message}`);
    } else if (error instanceof Z3rnoTimeoutError) {
      // Request exceeded the configured timeout
      console.error(`Timed out after ${error.timeout}ms`);
    } else if (error instanceof Z3rnoConnectionError) {
      // Network / DNS failure
      console.error(`Connection failed: ${error.message}`);
    } else if (error instanceof Z3rnoError) {
      // Catch-all for any other SDK error
      console.error(`Z3rno error (${error.statusCode}): ${error.message}`);
    } else {
      throw error; // unexpected non-SDK error
    }
  }
}

await safeStore();
```

---

## 13. Retry behavior

The SDK automatically retries on transient failures. No additional code is
required -- retries are built into every method call.

**What is retried:**

| Condition | Strategy |
|---|---|
| HTTP 5xx (server error) | Exponential backoff: 1s, 2s, 4s |
| HTTP 429 (rate limit) | Honors the `Retry-After` header |
| Network / DNS failure | Exponential backoff: 1s, 2s, 4s |
| Timeout (`AbortError`) | Exponential backoff: 1s, 2s, 4s |

**What is NOT retried:**

- 4xx client errors (401, 404, 400, 422) -- these indicate a bug in the
  calling code and are thrown immediately.

**Configuration:**

```typescript
const client = new Z3rnoClient({
  baseUrl: "http://localhost:8000",
  apiKey: "z3rno_sk_test_...",
  timeout: 10_000,   // per-attempt timeout (ms)
  maxRetries: 5,     // up to 5 retries (6 total attempts)
});
```

After all retry attempts are exhausted, the SDK throws the appropriate typed
error (`ServerError`, `RateLimitError`, `Z3rnoTimeoutError`, or
`Z3rnoConnectionError`).

---

## Full runnable script

Save the snippets above into a single file and run it:

```bash
npx tsx quickstart.ts
```

Or compile first:

```bash
npx tsc quickstart.ts --module nodenext --moduleResolution nodenext --target es2022
node quickstart.js
```
