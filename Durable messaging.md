# Network is reliable

is the first of so-called fallacies of distributed computing. Attributed to L. Peter Deutsch (co-authored by Bill Joy, Dave Lyon and James Gosling), the list constains statements that, on the surface, seem to be commonsensical and true, but turn out to be fals upon detailed investigation. In other words, they are assumptions many inexperience engineers make when designing their software that come to bith them when the said software is deployed to production. If you want to learn more about the fallacies of distributed computing, check out this other mini-book (TODO: Link to Dr Harvey).

Network is reliable, isn't it? Many smart folks have dedicated their careers to making it so. Take the Ethernet protocol as an example. In its original form when many computers were connected via a single copper wire there was a potential for multiple machines to attempt to talk past each other. Due to the limited nature of the speed of light two computers could start sending their messages over the network at the very same time, unaware of one another. As a result, the interference of the two signals made the transmission illegible. The Ethernet protocol dealt with this by detecting the interfence and scheduling re-transmission of the message with pseudo-random delay, minimizing the changes of subsequent conflict.

At a higher level the TCP protocol guarantees that underlying IP packets are re-transmitted transparently to the TCP protocol application, effectively hiding any intermitent network issues. So, is the network reliable? Unfortunately it is not. Event TCP with all its smart features can't handle more persistent problems. When the other party seems to not be responding in any way it has no choice but give up and terminate the connection. When that happens the application cannot know for sure if the process on the other side recieved all the data or even if it received any data at all -- the only safe assumption is that it did not.

## At-most-once

This behavior that we have described is referred to as _at-most-once_ communication. The TCP protocol guarantees that between the initation of a TCP connection and its normal or abnormal termination the data requested to be sent has been delivered to the receving process _at-most-once_. It also ensures that the data has been delivered in the same order as sent. 

This basic guarantee turns out to be suprisingly useful. Higher level protocols, such as HTTP, have been built on top of it, delagating all their reliability to the underlying TCP. Consequently, when sending a HTTP request one might be able to obtain the response code indicating success or failure of the _processing_, but one must always be prepared to deal with _connection terminated_ error, in which case the caller cannot know anything about the result.

As proven by the tremendous success of the HTTP protocol, this is perfectly fine. The reason is that the application can always retry the whole operation until it finally succeeds.

## Boilerplate

In reality retrying is not as easy as it seems. We, humans, can deal with this task relatively well. When the website I use to order coffee beans fails with _connection terminated_ error I quickly hit `F5` to retry the form submission. If it does not work again, I _remember_ to try in a couple of seconds and if that does not resolve the problem, I came back after a few hours to try again. At that point my shopping cart might already be gone but I have built in _memory_ that keeps a mental copy of my order. My addition to coffee on the other hand _drives me_ to continue trying time and again until I succeed.

Software programs need to be designed to have the _memory_ and the _drive_ to retry communication. It is not an easy task, by no means, as it involves talking to disk drives and dealing with network connections. This is the reason why there is a specialized clas of software components that do just that: take the burden of retrying off from the application. In other words they provide _at-least-once_ communication semantics.

## At-least-once

When a message is handed over to such a specialized component, that component stores it durably (equivalent of my own memory) and then tries to forward it to the destination. If it fails, it places the message on the retry loop (equivalent to my apetite for coffee). Eventually after a successful result, the message is removed from the storage to conserve space. This idea is called _durable asynchronoous messaging_.

Software that realizes this pattern was among the first to be available e.g. IBM MQ. This proves how useful the idea of _at-least-once_ communication has been since the very beginning of business software. One key difference between the early implementations of durable messaging solutions and the modern ones is in the way they handle transactions.

## Exactly-once

In the past these technologies, in one way or another, allowed sharing atomic transaction context with a database (e.g. IBM MQ and IBM DB2 or more recently Microsoft SQL Server and MSMQ). This meant that the application can, in a single atomic transaction, consume a message, modify data, and send out follow-up messages. 

Combined with broker's advanced internal message deduplication and exclusive consumption we get the end-to-end guarantee of exactly-once communication, that is the code processing messages can always assume that it will event get a single copy of each message and, once that copy is acknowledged as processed, it will never appear again.

This _Atomic Message Handler_ pattern proved to be tremendously useful and gained popularity over the years. Unfortunately, most modern cloud-native message brokers do not offer similar transaction capabilities. In one of the next chapters we will dig deeper into reasons why it is the case.

## Reliability

Since a message broker is a separate piece of software that runs outside of application's address space, communication with it is a subject of failure. If so, the application must be able to retry until it succeeds but this seems to defeat the whole purpose of having an _at-least-once_ message broker. All we did is we have just moved the problem to a different place, didn't we?

Not exactly. First of all, we traded the problem of retrying talking to another application for a problem of retrying communication with a message broker. These pieces of infrastructure tend to be quite reliable if well-maintained so the actual likelihood of failure is much smaller (though of course not zero) and the expected periods of unavailability -- much shorted. This means that it _might be acceptable_ to move the burden of dealing with an unlikely failure to the user. We will explore that option and its (much better) alternatives later.

Second, the job of retrying to send a message can be, in many cases, entirely off-loaded to a durable message broker. How so? If the message processing code is executed as a result of receiving a message from a broker (as opposed to e.g. receiving a HTTP request), the message broker guarantees to not drop that message until it receives acknowledgement from the application code. If that code fails to send a follow-up message, an exception is raised that prevents the acknowledgement to be delivered. As a result, the message broker schedules the message to be retried again. As you can see, **no application code is required to ensure the reliability of the communication when a durable message broker is used both as a source and as a sink of messages**. We sometimes say that these _happy_ message handlers are in the inside of a system. The other ones for which the source of the signal is non-durable are said to _sit on the boundary_. We are going to explore the subject of that boundary later.

There is one caveat, however. Can you spot it? While the broker can guarantee reliability of the communication inside of the system, it can't guarantee the consistency. Not unless the application code that is executed when handling a message is **idempotent**. Than means that it is safe for the broker to retry delivering a message as many times as needed -- the visible side effects of all these retries will be exactly the same as the result of one successful attempt.

## End-to-end argument

In their landmark paper, [End-to-end arguments in system design](https://web.mit.edu/saltzer/www/publications/endtoend/endtoend.pdf), J.H. Saltzer et al. point out that the correctness of a distributed system cannot be guaranteed by the lower layers of the communication subsystem, only the ends of the information exchange. Let's come back to the TCP example from the beginning of a chapter and suppose application `A` sends to application `B` a command to increment a counter. It opens a `TCP` socket and transmitts a couple of bytes that represent the request. On the other side application `B` reads from its socket and executes the operation. Finally it sends back a confirmation message. Only upon receiving that message from the TCP stream can the application `A` be sure that the request has been executed. No socket-level API can be of any help.

Does that mean that these durable message brokers we have spent so much time discussing are useless? The short answer is: no! The long answer consists of two parts. The first part is related to reliability of transferring a single message. The end-to-end guarantee of consistency is delegated to SKDs that facilitate communication with the broker. These SDKs run in the applications address space and make sure that the received message in the applicatoin's memory is byte-by-byte equal to the one that has been sent.

The second part of the answer is more tricky. To understand it we need to visualize how the TODO
