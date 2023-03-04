# MillWheel
Strong productions: 

each operation is atomic(may be with transaction)
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U->>P: record
P->>P: deduplication, process, update state, and checkpoint
P->>U: ack
U->>U: delete checkpoint
P->>D: record
D->>D: deduplication, process, update state, and checkpoint
D->>P: ack
P->>P: delete checkpoint
```
if record sending fails, then the record is re-sent
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U--xP: record
U->>U: timeout
U->>P: record
P->>P: deduplication, process, update state, and checkpoint
P->>U: ack
U->>U: delete checkpoint
P->>D: record
```
If the process/state update/checkpoint/ack fails, then the record is re-sent
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U->>P: record
P--xP: deduplication, process, update state, and checkpoint
U->>U: timeout
U->>P: record
P->>P: deduplication, process, update state, and checkpoint
P->>U: ack
U->>U: delete checkpoint
P->>D: record
```
Because the records will be deduplicated, crash during ack does not matter


Weak productions:

Most weak solution is just don't checkpoint at all. For retryability, The upstream may open many RPC if any downstream is slow
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U->>P: record
P->>P: deduplication, process, update state
P->>D: record
D->>D: deduplication, process, update state, send to downstream
D->>P: ack
P->>U: ack

```

send to downstream before checkpoint
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U->>P: record
P->>P: deduplication, process, update state
P->>D: record
P->>P: checkpoint
P->>U: ack
```
Smarter solution, only checkpoint if downstream acks is slow
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant D as downstream
U->>P: record
P->>P: deduplication, process, update state
P->>D: record
P->>P: timeout
P->>P: checkpoint
P->>U: ack
D->>D: deduplication, process, update state, send to downstream
D->>P: ack
```

# State Manipulation
If the process/state update/checkpoint/ack delays, the system start a new machine.
```mermaid
sequenceDiagram
participant U as upstream
participant P as processor
participant P' as new processor
participant D as downstream
participant B as Backing Store
U->>P: record
P--xP: deduplication, process, update state
P->>P': restart P
U->>U: timeout
U->>P': record
P'->>P': deduplication, process, update state
P'->>B: checkpoint
P->>B: checkpoint from zombie
P->>U: ack from zombie
U->>B: delete checkpoint
P'->>U: ack
P->>D: record
```
