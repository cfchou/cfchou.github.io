---
layout: post
title: "Pattern Matching for Type Erasure"
date: 2015-02-24 12:55:39 +0800
comments: true
categories: [scala, type erasure, generics, akka, kafka]
---


#Producer as an Actor
In one of my toy projects, I use an **Akka** `Actor` to encapsulate a **Kafka** `Producer`. The benefit is twofold. First, it's recommanded that [a producer should be shared among all threads for best performance](http://kafka.apache.org/082/javadoc/index.html?org/apache/kafka/clients/producer/KafkaProducer.html). Maintaining a seemingly long-lived actor/producer can be easily achieved thanks to **Akka**'s `SupervisorStrategy`. Second, resources can be well managed by the hooks inserted in the life-cycle of the actor. However, indirectly asking producer to send messages of a parameterized type would sacrifice type safety due to **type erasure** as described bellow.

<!-- more -->

#Problems caused by Type Erasure
**Type erasure** removes some type information of parameterized types. It is used to fill the gap between java generics and the legacy code written prior generics. Scala, while is subject to the fact that it is implemented in java, introduces a few mechanisms to get around type erasure.

To see one of the limitations caused by **type erasure**, let's look at this piece of code:
```scala
class ProducerActor[K, V] extends Actor {
  val producer = new Producer[K, V]
  def receive: Receive = {
    case m: KeyedMesssage[K, V] =>
      producer.send(m)
    case _ => log.debug("Unknown is discarded")
  }
}

object TestApp extends App {
  val actor = new ProducerActor[Int, String]
  actor ! new KeyedMessage[Int, String](1, "legal")
  actor ! new KeyedMessage[String, String]("illegal", "but compiled!")
}

```
The compiler don't complain about the illegal message, since **Actor**'s bang `!` function accepts the type of `Any`. But can we rely on pattern matching in `receive` to spot misuse and safely discard them during run-time? Unfortunately, we can't, at least not in this way.

At the end of the day, JVM only knows `ProducerActor[_, _]` `KeyedMessage[_, _]` at run-time. Type information of the type parameters are not carried over from compile-time. Matching whatever instantiation of `KeyedMessage[X, Y]` to `KeyedMessage[_, _]` is unduly legal(though an unchecked warning should be issued, more on that later). In `TestApp`, the illegal `KeyedMessage[String, String]` will match the first clause. A run-time exception will be thrown because `producer.send` is expecting `KeyedMessage[Int, String]`.


#Reification using TypeTag[T]

