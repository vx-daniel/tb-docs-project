# CLAUDE.md - Rule Engine Section

This file provides guidance for working with the Rule Engine documentation section.

## Section Purpose

This section documents ThingsBoard's data processing pipeline:

- **Rule Chains**: Visual workflows for message processing
- **Rule Nodes**: Individual processing units (filter, transform, action, etc.)
- **Message Flow**: How TbMsg routes through the system
- **TBEL**: ThingsBoard Expression Language for scripting
- **Queues**: Processing strategies and configuration

## File Structure

```
04-rule-engine/
├── README.md                    # Comprehensive rule engine overview
├── rule-chain-structure.md      # Chain composition, root/nested chains
├── message-flow.md              # Message routing, relation types
├── node-categories.md           # Overview of all node types
├── tbel.md                      # TBEL scripting language
├── queues.md                    # Queue configuration
├── node-development-contract.md # Custom node development
└── nodes/                       # Node reference by category
    ├── filter-nodes.md          # Routing and conditions
    ├── enrichment-nodes.md      # Loading additional data
    ├── transformation-nodes.md  # Modifying messages
    ├── action-nodes.md          # Save, alarm, RPC operations
    ├── external-nodes.md        # External system integration
    ├── flow-nodes.md            # Rule chain flow control
    └── analytics-nodes.md       # Data aggregation (PE)
```

## Writing Guidelines

### Audience

Developers building rule chains for IoT data processing. Assume familiarity with basic programming but not necessarily with stream processing or workflow systems.

### Content Pattern

Rule engine documents should include:

1. **Overview** - What the node/concept does
2. **When to Use** - Common scenarios and use cases
3. **Configuration** - Parameters and options
4. **Input/Output** - Message structure before and after
5. **Example** - Practical TBEL/JavaScript code
6. **Pitfalls** - Common mistakes and solutions
7. **See Also** - Related nodes and concepts

### Node Documentation Pattern

For each rule node:

```markdown
## Node Name

**Category**: Filter / Enrichment / Transformation / Action / External / Flow

**Purpose**: One-sentence description

### Configuration

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| ... | ... | ... | ... |

### Input

- **Message Type**: What message types this node handles
- **Expected Data**: Required payload fields

### Output Relations

| Relation | When Used |
|----------|-----------|
| Success | ... |
| Failure | ... |
| ... | ... |

### Example

\`\`\`javascript
// TBEL example
\`\`\`

### Common Pitfalls

| Pitfall | Solution |
|---------|----------|
| ... | ... |
```

### Terminology

- Use "Rule Chain" not "workflow" or "pipeline"
- Use "Rule Node" not "processor" or "handler"
- Use "TbMsg" or "message" for the data object
- Use "Relation" not "connection" or "link" for node outputs
- Use "Originator" for the source entity of a message
- Use "TBEL" (ThingsBoard Expression Language) not "script" when referring to the language

### Diagrams

Use Mermaid diagrams to show:

- Rule chain flow (`graph LR`)
- Message processing (`sequenceDiagram`)
- Decision logic (`flowchart`)
- State transitions (`stateDiagram-v2`)

### Technology-Agnostic Rule

Focus on behavior and configuration, not implementation:

**DO**: "The Script Filter node evaluates a TBEL expression and routes to True or False"
**DON'T**: "TbFilterNodeConfiguration injects ScriptEngine with Nashorn/GraalJS runtime"

**DO**: "Messages flow through nodes sequentially, with each node producing output relations"
**DON'T**: "RuleNodeActor.process() invokes TbNode.onMsg() callback with TbContext"

**DO**: "Deduplication prevents duplicate messages within a time window"
**DON'T**: "DeduplicationProcessor uses ConcurrentHashMap with TTL eviction"

## Reference Sources

When updating this section, cross-reference:

- `~/work/viaanix/thingsboard.github.io-master/docs/user-guide/rule-engine-2-0/` - Official rule engine docs
- `~/work/viaanix/thingsboard-master/rule-engine/rule-engine-components/` - Node implementations
- `~/work/viaanix/thingsboard-master/common/message/` - TbMsg structure

## Related Sections

- `03-actor-system/` - RuleChainActor and RuleNodeActor
- `02-core-concepts/data-model/` - Telemetry and attribute structures
- `06-api-layer/` - Notifications triggered by rule engine
- `08-message-queue/` - Queue processing for rule engine

## Common Tasks

### Documenting a New Rule Node

1. Identify the node category (filter, action, etc.)
2. Add to appropriate `nodes/{category}-nodes.md` file
3. Follow the node documentation pattern above
4. Include working TBEL/JavaScript examples
5. Document all output relations
6. Add common pitfalls
7. Update `node-categories.md` count if needed

### Updating TBEL Documentation

1. Verify against source in `~/work/viaanix/thingsboard-master/`
2. Test all code examples for correctness
3. Ensure examples use TBEL (not JavaScript) as primary
4. Note any version-specific features

### Adding Message Type Documentation

1. Add to message types table in README.md
2. Document payload structure
3. Document metadata fields
4. Show which nodes commonly handle this type

### Cross-Reference Validation

Ensure all `See Also` links point to valid files:

```bash
grep -r "\.\.\/" docs/04-rule-engine/ | grep -o '\.\./[^)]*' | sort -u
```

## Recommended Skills

Use these skills when working on this section:

| Skill | Command | Use For |
|-------|---------|---------|
| **workflow-orchestrator** | `/workflow-orchestrator` | Rule chain design, flow patterns, error handling |
| **javascript-expert** | `/javascript-expert` | TBEL/JavaScript scripting, async patterns |
| **iot-engineer** | `/iot-engineer` | IoT data processing patterns |
| **technical-writer** | `/technical-writer` | Clear documentation structure |

### When to Use Each Skill

- **Designing rule chain patterns**: Use `/workflow-orchestrator` for flow design
- **Writing TBEL examples**: Use `/javascript-expert` for scripting best practices
- **Documenting IoT use cases**: Use `/iot-engineer` for telemetry processing patterns
- **Explaining complex flows**: Use `/technical-writer` for clarity

## Key Rule Engine Concepts

When documenting the rule engine, emphasize:

| Concept | Key Points |
|---------|------------|
| **TbMsg Immutability** | Messages are immutable; transformations create new messages |
| **Relation-Based Routing** | Nodes output to named relations (Success, Failure, True, False) |
| **Root Chain** | Every tenant has one root chain receiving all device messages |
| **Debug Mode** | Captures message snapshots for troubleshooting |
| **Deduplication** | Prevents duplicate processing within time window |
| **Queue Strategies** | BATCH, SEQUENTIAL_BY_ORIGINATOR, etc. |

## Common Pitfalls to Document

Ensure documentation covers these issues:

| Pitfall | Description |
|---------|-------------|
| Unhandled Failure | No Failure relation connected = message marked failed |
| Case-sensitive relations | "success" ≠ "Success" |
| Script null checks | Missing null checks cause runtime exceptions |
| Deduplication scope | Applies to entire message, not individual keys |
| Infinite loops | Scripts with while loops can timeout |
| Debug mode in production | Impacts performance significantly |

## TBEL vs JavaScript

When documenting scripts, prefer TBEL:

| Aspect | TBEL | JavaScript |
|--------|------|------------|
| Performance | Faster (compiled) | Slower (interpreted) |
| Security | Sandboxed | More attack surface |
| Recommendation | Primary choice | Legacy/complex cases |

Always show TBEL examples first, JavaScript as alternative.

## Helpful Paths

- local-skillz: `~/Projects/barf/repo/SKILLS/README.md`
