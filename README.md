# TOC

1. Network is reliable - what happens when a message is send using a TCP socket
1. End-to-end argument
1. Durble messaging ⌛
1. Partial failures -- how to receive message, update data and send a follow-up message
1. Duplication - processing side, broker side, sender-side ⌛
1. Determinisic message generation based on [this post](https://exactly-once.github.io/posts/consistent-messaging/)
1. Outbox intro as a solution to deterministic message problem ✔️ (do not introduce variants here, just plain vanilla polling version)
1. State-based deterministic messaging based on [this post](https://exactly-once.github.io/posts/state-based-consistent-messaging/) as an alternative to outbox
1. Distributed transaction - another alternative to outbox. Based on [this post](https://exactly-once.github.io/posts/notes-on-2pc/) but extended with the intro to 2PC
1. Deduplication - idempotence-based, state-based, id-based
1. Outbox combined with de-duplication
1. Side-effects application re-visited -- message retries as alternative to polling
1. Transactional session -- control message as alternative to polling
1. Outbox in HTTP-based APIs -- storing response as side effect
1. Optimizing deduplication data storage -- separate deduplication store
1. Optimizing outbox for Azure Service Bus
1. Zero-data outbox with token-based deduplication
