# Durable messaging

Now that we have introduced the main hero of our book we should take a step back and fill in some datails about the context, that is _durable messaging_.

## Network is reliable

is the first of so-called fallacies of distributed computing. Attributed to L. Peter Deutsch (co-authored by Bill Joy, Dave Lyon and James Gosling), the list constains statements that, on the surface, seem to be commonsensical and true, but turn out to be fals upon detailed investigation. In other words, they are assumptions many inexperience engineers make when designing their software that come to bith them when the said software is deployed to production.

We are going to come back to this list many times in subsequent chapters. If you want to learn more, check out this other mini-book (TODO: Link to Dr Harvey).

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

When a message is handed over to such a specialized component, that component stores it durably (equivalent of my own memory) and then tries to forward it to the destination. If it fails, it places the message on the retry loop. Eventually after a successful result, the message is removed from the storage to conserve space. This idea is called _durable asynchronoous messaging_. 