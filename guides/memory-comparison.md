# Z3rno vs Native Memory in Each Framework

A practical comparison of built-in memory options versus Z3rno across the major AI agent frameworks.

---

## LangChain

**Built-in:** `ConversationBufferMemory`, `ConversationSummaryMemory`, `VectorStoreRetrieverMemory`

LangChain offers several in-process memory classes. They work well for single-session prototypes but have significant gaps in production:

| Aspect | LangChain Native | Z3rno |
|--------|-----------------|-------|
| Storage | In-process Python objects (lost on restart) | Durable server-side storage (Postgres + vector DB) |
| Multi-agent | Each chain has its own memory instance | Shared memory layer across any number of agents |
| Vector search | Requires manually wiring a vector store | Built-in semantic recall with similarity scores |
| Graph context | Not supported | Entity and relationship graph across memories |
| Audit trail | Not supported | Full audit log of every store/recall/forget |
| GDPR delete | Manual implementation | Hard delete with cascade in one API call |

**When to use native:** Quick prototypes, single-session chatbots with no persistence requirement.

**When to use Z3rno:** Production agents that need cross-session memory, multi-agent coordination, or compliance.

---

## CrewAI

**Built-in:** CrewAI has built-in short-term, long-term, and entity memory that persists within a crew run.

| Aspect | CrewAI Native | Z3rno |
|--------|--------------|-------|
| Persistence | Within a crew execution; resets between runs | Permanent across runs, deploys, and restarts |
| Sharing | Scoped to a single crew | Any crew, agent, or external system can read/write |
| Graph relationships | Basic entity extraction | Full graph with typed relationships and traversal |
| Temporal queries | Not supported | Query memories by time range or recency |
| Audit / compliance | Not available | Immutable audit log with operation history |

**When to use native:** Self-contained crew tasks that don't need memory beyond the current run.

**When to use Z3rno:** Crews that should learn over time, share knowledge across runs, or require audit trails.

---

## OpenAI Agents SDK

**Built-in:** The OpenAI Agents SDK is stateless by design. There is no built-in memory. Conversation history is passed explicitly in the `messages` array.

| Aspect | OpenAI Agents (No Memory) | Z3rno |
|--------|--------------------------|-------|
| Persistence | None; developer manages history manually | Server-side durable memory |
| Context window | Entire history must fit in the prompt | Semantic search returns only relevant memories |
| Multi-agent | No shared state between agents | Shared memory accessible by agent ID or user ID |
| Knowledge growth | Static per conversation | Accumulates knowledge over time |

**When to use without Z3rno:** Simple request/response agents with no continuity requirement.

**When to use Z3rno:** Any OpenAI agent that should remember users, past decisions, or context across sessions.

---

## Anthropic / Claude (MCP)

**Built-in:** Claude has no persistent memory. Each conversation starts from scratch (unless you re-inject history manually).

| Aspect | Claude (No Memory) | Claude + Z3rno MCP |
|--------|-------------------|-------------------|
| Persistence | None | Durable store/recall via MCP tools |
| Setup | N/A | `pip install z3rno-mcp` + config snippet |
| Cross-session | Not possible | Memories persist across conversations |
| User preferences | Re-stated every conversation | Stored once, recalled automatically |
| GDPR | N/A | Hard delete and audit log built in |

**When to use without Z3rno:** One-off questions, stateless API calls.

**When to use Z3rno:** Claude Desktop, Cursor, or Claude Code workflows where continuity matters.

---

## Feature Comparison Table

| Feature | LangChain Native | CrewAI Native | OpenAI Agents | Claude (No MCP) | Z3rno |
|---------|:---:|:---:|:---:|:---:|:---:|
| **Persistent storage** | - | Partial | - | - | Yes |
| **Survives restarts** | - | - | - | - | Yes |
| **Vector / semantic search** | Manual setup | Basic | - | - | Yes |
| **Graph context** | - | Basic entities | - | - | Yes |
| **Temporal queries** | - | - | - | - | Yes |
| **Multi-agent sharing** | - | Within crew | - | - | Yes |
| **Multi-tenant** | - | - | - | - | Yes |
| **Audit log** | - | - | - | - | Yes |
| **GDPR hard delete** | - | - | - | - | Yes |
| **TTL / auto-expiry** | - | - | - | - | Yes |
| **Works across frameworks** | - | - | - | - | Yes |

"-" = not available or not built-in.

---

## Summary

Native memory in LangChain and CrewAI is useful for prototyping but falls short in production scenarios that require durability, cross-agent sharing, compliance, or semantic search. OpenAI Agents and Claude have no built-in memory at all.

Z3rno is a dedicated memory layer that plugs into all four frameworks, giving every agent the same persistent, searchable, auditable memory -- regardless of which SDK you build with.

See the integration guides for setup instructions:

- [LangChain](/integrations/langchain)
- [CrewAI](/integrations/crewai)
- [OpenAI Agents](/integrations/openai-agents)
- [Anthropic / Claude](/integrations/anthropic)
