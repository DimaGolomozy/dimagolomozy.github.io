---
layout: post
title:  "How to create Akka Prioritized Mailbox"
date: 2019-01-19 01:00:00
description : "A simple and nice way to create akka prioritized mailbox"
image: assets/images/akka-prioritized-mailbox/prioritized-mailbox.jpg
comments: true
tags: [Java, Akka, priority, prioritized, mailbox]
---
![Prioritized Mailbox]({{ site.url }}/assets/images/akka-prioritized-mailbox/prioritized-mailbox.jpg)

In my previous job (at [TIM Media][tim-site]) we used [Akka][akka] for our RTB system. Lots of actors, lots of messages and a lot of a mess (that's what you get in RTB systems).
When the system got bigger and bigger with new features and new message types, we came across a situation that we need to prioritize the messages.
Well at that point creating a `switch case` or `if-else` like below, will not work nicely. Plus, we had the prioritized mailbox in a shared common package, so if we wanted to add a new type of message, we would need to update all the projects that use this common package - hell no!

{% highlight java %}
public class MyPrioritizedMailbox extends UnboundedStablePriorityMailbox {
  public MyPrioritizedMailbox(ActorSystem.Settings settings, Config config) {
    super(new PriorityGenerator() {
      @Override
      public int gen(Object message) {
        if (message instanceof Message1)
          return 0;
        else if (message instanceof Message2)
          return 2;
        else if (message instanceof Message3)
          return 3;
        else
          return 1;
      }
    });
  }
}
{% endhighlight %}

## The Solution

In our solution, we thought it will be easier just to add an `interface` to messages that we want to prioritize and all the rest will be with `MEDIUM` priority.
We created an interface called `PrioritizedMessage` and 4 more inner interfaces for the priority types.

{% highlight java %}
public interface PrioritizedMessage {
    int URGENT = 100, HIGH = 200, MEDIUM = 300, LOW = 400;
    int getPriority();

    interface Urgent extends PrioritizedMessage {
        default int getPriority() { return URGENT; }
    }

    interface High extends PrioritizedMessage {
        default int getPriority() { return HIGH; }
    }

    interface Medium extends PrioritizedMessage {
        default int getPriority() { return MEDIUM; }
    }

    interface Low extends PrioritizedMessage {
        default int getPriority() { return LOW; }
    }
}
{% endhighlight %}
The `100` fix diff between each priority number, is to allow flexibility between them. If we'll want to create a new message type that is not `High` and not `Urgent`, but it's something in between,
we will just implement the `getPriority` method and return a number between 100 to 200, that simple.

Java 8 has introduced the concept of default methods which allow the interfaces to have methods with implementation without affecting the classes that implement the interface,
a simple `implement PrioritizedMessage.Urgent` will work without having the compiler to scream on you to implement the `getPriority` method.

That's leave us with a simple and clean `UnboundedStablePriorityMailbox` that will use the `PrioritizedMessage` interface to determine the priority.
{% highlight java %}
public class PrioritizedMailbox extends UnboundedStablePriorityMailbox {
    public PrioritizedMailbox(ActorSystem.Settings settings, Config config) {
        super(new PrioritizedGenerator(
                config.hasPath("priority.poison-pill") ? config.getInt("priority.poison-pill") : PrioritizedMessage.HIGH,
                config.hasPath("priority.kill-pill") ? config.getInt("priority.kill-pill") : PrioritizedMessage.HIGH
        ));
    }

    private static class PrioritizedGenerator extends PriorityGenerator {
        private final int poisonPillPriority;
        private final int killPriority;

        private PrioritizedGenerator(int poisonPillPriority, int killPriority) {
            this.poisonPillPriority = poisonPillPriority;
            this.killPriority = killPriority;
        }

        @Override
        public int gen(Object message) {
            if (message instanceof PrioritizedMessage) {
                return ((PrioritizedMessage) message).getPriority();
            } else if (message instanceof PoisonPill) {
                return poisonPillPriority;
            } else if (message instanceof Kill) {
                return killPriority;
            } else
                return PrioritizedMessage.MEDIUM;
        }
    }
}
{% endhighlight %}

#### PoisonPill / Kill Messages

Once Prioritized Mailbox is configured, the `PoisonPill` and `Kill` message types are also been prioritized.
In the example above, the default priority of those types is `PrioritizedMessage.HIGH`, and by passing in the configuration the priority, one can configure a different priority for those types.

## Another Way - Annotation
Instead of using an interface `PrioritizedMessage` with 4 inner interfaces, a single [annotation][java-annotation] with a single `priority` method can be used.

{% highlight java %}
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.TYPE)
public @interface Prioritized {
    int URGENT = 100, HIGH = 200, MEDIUM = 300, LOW = 400;

    int priority() default MEDIUM;
}
{% endhighlight %}

and with a little change in the `PrioritizedGenerator` class:
{% highlight java %}
@Override
public int gen(Object message) {
    if (message.getClass().isAnnotationPresent(Prioritized.class)) {
        return message.getClass().getAnnotation(Prioritized.class).priority();
    } else if (message instanceof PoisonPill) {
        return poisonPillPriority;
    } else if (message instanceof Kill) {
        return killPriority;
    } else
        return PrioritizedMessage.MEDIUM;
}
{% endhighlight %}
_hell, you can use even both methods... crazy!!!_


[tim-site]: http://thetimmedia.com
[akka]: https://akka.io
[java-annotation]: https://en.wikipedia.org/wiki/Java_annotation