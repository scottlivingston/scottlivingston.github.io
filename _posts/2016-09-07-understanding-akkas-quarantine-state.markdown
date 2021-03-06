---
title: Understanding Akka's Quarantine State
date: 2016-09-07 19:00:00 -0700
layout: post
description: Akka's quarantine system has some interesting caveats that are not immediately obvious. I found out the hard way.
---

At Credit Karma, my team maintains a service powered by a multi node [Akka](http://akka.io/) cluster. Akka is a framework for building distributed, concurrent, and message-driven systems in Scala. It helps us easily scale our service, and gives us some resilience when problems inevitably happen in production. In this post, I'll cover two problems - auto down, and quarantine - and share what we learned when we had problems.

One day, all large number of requests going to our service started failing. Looking at our monitoring tool, we saw that the one of the nodes that listens for REST calls was no longer making calls to other nodes in the cluster. So we started digging though our logs to try and figure out what happened. This is what we found:

```nohighlight
WARN Cluster Node [akka.tcp://***] - Marking node(s) as UNREACHABLE [Member(address = akka.tcp://***, status = Up)]
INFO Cluster Node [akka.tcp://***] - Leader is auto-downing unreachable node [akka.tcp://***]
INFO Cluster Node [akka.tcp://***] - Marking unreachable node [akka.tcp://***] as [Down]
INFO Cluster Node [akka.tcp://***] - Leader is removing unreachable node [akka.tcp://***]
WARN Tried to associate with unreachable remote address [akka.tcp://***]. Address is now gated for↩
  5000 ms, all messages to this address will be delivered to dead letters. Reason: [The remote system↩
  has quarantined this system. No further associations to the remote system are possible until this↩
  system is restarted.]
...
```

As a new user of Akka, I had never seen something like this happen before let alone know what it means when a node gets quarantined. But the error message was clear enough; we needed to restart that node's actor system to fix it. After that, everything was back to normal and the task switched from firefighting to investigation, starting with what the heck quarantine actually means.

## Quarantine
An actor system can be placed in a quarantine state when a remote actor system determines that further communication to that system would be harmful. It does this by dropping all incoming messages from, and outgoing messages to the quarantined actor system. The [Akka documentation](http://doc.akka.io/docs/akka/current/scala/remoting.html#Lifecycle_and_Failure_Recovery_Model) lists three reasons why a node would become quarantined:

>- System message delivery failure
>- Remote DeathWatch trigger
>- Cluster MemberRemoved event

This is a pretty short list without a ton of information, so let's dig into each one of them.

### System message delivery failure
This one is less specific than the other two so we will start with it. The code that powers it can be found [here](https://github.com/akka/akka/blob/v2.4.4/akka-remote/src/main/scala/akka/remote/Remoting.scala#L475):
```scala
case HopelessAssociation(localAddress, remoteAddress, Some(uid), reason) =>
  log.error(reason, "Association to [{}] with UID [{}] irrecoverably failed. Quarantining address.",
    remoteAddress, uid)
  settings.QuarantineDuration match {
    case d: FiniteDuration =>
      endpoints.markAsQuarantined(remoteAddress, uid, Deadline.now + d)
      eventPublisher.notifyListeners(QuarantinedEvent(remoteAddress, uid))
    case _ => // disabled
  }
  Stop
```
HopelessAssociation is an exception that is thrown when Akka gives up trying to associate a node with a cluster. There are a few cases where this can happen, but it's all pretty deep in the internals of Akka. Suffice to say, the application logic of your code will most likely not be the cause of this quarantine.

### Remote DeathWatch trigger
In addition to the [normal actor death watch mechanism](http://doc.akka.io/docs/akka/current/scala/actors.html#deathwatch-scala), the remote death watch system augments itself with a periodic heartbeat to detect when a remote actor has failed due to a network problem or a remote JVM crash.
The code that powers this transition is shown [here](https://github.com/akka/akka/blob/v2.4.4/akka-remote/src/main/scala/akka/remote/RemoteWatcher.scala#L160):
```scala
def reapUnreachable(): Unit =
  watchingNodes foreach { a =>
    if (!unreachable(a) && !failureDetector.isAvailable(a)) {
      log.warning("Detected unreachable: [{}]", a)
      quarantine(a, addressUids.get(a))
      publishAddressTerminated(a)
      unreachable += a
    }
  }
```
`reapUnreachable()` is called periodically by a schedule call higher up in the same file. If an actor is watching another actor on a remote actor system and the failure detector has deemed it unavailable, then the watched actor system becomes quarantined. This is an important distinction because if a remote actor system becomes temporarily unreachable, but nothing is watching it, it **will not** become quarantined.

### Cluster MemberRemoved event
This one is a little tricky and, in the end, what caused the problem at the beginning of the post. The code can be found [here](https://github.com/akka/akka/blob/v2.4.4/akka-cluster/src/main/scala/akka/cluster/ClusterRemoteWatcher.scala#L93):
```scala
def memberRemoved(m: Member, previousStatus: MemberStatus): Unit =
  if (m.address != selfAddress) {
    clusterNodes -= m.address
    if (previousStatus == MemberStatus.Down) {
      quarantine(m.address, Some(m.uniqueAddress.uid))
    }
    publishAddressTerminated(m.address)
  }
```
Nodes can be removed from a cluster from one of two states: `Exiting` and `Down`. The main difference between these states is `Exiting` is the happy path, where as `Down` is the sad path. If a node is removed from the cluster while in the `Exiting` state, then the node will not be quarantined. If the node is removed while in the `Down` state, then it will be quarantined.

So what caused the errors we saw? Before we get there we have to talk about one more thing.

## Automatic Downing
When a member node in an Akka cluster becomes unreachable, the state of the cluster essentially becomes locked unless the unreachable node becomes reachable again, or it is marked as `Down`. There are two ways to do this, [manually or automatically](http://doc.akka.io/docs/akka/current/scala/cluster-usage.html#Automatic_vs__Manual_Downing). When using the automatic system, a timeout is set that will automatically mark the unreachable node as `Down` if it stays unreachable for longer than the timeout.

While we were developing the service, we had automatic downing enabled and set trigger after only a few seconds. Experienced Akka developers may look at this and think, "The docs say not to use auto-down in production!" and they would be correct. The simple fact is, we had forgotten it was there and it made it into production. Because of this, small network problems were causing nodes to be marked as unreachable, immediately transitioning them to `Down`, and then promptly removing them. As we learned earlier, if a node is `Down` when it's removed from the cluster, it will get quarantined.

Sometimes this would occur on one of the dedicated REST nodes and, even though they were quarantined from the rest of the cluster, the client would continue to make calls to it which caused errors. It has been some time now since we removed the auto-down flag from our config file and we have never had a node get quarantined since.

## What we learned

### Things will fail, so try to make it graceful.
We built our Akka cluster to stay available in all kinds of situations, but you cannot protect against everything. The best thing to do is make sure you can fail gracefully. In our case, the client we built to call our service is smart enough to retry in the event of a failure on a different node.

### Sometimes the best documentation is the code itself.
Sometimes, when faced with a problem, you have to go to the source to see what's actually happening. It can be rather daunting when you first start, but a good integrated development environment can help immensely. I recommend starting from a single point and branching out. In this case, I knew that a `QuarantineEvent` is passed onto the event bus when a quarantine happens. So I started by looking at all the places where that event is used, and then found only the places that put that event on the event bus.
