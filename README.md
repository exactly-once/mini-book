# TOC

1. Outbox intro - one with polling for side effects application. Explain that outbox is bound to generate duplicates so the next thing is to look at de-duplication but before we do that, let's first come back to basics
1. Durable messaging
1. Distributed transaction - is it an alternative to outbox?
1. Deduplication - idempotence-based, state-based, id-based
1. Message-generation issues when using pure deduplication -- need for deterministic message generation. Bsaed on [this post](https://exactly-once.github.io/posts/consistent-messaging/)
1. Outbox combined with de-duplication
1. Side-effects application re-visited -- message retries as alternative to polling
1. Transactional session -- control message as alternative to polling
1. Outbox in HTTP-based APIs -- storing response as side effect
1. Optimizing deduplication data storage -- separate deduplication store
1. Optimizing outbox for Azure Service Bus
1. Zero-data outbox with token-based deduplication
