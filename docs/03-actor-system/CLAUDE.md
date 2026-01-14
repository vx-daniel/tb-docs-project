# CLAUDE.md - Actor System Section

This file provides guidance for working with the Actor System documentation section.

## Section Purpose

This section documents ThingsBoard's concurrency model based on the Actor pattern:

- **Actor Model**: Message-driven concurrency with isolated state
- **Actor Hierarchy**: AppActor → TenantActor → DeviceActor/RuleChainActor
- **Message Flow**: How messages traverse the actor system
- **Mailbox & Dispatchers**: Thread pools and message queuing

## File Structure

```
03-actor-system/
├── README.md              # Comprehensive actor system documentation
├── message-types.md       # Message categories and routing
├── device-actor.md        # Device actor responsibilities
└── rule-chain-actor.md    # Rule chain actor processing
```

## Writing Guidelines

### Audience

Backend developers understanding concurrency patterns. Assume familiarity with basic threading concepts but not necessarily the Actor model.

### Content Pattern

Actor system documents should include:

1. **Overview** - What the actor/concept does
2. **Why** - Problem it solves, alternative approaches rejected
3. **Message Types** - What messages the actor handles
4. **State** - What data the actor maintains
5. **Lifecycle** - Creation, processing, destruction
6. **Diagrams** - Visual representation of message flow
7. **Pitfalls** - Common issues and mitigations
8. **See Also** - Related documentation

### Terminology

- Use "Actor" for the concurrency entity
- Use "Mailbox" for the message queue per actor
- Use "Dispatcher" for the thread pool
- Use "Message" not "event" or "command" (unless specifically typed)
- Use "tell" for sending messages (fire-and-forget)
- Use "process" for handling messages

### Diagrams

Use Mermaid diagrams to show:

- Message flow (`sequenceDiagram`)
- Actor hierarchy (`graph TB`)
- State machines (`stateDiagram-v2`)
- Processing flow (`flowchart`)

### Technology-Agnostic Rule

Focus on behavioral contracts, not implementation:

**DO**: "Each actor processes one message at a time, eliminating race conditions"
**DON'T**: "TbActorMailbox uses AtomicBoolean compareAndSet for lock-free state"

**DO**: "Actors are created lazily when the first message arrives for them"
**DON'T**: "getOrCreateChildActor() uses ReentrantLock with double-checked locking"

**DO**: "High-priority messages are processed before normal messages"
**DON'T**: "highPriorityMsgs ConcurrentLinkedQueue is polled before normalPriorityMsgs"

**DO**: "Dispatchers provide isolated thread pools for different actor types"
**DON'T**: "TbDispatcher wraps ExecutorService with throughput-limited scheduling"

## Concurrency Concepts to Explain

When documenting, ensure readers understand:

| Concept | Explanation |
|---------|-------------|
| **Sequential Processing** | One message at a time per actor |
| **Isolation** | Actor state not shared with other actors |
| **Location Transparency** | Actors can be local or remote |
| **Supervision** | Parent actors manage child failures |
| **Mailbox Ordering** | Same-sender messages maintain order |

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard-master/common/actor/` - Actor system implementation
- `~/work/viaanix/thingsboard-master/application/src/main/java/org/thingsboard/server/actors/` - Actor implementations
- `~/work/viaanix/thingsboard-master/common/message/` - Message type definitions

## Related Sections

- `01-architecture/` - System-level context for actors
- `02-core-concepts/` - Entities that actors manage
- `04-rule-engine/` - RuleChainActor and RuleNodeActor details
- `11-microservices/` - How actors distribute across services

## Common Tasks

### Documenting a New Actor Type

1. Create `{actor-name}-actor.md`
2. Document message types handled (table format)
3. Document internal state maintained
4. Show message flow with sequence diagram
5. List lifecycle events (init, destroy)
6. Add pitfalls specific to this actor
7. Update README.md with reference

### Updating Message Flow Documentation

1. Verify against source in `~/work/viaanix/thingsboard-master/`
2. Update sequence diagrams for accuracy
3. Check for new message types
4. Validate routing logic descriptions

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/03-actor-system/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **java-expert** | `/java-expert` | Java concurrency, java.util.concurrent, threading patterns |
| **java-architect** | `/java-architect` | Enterprise Java patterns, Spring ecosystem, concurrency design |
| **software-architecture** | `/software-architecture` | Design patterns, actor model concepts |
| **technical-writer** | `/technical-writer` | Clear documentation of complex concepts |

### When to Use Each Skill

- **Documenting concurrency guarantees**: Use `/java-expert` for threading semantics
- **Designing actor hierarchies**: Use `/java-architect` for enterprise concurrency patterns
- **Explaining actor patterns**: Use `/software-architecture` for design rationale
- **Writing message flow docs**: Use `/technical-writer` for clarity
- **Documenting mailbox behavior**: Use `/java-expert` for queue semantics

## Key Actor System Concepts

When documenting the actor system, emphasize:

| Concept | Key Points |
|---------|------------|
| **Single-Threaded Processing** | No synchronization needed within actor |
| **Mailbox** | Dual-priority queues, unbounded by default |
| **Dispatcher** | Thread pool isolation prevents starvation |
| **Supervision** | Parent-child failure handling |
| **Lazy Creation** | Actors created on first message |
| **Message Ordering** | Guaranteed for same sender-receiver pair |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Unbounded queues | Memory growth if processing lags |
| Message loss on shutdown | Messages in flight during destroy |
| Throughput starvation | High-volume actors yield frequently |
| Async callback safety | State access outside actor thread |
| Init retry delays | Exponential backoff can be slow |

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
