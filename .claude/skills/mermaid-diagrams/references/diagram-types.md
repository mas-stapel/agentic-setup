# Mermaid Diagram Types — Complete Syntax Reference

Full syntax reference for all Mermaid diagram types. Load this file when the
SKILL.md quick reference isn't sufficient for the diagram being produced.

---

## Table of Contents

1. [flowchart — complete syntax](#1-flowchart--complete-syntax)
2. [sequenceDiagram — complete syntax](#2-sequencediagram--complete-syntax)
3. [stateDiagram-v2 — complete syntax](#3-statediagram-v2--complete-syntax)
4. [classDiagram — complete syntax](#4-classdiagram--complete-syntax)
5. [erDiagram](#5-erdiagram)
6. [timeline](#6-timeline)
7. [block-beta](#7-block-beta)
8. [Common pitfalls and rendering gotchas](#8-common-pitfalls-and-rendering-gotchas)

---

## 1. `flowchart` — Complete Syntax

### Direction

```
flowchart TD   %% top-down
flowchart LR   %% left-right
flowchart BT   %% bottom-top
flowchart RL   %% right-left
```

### Node Shapes

```
A[Rectangle]
B(Rounded rectangle)
C([Stadium / pill])
D[[Subroutine / double bracket]]
E[(Database / cylinder)]
F((Circle))
G>Flag / asymmetric]
H{Diamond / decision}
I{{Hexagon}}
J[/Parallelogram — slant right/]
K[\Parallelogram — slant left\]
L[/Trapezoid\]
M[\Trapezoid inverted/]
```

### Edge Types

```
A --> B             %% Solid arrow
A --- B             %% Solid line, no arrow
A -- label --> B    %% Labeled solid arrow
A -. label .-> B    %% Labeled dashed arrow
A -.-> B            %% Dashed arrow
A ==> B             %% Thick arrow
A == label ==> B    %% Labeled thick arrow
A ~~~ B             %% Invisible link (for layout only)
```

### Chained Links

Multiple edges in one statement:

```
A --> B --> C --> D
A & B --> C & D
```

### Subgraphs

```
subgraph GroupID [Display Name]
    direction LR
    A --> B
end

subgraph TopLevel
    subgraph Nested
        C --> D
    end
    TopLevel --> Nested
end
```

Note: `direction` inside a subgraph overrides the diagram's global direction for nodes within that subgraph only.

### Comments

```
%% This is a comment — ignored by the renderer
```

### Node Styling with `classDef`

```
classDef rustStyle fill:#b7410e,color:#fff,stroke:#8b3008
classDef tsStyle fill:#3178c6,color:#fff,stroke:#235f9e

class RustNode rustStyle
class TSNode tsStyle
```

Apply to multiple nodes at once: `class A,B,C myStyle`

### Link / Click Events

```
click A href "https://example.com" _blank
click B call myCallback()
```

---

## 2. `sequenceDiagram` — Complete Syntax

### Participant Declaration

```
sequenceDiagram
    participant A as Alice
    participant B as Bob
    actor U as User
```

`actor` renders as a stick figure; `participant` renders as a box. Declare participants explicitly to control ordering — undeclared participants appear in the order they first appear in messages.

### Create and Destroy

```
create participant C as Cache
A->>C: store(key, value)

destroy C
C-->>A: ok
```

### Message Arrows

```
A->>B: Synchronous call (solid, filled head)
A-->>B: Response / async (dashed, filled head)
A-)B: Async without waiting (solid, open head)
A--)B: Async response (dashed, open head)
A-xB: Call with failure (solid, cross)
A--xB: Response failure (dashed, cross)
```

### Activation Bars

```
activate A
A->>B: request
B-->>A: response
deactivate A
```

Shorthand using `+` / `-`:

```
A->>+B: request
B-->>-A: response
```

### Notes

```
Note right of A: Text on the right
Note left of B: Text on the left
Note over A,B: Text spanning both A and B
```

### Loops

```
loop Every 5 seconds
    A->>B: heartbeat
    B-->>A: ack
end
```

### Conditionals

```
alt Condition is true
    A->>B: success path
else Condition is false
    A->>B: fallback path
end

opt Only if user is admin
    A->>B: admin action
end
```

### Parallel Execution

```
par Thread 1
    A->>B: task 1
and Thread 2
    A->>C: task 2
end
```

### Critical Section

```
critical Acquire resource lock
    A->>B: acquire(lock)
option Timeout
    A->>B: release(lock)
end
```

### Break / Abort

```
break On parse error
    A->>B: error("invalid XML")
end
```

### Box Grouping (visual background)

```
box rgba(0,128,255,0.1) Frontend Layer
    participant UI
    participant Store
end
box rgba(200,100,0,0.1) Backend Layer
    participant Rust
end
```

### Autonumber

Add `autonumber` after `sequenceDiagram` to prefix each message with an incrementing number:

```
sequenceDiagram
    autonumber
    A->>B: first (1)
    B-->>A: second (2)
```

---

## 3. `stateDiagram-v2` — Complete Syntax

### Basic States and Transitions

```
stateDiagram-v2
    [*] --> Idle
    Idle --> Running : start
    Running --> Idle : stop
    Running --> [*] : complete
```

`[*]` is the initial state when used as source, and the final/terminal state when used as target.

### Named States

For long labels, declare a state alias:

```
state "Parsing XML file" as Parsing
state "Waiting for user input" as Waiting
[*] --> Waiting
Waiting --> Parsing : file dropped
```

### Composite (Nested) States

```
state Importing {
    [*] --> Scanning
    Scanning --> Reading
    Reading --> [*]
}
[*] --> Importing
Importing --> Done
```

### Concurrency (Parallel Regions)

Use `--` to split a composite state into parallel regions:

```
state Uploading {
    [*] --> Compressing
    Compressing --> Sending
    --
    [*] --> Logging
    Logging --> [*]
}
```

### Fork / Join

```
state fork_state <<fork>>
state join_state <<join>>

[*] --> fork_state
fork_state --> StateA
fork_state --> StateB
StateA --> join_state
StateB --> join_state
join_state --> [*]
```

### Notes

```
Idle : This is a note attached to Idle
note right of Running
    This state handles
    all active processing
end note
```

### Direction Override

```
stateDiagram-v2
    direction LR
    [*] --> A
    A --> B
```

---

## 4. `classDiagram` — Complete Syntax

### Class Definition

```
classDiagram
    class Animal {
        +String name
        +Int age
        -String secret
        #String protected
        ~String packagePrivate
        +makeSound() String
        +move(distance Int) void
        $staticMethod() void
        *abstractMethod() void
    }
```

Visibility: `+` public, `-` private, `#` protected, `~` package/internal.
Modifiers: `$` static, `*` abstract.

### Classifiers / Stereotypes

```
class Duck {
    <<interface>>
    +quack() void
}

class Color {
    <<enumeration>>
    RED
    GREEN
    BLUE
}

class Template {
    <<Abstract>>
}
```

### Relationships

```
A <|-- B         %% B inherits from A (inheritance)
A *-- B          %% B is part of A (composition)
A o-- B          %% B aggregates A (aggregation)
A --> B          %% A associates with B
A -- B           %% Link (solid, no arrow)
A ..> B          %% Dependency (dashed arrow)
A ..|> B         %% B implements A (dashed inheritance)
A .. B           %% Link (dashed, no arrow)
```

### Cardinality Labels

```
Customer "1" --> "*" Order : places
Order "1" --> "1..*" LineItem : contains
```

### Link and Callback

```
link Animal "https://example.com/animal" _blank
callback Track "showTrackDetail" "Open track detail"
```

### Namespaces

```
namespace CoreTypes {
    class Track
    class Collection
}
namespace DTOs {
    class TrackDto
    class CollectionDto
}
```

---

## 5. `erDiagram`

Entity-Relationship diagrams for data schema documentation.

```
erDiagram
    COLLECTION {
        string version
        int totalTracks
    }
    TRACK {
        string trackId PK
        string name
        string artist
        string averageBpm
        string tonality
        date dateAdded
        boolean isNew
    }
    PLAYLIST_NODE {
        string nodeId PK
        string name
        string nodeType
        string parentId FK
    }
    COLLECTION ||--o{ TRACK : "contains"
    COLLECTION ||--o{ PLAYLIST_NODE : "organises"
    PLAYLIST_NODE ||--o{ PLAYLIST_NODE : "nests"
    PLAYLIST_NODE }o--o{ TRACK : "references"
```

### Relationship Cardinality

| Left | Right | Meaning |
|---|---|---|
| `||` | `||` | Exactly one |
| `||` | `o|` | Zero or one |
| `||` | `}|` | One or more |
| `||` | `o{` | Zero or more |
| `o|` | `}|` | Zero or one to one or more |

### Attribute Keys

Append `PK`, `FK`, or `UK` after the type to label primary, foreign, and unique keys:

```
TRACK {
    string trackId PK
    string collectionId FK
    string upc UK
}
```

---

## 6. `timeline`

Simple timelines for roadmaps, project slices, and chronological events.

```
timeline
    title Sonic Ledger — Development Roadmap

    section Foundation
        Slice 1 : Tauri shell, XML import, collection display

    section DJ Workflow
        Slice 2 : Add Tracks, Export, New Additions node

    section Intelligence
        Slice 3 : AI review, cloud metadata enrichment

    section Polish
        Slice 4 : Persistent settings, onboarding, auto-update
```

Notes:

- `title` is optional but recommended.
- `section` creates a labeled row group.
- Multiple events per section: just list them under the same `section` header with `:` separators.

---

## 7. `block-beta`

High-level architecture diagrams using block layout — good for system context / C4-style overviews.

```
block-beta
    columns 3

    Frontend:1
    space
    Backend:1

    block:Frontend
        UI["React + Tauri WebView"]
        Store["Zustand Store"]
    end

    Ipc(["IPC Bridge"])

    block:Backend
        Cmd["Tauri Commands"]
        Core["sonic-ledger-core"]
        FS["File System"]
    end

    Frontend --> Ipc
    Ipc --> Backend
    Core --> FS
```

Notes:

- `columns N` sets the grid width.
- `block:ID … end` groups nodes into a labelled container.
- `space` inserts an empty cell.
- Nodes inside blocks use the same shape syntax as `flowchart`.

---

## 8. Common Pitfalls and Rendering Gotchas

### Special characters in labels must be quoted

Characters that break parsing when unquoted: `:` `(` `)` `[` `]` `{` `}` `"` `>` `<` `#`

```
%% ❌ Breaks rendering
A[invoke("importCollection")]

%% ✅ Wrap in double quotes
A["invoke('importCollection')"]
```

### Node ID collisions

Every node ID in a `flowchart` must be unique within the diagram. Reusing an ID references the existing node — it does not create a new one. This is intentional for sharing nodes but causes confusing diagrams when accidental.

```
%% ❌ Accidental reuse — B is referenced, not redefined
A --> B[First node]
C --> B[Second node]  %% "Second node" label is ignored — B already exists as "First node"
```

### Semicolons are optional but consistent

Mermaid allows (but does not require) semicolons at the end of statements. Be consistent within a diagram; mixing semicolons and no-semicolons is valid but messy.

### `stateDiagram-v2` vs `stateDiagram`

Always use `stateDiagram-v2`. The original `stateDiagram` is the legacy syntax and lacks composite states, concurrency, and fork/join. The two are not interchangeable.

### `classDiagram` generic types

Use `~` to denote generics (TypeScript angle brackets aren't valid in Mermaid labels):

```
class Collection {
    +Map~String, Track~ tracks    %% represents Map<String, Track>
    +Option~String~ version       %% represents Option<String>
}
```

### `sequenceDiagram` activation mismatch

Every `activate A` must have a matching `deactivate A`. Unmatched activations cause rendering failures in some renderers (notably GitHub). Use the `+`/`-` shorthand to avoid mismatches:

```
A->>+B: request   %% activates B
B-->>-A: response %% deactivates B
```

### Long labels and line wrapping

Mermaid does not wrap long node labels automatically. Break manually with `<br/>` in quoted labels:

```
A["This is a very long label<br/>split across two lines"]
```

### `%%` comments vs `//` comments

Only `%%` is valid for Mermaid comments. `//` and `/* */` are not valid and will cause parse errors.

### Renderer differences

Mermaid syntax is parsed client-side. Minor version differences between:

- **GitHub Markdown** — uses a pinned Mermaid version; some newer features may not render.
- **mermaid.live** — always uses the latest release; most complete support.
- **VS Code** (Markdown Preview Mermaid Support) — uses a separately pinned version.

When in doubt, test the output at [mermaid.live](https://mermaid.live) which provides real-time parse error feedback.

### Escaping backslashes in flowchart labels

Inside a node label string, backslashes must be escaped: `\\n` not `\n`.

### `flowchart` vs `graph`

`graph` is the legacy keyword for flowcharts. `flowchart` is the modern equivalent and supports all the same syntax plus additional features (subgraph direction, etc.). Always use `flowchart`.
