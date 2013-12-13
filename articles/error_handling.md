---
title: "Using RabbitMQ From Clojure With Langohr: Error Handling and Recovery"
layout: article
---

## About this guide

Development of a robust application, be it message publisher or
message consumer, involves dealing with multiple kinds of failures:
protocol exceptions, network failures, broker failures and so
on. Correct error handling and recovery is not easy. This guide
explains how the amqp gem helps you in dealing with issues like

 * Initial connection failures
 * Network connection failures
 * AMQP 0.9.1 connection-level exceptions
 * AMQP 0.9.1 channel-level exceptions
 * Broker failure
 * TLS (SSL) related issues

as well as

 * How does the automatic recovery mode in Langohr 2.0+ work

This work is licensed under a <a rel="license" href="http://creativecommons.org/licenses/by/3.0/">Creative Commons Attribution 3.0 Unported License</a>
(including images and stylesheets). The source is available [on Github](https://github.com/clojurewerkz/langohr.docs).


## What version of Langohr does this guide cover?

This guide covers Langohr 2.0.x.


## Initial RabbitMQ Connection Failures

When applications connect to the broker, they need to handle
connection failures. Networks are not 100% reliable, even with modern
system configuration tools like Chef or Puppet misconfigurations
happen and the broker might also be down. Error detection should
happen as early as possible.

`langohr.core/connect` will raise `java.net.ConnectException` or `java.net.UnknownHostException` if a
connection fails. Code that catches it can write to a log about the
issue or use retry to execute the begin block one more time. Because
initial connection failures are due to misconfiguration or network
outage, reconnection to the same endpoint (hostname, port, vhost
combination) may result in the same issue over and over.

``` clojure

(require '[langohr.core :as rmq])

(rmq/connect {:host "127.0.0.1" :port 2887})
;; throws java.net.ConnectException due to incorrect port

(rmq/connect {:host "asd88asd.megacorp.com"})
;; throws java.net.UnknownHostException due to incorrect host
```



## Authentication Failures

Another reason why a connection may fail is authentication
failure. Handling authentication failure is very similar to handling
initial TCP connection failure:

``` clojure
(rmq/connect {:uri "amqp://sdfoiu:sd899937@hub.megacorp.local:5672/%2F"})
;; throws com.rabbitmq.client.PossibleAuthenticationFailureException
;; due to invalid credentials
```

In case you are wondering why the exception name has "possible" in it:
[AMQP 0.9.1 spec](http://www.rabbitmq.com/resources/specs/amqp0-9-1.pdf) requires broker
implementations to simply close TCP connection without sending any
more data when an exception (such as authentication failure) occurs
before AMQP connection is open. In practice, however, when broker
closes TCP connection between successful TCP connection and before
AMQP connection is open, it means that authentication has failed.

RabbitMQ 3.2 introduces [authentication failure
notifications](http://www.rabbitmq.com/auth-notification.html) which
Langohr supports. When connecting to RabbitMQ 3.2 or later, Langohr
will raise `com.rabbitmq.client.AuthenticationFailureException` when
it receives a proper authentication failure notification.


## Network Connection Failures

Detecting network connections is nearly useless if an application
cannot recover from them. Recovery is the hard part in "error handling
and recovery". Fortunately, the recovery process for many applications
follows one simple scheme that Langohr can perform automatically for
you.

### Automatic Recovery

When Langohr detects TCP connection failure, it will try to reconnect
every 5 seconds. Currently there is no limit on the number of reconnection
attempts.

To completely disable automatic connection recovery, pass
`:automatically-recover` as `false` `langohr.core/connect`.

### Topology Recovery

Many applications use the same recovery strategy that consists of the following steps:

 * Reconnect
 * Re-open channels
 * For each channel, re-declare exchanges (except for predefined ones)
 * For each channel, re-declare queues
 * For each queue, recover all bindings
 * For each queue, recover all consumers

Langohr provides a feature known as "automatic topology recovery" that performs these steps
after connection recovery, while taking care of some of the more tricky details
such as recovery of server-named queues with consumers.

To recover your topology, for every channel Langohr will

 * Re-declare queues originally declared on it
 * Re-declare exchanges originally declared on it
 * Re-establish bindings
 * Re-register consumers

**Server-named queues will be declared with new names** and their bindings and consumers
will be updated accordingly.

Langohr will not track inter-channel dependencies, e.g. when a
server-named queue was declared on channel 10 but used to consume
messages from on channel 20. This means that for automatic topology
recovery to work, all operations on a queue (declaration, binding,
consuming messages, etc) **must happen on the same channel**, otherwise
there is a possibility of the queue not being declared by the time another
channel recovers and tries to use it.

To disable topology recovery, pass `:automatically-recover-topology`
as `false`. Then Langohr will only recover connections and channels
(given that automatic recovery in general is not disabled).


## Channel-level Exceptions

Channel-level exceptions are more common than connection-level ones and often indicate
issues applications can recover from (such as consuming from or trying to delete
a queue that does not exist).

With Langohr, channel-level exceptions are raised as Java exceptions
(`IOException` or `ShutdownSignalException`) that provide access to
the underlying `channel.close` method information.

Shutdown exceptions can be inspected using functions in the `langohr.shutdown`
namespace:

``` clojure
(require '[langohr.queue    :as lhq])
(require '[langohr.shutdown :as lh])

(try
  ;; bind a non-existent queue
  (lhq/bind ch "ugggggh" "amq.fanout")
  (catch java.net.IOException ioe
    (lh/soft-error? ioe)
    ;= true, it's possible to recover from this exception
    (lh/initiated-by-broker? ioe)
    ;= true, RabbitMQ closed the channel
    (lh/initiated-by-application? ioe)
    ;= false
    (println (lh/reason-of ioe))
    (println (lh/channel-of ioe))
    (println (lh/connection-of ioe))))
```


### Common channel-level exceptions and what they mean

A few channel-level exceptions are common and deserve more attention.

#### 406 Precondition Failed

<dl>
  <dt>Description</dt>
  <dd>The client requested a method that was not allowed because some precondition failed.</dd>
  <dt>What might cause it</dt>
  <dd>
    <ul>
      <li>AMQP entity (a queue or exchange) was re-declared with attributes different from original declaration. Maybe two applications or pieces of code declare the same entity with different attributes. Note that different RabbitMQ client libraries historically use slightly different defaults for entities and this may cause attribute mismatches.</li>
      <li>`langohr.tx/commit` or `langohr.tx/#rollback` might be run on a channel that wasn't previously made transactional with `langohr.tx/select`</li>
    </ul>
  </dd>
  <dt>Example RabbitMQ error message</dt>
  <dd>
    <ul>
      <li>PRECONDITION_FAILED - parameters for queue 'langohr.examples.channel_exception' in vhost '/' not equivalent</li>
      <li>PRECONDITION_FAILED - channel is not transactional</li>
    </ul>
  </dd>
</dl>

#### 405 Resource Locked

<dl>
  <dt>Description</dt>
  <dd>
    The client attempted to work with a server entity to which it has no access because
    another client is working with it.
  </dd>
  <dt>What might cause it</dt>
  <dd>
    <ul>
      <li>
        Multiple applications (or different pieces of code/threads/processes/routines within a single application)
        might try to declare queues with the same name as exclusive.
      </li>
      <li>
        Multiple consumer across multiple or single app might be registered as exclusive for the same queue.
      </li>
    </ul>
  </dd>
  <dt>Example RabbitMQ error message</dt>
  <dd>RESOURCE_LOCKED - cannot obtain exclusive access to locked queue 'langohr.examples.queue' in vhost '/'</dd>
</dl>

#### 404 Not Found

<dl>
  <dt>Description</dt>
  <dd>The client attempted to use (publish to, delete, etc) an entity (exchange, queue) that does not exist.</dd>
  <dt>What might cause it</dt>
  <dd>Application miscalculates queue or exchange name or tries to use an entity that was deleted earlier</dd>
  <dt>Example RabbitMQ error message</dt>
  <dd>NOT_FOUND - no queue 'queue_that_should_not_exist0.6798199937619038' in vhost '/'</dd>
</dl>

#### 403 Access Refused

<dl>
  <dt>Description</dt>
  <dd>
    The client attempted to work with a server entity to which it has no access due
    to security settings.
  </dd>
  <dt>What might cause it</dt>
  <dd>
    Application tries to access a queue or exchange it has no permissions for
    (or right kind of permissions, for example, write permissions)
  </dd>
  <dt>Example RabbitMQ error message</dt>
  <dd>ACCESS_REFUSED - access to queue 'langohr.examples.channel_exception' in vhost 'langohr_testbed' refused for user 'langohr_reader'</dd>
</dl>



## What to Read Next

The documentation is organized as [a number of guides](/articles/guides.html), covering various topics.

We recommend that you read the following guides first, if possible, in this order:

 * [Using TLS (SSL) Connections](/articles/tls.html)
 * [RabbitMQ Extensions to AMQP 0.9.1](/articles/extensions.html)
