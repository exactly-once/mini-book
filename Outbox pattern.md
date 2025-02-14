# The Outbox pattern

The simplest software consists of a user interface and a databse. A company ran by a friend of ours has one of these. It is built using Microsoft Access or similar technology. It allows creating invoices, tracking orders, monitoring the warehouse state -- everything a small shop needs. There is just one tiny detail that distinguishes this piece of software from the one that powers Amazon. The former isolated. It does not talk to any other piece of software. When an invoice needs to be sent, he manually copies the data from the system to his favourite email application. The latter talks to vast number of other systems. Moreover, it has an internal structure that consists of components that communicate with one another. This approach our friend uses solves one of the biggest challenges modern business software has to face:

> How do I modify my own state and communicate with external party in one atomic step

One way of solving that problem is by using a pair of humuans and well-known communication medium, such as email. When, for some reason, using this approach is not feasible, another good way is by employing the Outbox pattern. Before we jump into the solution mode, let us explore the problem space a bit and, among other concerns, discuss why the **one atomic step** mentioned above is so important.

## Context and problem

Whenever software systems communicate, there is a need to send a signal and modify internal state. That state modification, at minimum, is marking that the communication has occurred, but in a real-world system can be much more than that.

This introduces a problem of _partial failures_. We have a situation when a piece of code needs to interact with two external resources: a database and another system. Each of these interaction can have one of six outcomes. The call request might have succeeded, failed or been rejected and the caller might received the notification of the outcome or a general network error. The difference between the failed and rejected outcome is that a failure means the operation can be retried and the caller may expect success. If an operation has been rejected, however, it means that the subsequent calls won't ever succeed.

Partial failures are something we want to avoid at all costs in business applications. Dealing with them moves the focus away from the business problem at hand and forces the implementation code to be jumping back-and-forth between two different abstraction levels: the domain and the plumbing.

This problem has been solved in the past using distributed transaction technologies. These clever pieces of software have been designed to remove the burden of dealing with partial failures in the calling code. As long as the resources involved adhered to a defined protocol (e.g. 2PC), the distributed transaction _coordinator_ was able to guarantee that a transaction that involves multiple resources either eventually entirely succeeded (all resources reported success) or is rolled back (none of the resources have been modified).

Unfortunately, due to significant technical issues, distributed transaction technologies did not survive until the present day. We are going to come back to them later, explaining in detail the reasons for their initial succeess and subsequent decline but now we should just point out that the problem we are dealing here can be made a bit simpler then outlined above. If we **assume that the communication between systems is asynchronous and one-way**, the external system can be represented as a slightly simpler resource with only four possible outcomes of each operation as, by definition, there is no option for rejection or an asynchronous one-way call.

NOTE: To be preceise, what the assumption about asynchronous one-way communication __really__ means is that the **application, rather than infrastructure, is going to handle rejections**.

To summarize, the problem is coordinating operations on two resources, one of which can either accept or reject the outcome while the other always accepts. The resources are in separate address spaces which introduces possibility of each operation to end with a network error that hides the actual outcome.

## Solution

The solution, that is the Outbox pattern, is to append the representation of the operation to be executed on the always-accepting resource to the operation executed against the potentialy-rejecting resource. This means that the latter has to be able to store and retrieve arbitrary information. For all practical purposes it means that the latter needs to be a **database**.

The other resource can be any communication technology that has one-way semantics. A message broker, like RabbitMQ, is one of the most popular choices but a HTTP API endpoint that does not return any response other than `201 Accepted` works perfectly fine, too.

The key requirement of the Outbox pattern is that the representation of the operations to be executed on the one-way resource is stored atomically with the database update. This guarantees that there are no partial failures e.g. data changed but operations not stored as the whole point of the Outbox pattern is to remove the need to deal with this kind of failures.

Next, after the one-way operations are stored, a separate process makes sure they are executed against their target resources e.g. by pushing a message to a queue or making a HTTP request. Because of the ever present possiblity of network errors the execution of the operations needs to be repeated until a successful outcome is returned.

## Consequences

The major consequence of using this pattern is the fact that operations executed against the database resource under the control of the Outbox pattern need to be *idempotent*, that is have the same side effects, regardless if their are executed one or multiple times.

To understand this we need to pull the camera back and imagine how long-running business processes are composed from smaller buildig blocks. Each of these blocks is one single Outbox transaction. When one transaction completes, it stores a one-way signal that is transmitted to another component that reacts to it by execting an Outbox transaction of its own. Given that the first component's outbox re-tries sending the signal until a success outcome is obtained, the signal may be duplicated. For any single signal sent by the first component, the second may be forced to execute multiple copies of its Outbox transaction. To ensure the data stays consistent, this transactions needs to be *idemponent*.

In subsequent chapters we are going to discuss various techniques of ensuring idempotency that can be used *in conjunction* with the Outbox pattern in order to guarantee end-to-end consistency across the whole system.

One notable exception to the rule above, that we are also going to deal separately, is the behavior at the boundary of the system where the code interacts with a human being. The difference stems from the fact that most human beings do not have reliable transmission retry mechanism built-in (they have other useful features, though).

## Variants

Based on the above, one aspect in which real-world implementations of the Outbox pattern differ is the way they handle (or not) the idempotency of the database operations:
 - no support for automatic idempotency
 - automatic idempotency via keeping track of processed signals
 - automatic idempotency via keeping track of signals to be processed

Implementations of the Outbox pattern differ in one more important way, the mechanism that ensures stored operations are executed. There are four generally-accepted approaches:
 - driven by a time-based polling mechanism
 - driven by a time-based non-polling mechanism 
 - driven by a message retry mechanism
 - driven by the client

We are going to deal with all this variety in subsequent chapters so stay tuned!


