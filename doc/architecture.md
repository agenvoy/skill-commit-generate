# commit-generate - Architecture

> Back to [README](../README.md)

## Overview

```mermaid
graph TB
    User[User invokes /commit-generate] --> Skill[SKILL.md rules]
    Skill --> Input[Input layer<br/>four parallel reads]
    Input --> Diff[git diff --cached<br/>content source]
    Input --> Status[git status --short<br/>unstaged reminder]
    Input --> Context[git log / branch<br/>wording reference]
    Diff --> Detect[Multi-Topic Detection]
    Detect --> Upgrade[Tag Upgrade Signals]
    Upgrade --> Tags[Tag Priority]
    Tags --> Format[Output Format Rules]
    Status --> Output[Bilingual commit message]
    Context --> Format
    Format --> Output
```

## Module: Input

Reads four sources in one parallel round; an empty diff stops with an error and never falls back to the working tree.

```mermaid
graph TB
    subgraph Input
        A[git diff --cached] --> B{Output empty?}
        S[git status --short] --> R[Match same-module unstaged files]
        L[git log --oneline -10] --> W[Wording reference]
        Br[git branch --show-current] --> W
        B -->|Yes| C[Emit error message]
        B -->|No| D[Forward to Multi-Topic Detection]
    end
    C --> Stop[Stop]
    R --> Reminder[Reminder line above the message]
    W --> Wording[Module names and phrasing]
```

## Module: Multi-Topic Detection

Decides whether a single diff mixes unrelated intents and, if so, emits a split recommendation.

```mermaid
graph TB
    subgraph MultiTopic[Multi-Topic Detection]
        A[Receive diff] --> B{Touches 2+ primary tags?}
        A --> C{Spans 2+ unrelated modules?}
        A --> D{Contains 3+ unrelated topics?}
        B -->|Any true| E[Mark as multi-topic]
        C -->|Any true| E
        D -->|Any true| E
        B -->|All false| F[Mark as single-topic]
        C -->|All false| F
        D -->|All false| F
    end
    E --> SplitWarn[Emit split recommendation]
    F --> Upgrade[Tag Upgrade Scan]
    SplitWarn --> Upgrade
```

## Module: Tag Upgrade Signals

Scans signals top-down; any hit forces an upgrade and blocks downgrades to `feat` or `update`.

```mermaid
graph TB
    subgraph Upgrade[Tag Upgrade Signals]
        A[Receive diff] --> B{Breaking signals hit?}
        B -->|Yes| C[Tag = breaking]
        B -->|No| D{Security signals hit?}
        D -->|Yes| E[Tag = security]
        D -->|No| F[Match intent via tag priority]
    end
    C --> Out[Emit tag]
    E --> Out
    F --> Out
```

## Module: Output Format

Builds the English subject and Traditional Chinese body from a single tag.

```mermaid
graph LR
    subgraph Format[Output Format]
        T[Tag] --> EN[English subject<br/>imperative, ≤ 72 chars]
        T --> ZH[Traditional Chinese body<br/>verb-first, ≤ 50 chars]
    end
    EN --> Msg[tag: English description<br/>tag: 中文描述]
    ZH --> Msg
```

## Data Flow

```mermaid
sequenceDiagram
    participant User
    participant Agent as Agent Harness
    participant Skill as SKILL.md
    participant Git

    User->>Git: git add <files>
    User->>Agent: /commit-generate
    Agent->>Skill: Load skill definition
    par Parallel reads
        Skill->>Git: git diff --cached
        Skill->>Git: git status --short
        Skill->>Git: git log --oneline -10
        Skill->>Git: git branch --show-current
    end
    Git-->>Skill: Four results
    alt Diff empty
        Skill-->>User: No staged changes
    else Diff present
        Skill->>Skill: Multi-topic detection
        Skill->>Skill: Tag upgrade scan
        Skill->>Skill: Apply output format
        opt Same-module files unstaged
            Skill-->>User: Reminder line
        end
        opt Multi-topic
            Skill-->>User: Split recommendation
        end
        Skill-->>User: Bilingual commit message
    end
```

## Tag Decision State Machine

```mermaid
stateDiagram-v2
    [*] --> ScanBreaking
    ScanBreaking --> Breaking: Breaking signal hit
    ScanBreaking --> ScanSecurity: No hit
    ScanSecurity --> Security: Security signal hit
    ScanSecurity --> MatchIntent: No hit
    MatchIntent --> Resolved: Pick tag by priority
    Breaking --> Resolved
    Security --> Resolved
    Resolved --> [*]
```