The solution, as [suggested by Roland Kuhn](https://groups.google.com/forum/#!topic/akka-user/7gd2Tfwax5Q), is to use `TypeTag[T]`. Basically, to make JVM aware of what `K, V` really are, a technique called **type reification** is needed. It's a behaviour that enough information is retained so that JVM knows what type parameters are during run-time. Scala's `TypeTag[T]` provides such functionality:

```scala
class ProducerActor[K, V](implicit tk: TypeTag[K], tv: TypeTag[V])
  extends Actor {
  val producer = new Producer[K, V]
  def receive: Receive = {
    case m: Msg[K, V]
      if m.tk.tpe <:< tk.tpe && m.tv.tpe <:< tv.tpe =>
      producer.send(m)
    case _ => log.debug("Unknown is discarded")
  }
}

class Msg[K, V](override val key: K, override val value: V)
  (implicit val tk: TypeTag[K], val tv: TypeTag[V])
  extends KeyedMessage[K, V](key, value)
```
`TypeTag[T]::tpe` is reflective representation of type T. An instance of `A <:< B` witnesses that **A is a subtype of B**.

**Context bound** can be used to reduce a little bit of clumsiness of the `ProducerActor` interface:
```scala
class ProducerActor[K:TypeTag, V:TypeTag] extends Actor {
  val producer = new Producer[K, V]
  def receive: Receive = {
    case m: Msg[K, V]
      if m.tk.tpe <:< implicitly[TypeTag[K]].tpe
        && m.tv.tpe <:< implicitly[TypeTag[V]].tpe =>
      producer.send(m)
    case _ => log.debug("Unknown is discarded")
  }
}
```

#Props Factory Pattern

If we try to create a `ProducerActor` in this way:
```scala
val actor = context.actorOf(Props(classOf[ProducerActor[Int, String]]))
```
We'll get during run-time an **IllegalArgumentException**:

> java.lang.IllegalArgumentException: no matching constructor found on class ProducerActor for arguments []

The reason causing this problem is subtle. I guess that's because the needed **implicits TypeTag[K] and TypeTag[V]** are not there by the time the Props is used for creating actors.

We can define a **[Props factory](http://doc.akka.io/docs/akka/2.3.9/scala/actors.html#Recommended_Practices)** to avoid this problem(the major benefit of using a **Props factory** is described in the document and is beyond the scope of this post):
```scala
object ProducerActor {
  def props[K:TypeTag, V:TypeTag]: Props = Props(new ProducerActor[K, V])
}
// ...
val actor = context.actorOf(ProducerActor.props[Int, String])
```
By doing so, we're assured that the compiler would generate the needed implicits for `Props` object right at the line of `actorOf`.



#Reify List[KeyedMessage[K, V]]

Notice that **Kafka** `Producer`'s `send` actually accepts **vararg**:
```scala
def send(messages: KeyedMessage[K, V]*)
```
Sending one message is just sending a one-element list of messages. `receive` therefore only needs to handle the generalized case:
```scala
def receive: Receive = {
  case ms: List[KeyedMesssage[K, V]] =>
    producer.send(ms)
  case _ => log.debug("Unknown is discarded")
```
Likewise, JVM wouldn't know anything about the type parameters due to **type erasure**. The pattern is `List[_]` at run-time. Consequently, any `List[T]` would match. Unsurprisingly, we can enforce type safety by `TypeTag[T]`. Before we proceed, there are a few points worth noting.

1. This piece of code would compile with a warning saying **"non-variable type argument KeyedMessage[K, V] in type pattern List[KeyedMessage[K, V]] is unchecked since it is eliminated by erasure"**. Such warning rises because according to [Java Generics FAQ](http://www.angelikalanger.com/GenericsFAQ/FAQSections/TechnicalDetails.html#FAQ001), **"unchecked" warnings are reported when the compiler finds a cast whose target type is either a parameterized type or a type parameter.** However, why didn't we see any warning previously when we delt with `case m: KeyedMessage[K, V]`? Well, it is probably a bug and is addressed by [SI-9188](https://issues.scala-lang.org/browse/SI-9188).

2. We wouldn't discuss it at length but actually [there are different types of TypeTag](http://docs.scala-lang.org/overviews/reflection/typetags-manifests.html). `TypeTag[T]` is deliberately chosen since it is the strongest in the sense that it would retained type information for all nested type parameters.

By and large, the solution is:
```scala
case class ListMsg[U](ms: List[U], tag: TypeTag[U])

class ProducerActor[K:TypeTag, V:TypeTag] extends Actor {
  val producer = new Producer[K, V]
  def receive: Receive = {
    case ListMsg(ms, tag) if tag.tpe <:< typeOf[KeyedMessage[K, V]] =>
      val tmp = ms.asInstanceOf[List[KeyedMessage[K, V]]]
      producer.send(tmp: _*)
    case _ => log.debug("Unknown is discarded")
  }
}
```
Note that we have to cast to `List[KeyedMessage[K, V]]` to pass the static type check depite the fact that during run-time, JVM actually sees `ms.asInstanceOf[List[_]]`. Nevertheless, by filtering out unwanted types in the case clause, the cast would always happily succeed.


That's pretty much it. We have seen how **type erasure** steps in the way when pattern matching generic types. A solution of using `TypeTag[T]` is demonstrated. This technique of course can be used elsewhere to improve type safety.


-





