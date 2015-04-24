---
layout: post
title: "Translate and override val/var/def"
date: 2015-02-17 16:24:40 +0800
comments: true
categories: [scala]
---

**Val/var/def** are Scala's language constructs that manifest immutibility amongst other features. Looking behind how **scalac** translates them helps us to understand more about related semantics like instance initialization, override, etc..


#Translate val/var/def to java
There are a few simple rules to translate val/var/def from scala to java.

For traits

* private variables are replaced by ones with names "mangled"; protected varibles are spared
* getters and setters are generated
* "initializers" are generated

For classes

* getters and setters are generated

Note that terms like "translate", "mangle" and "initializer" are used here for the ease of understanding. I don't think they are common jargon in Scala community.

Now let's look at an example:
<!-- more -->

```scala
// scala
trait A {
  val a = 10
  private val b	= 20
  var c = 30
  private var d = 40
  def e
}
```

An interface and an abstract class is generated for trait A
```java
// java
interface A {
  // val a
  // an initializer in the form of A$_setter$<name>_$eq
  void A$_setter_$a_$eq(int paramInt);  // initializer for val
  int a();  // getter

  // private val b
  // a mangled name A$$b is generated to replace 'b'
  void A$_setter_$A$$b_$eq(int paramInt);
  int A$$b();

  // var c
  // no need for an initaliser for var
  // instead, var needs a setter
  int c();
  void c_$eq(int paramInt);  // setter

  // private var d
  // mangled name A$$d for getter/setter
  int A$$d();
  void A$$d_$eq(int paramInt);  // setter

  // def e
  int e();
}

abstract class A$class
{
  // we'll see how static $init$ will be used in concrete class's constructor
  static void $init$(A $this) {
    $this.A$_setter_$a_$eq(10);
    $this.A$_setter_$A$$b_$eq(20);
    $this.c_$eq(30);
    $this.A$$d_$eq(40);
  }
}
```

We can not "new A" as a trait is like an interface. A concrete class extends A must be there to instantiate instances. Say we have a concrete class A1:
```scala
class A1 extends A {
  def e = 50
  private val f = 60
}
```

It's then transformed to java code:
```java
class A1 implements A {
  // all fields in A, mangled or not
  private final int a;
  private final int A$$b;
  private int c;
  private int A$$d;

  // all implementations of getters/setters/initializers
  int a() { return this.a; }
  void A$_setter_$a_$eq(int x$1) { this.a = x$1; }

  int A$$b() { return this.A$$b; }
  void A$_setter_$A$$b_$eq(int x$1) { this.A$$b = x$1; }

  int c() { return this.c; }
  void c_$eq(int x$1) { this.c = x$1; }

  int A$$d() { return this.A$$d; }
  void A$$d_$eq(int x$1) { this.A$$d = x$1; }

  int e() { return 50; }

  // A1's newly added members, and their getters/setters.
  // No mangled names
  // No initializers
  // access qualifier is private if it's defined so
  private final int f;
  int f() { return this.f; }

  A1() {
    // static method "initialize" trait A's members
    A.class.$init$(this);

    // initialization of A1's added members
    this.f = 60;
    // ...
  }
}
```

If **val/var** are newly declared in a class rather than a trait, the corresponding getters and setters. Nevertheless, no name mangling and no initializers.


-
- - -


#Override between val/var/def
The other aspect of this topic is override rules. The rationale behind the rules normally can be obtained from applying **Substitution Principle**. It's the error messages that sometimes may cause confusion. We'll reason about the legality by **Substitution Principle** or by looking at the translated java code.


###var/def can't override val
```scala
class A1 {
  val a = 11
}
class A2 extends A1 {
  override var a = 12
}

> error: variable a needs to be a stable, immutable value
```
It's because clients of A1 would always expect `a` to be immutable. That's the value would always be the same for every reads. Nevertheless, A2 would break the contract.


###val/def can't override var
Firstly, **val** can't override **var**:
```scala
class A1 {
  var b = 21
  def inc_b = { b += 1 }
}
class A2 extends A1 {
  override val b = 22
}

> error: value b cannot override a mutable variable
```
Clients of A1 would always expect 'a' to be mutable and it's always legal to modify it.

Next, the reason why **def** can't override *var* is similar. Plus, the clients of A2 can't both read and write to **def** as it can to **var**. Under the hood, **def** is only one method, whilst **var** gets translated to a getter and a setter.


###val can override def with no parameters
Think everyone's agreed.


###var can't override def

```scala
class A1 {
  def e = 50
}
class A2 extends A1 {
  override var e = 51
}
> error: method e_= overrides nothing
```
Recall that a pair of getter/setter are generated for **var**, `e()` and  `e_$eq(Int)`, respectively. The later doesn't override anything hence the error.

While **var** can't override **def**, it's alright that **var** *implements* **abstract def**.
```scala
trait C {
  def e
}
class C1 extends C {
  var e = 51
}
```


-
- - -

#val/var/def with different access modifiers

We haven't disscussed about **val/var/def** with other access modifiers like protect, package private, etc.. However, one can decompile the translated java by using tools like javap or [JD-GUI](http://jd.benow.ca/) and examine the result himself.


-
- - -

