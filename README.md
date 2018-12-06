# Class 12

## Parametric Polymorphism and Generic Programming

We earlier discussed *parametric polymorphism* as a way to make a
language more expressive, while still maintaining static type safety.
A function or a class can be written such that it can handle values
identically without depending on their type. 

Generic programming is a style of programming in which algorithms are
written in terms of types to-be-specified-later. These type
dependencies are expressed using type parameters that are then
instantiated when needed for specific types. In other words, generics
allow you to abstract over types.

Why do we want such a language feature?

* We have a compiler for a reason. It finds bugs for us at
  compile time before we even run the program. Run-time bugs are
  generally much more difficult to find and resolve.

* Generic programming gives us another way of expressing some
  constraints to the compiler and therefore to prevent certain types
  of bugs.

* Generic programming also allows us to write less code, which is
  always a good thing.

Benefits:

* Stronger type checks at compile time: the compiler applies strong
  type checking to generic code and issues errors if the code violates
  type safety.

* Elimination of dynamic casts: values can be inserted and extracted
  from generic data structures without dynamic type checks.

* Enabling programmers to implement generic algorithms: programmers
  can implement generic algorithms that work on collections of
  different types, can be customized, and are type safe.

## Scala Generics

Scala Generics provide an example of a generic programming language
construct.  Generics enable types to be parameters when defining
classes and methods. Like formal parameters used in method
declarations, type parameters provide a way for you to re-use the same
Scala code with different types. Common examples are found in the
Scala API.

For example, similar to OCaml's type `'a list` that represents lists
of elements of some type parameter `'a`, Scala provides a type
`List[T]` for representing lists of elements of some type parameter
`T`. The following declares a list of `Int` values `l`:

```scala
  val l: List[Int] = List(1, 2, 3, 4)
```


### A Generic Functional Queue Implementation

We study Scala generics by implementing a generic functional queue
data structure similar to the
one
[we implemented in OCaml](https://github.com/nyu-pl-fa18/class10/#ocamls-module-system). The
following discussion builds on Ch. 19 of OSV, so I encourage you to
read that chapter for additional details. Though, we will cover some
orthogonal aspects of generics that are not covered in the book.

Our queue data structure supports three operations:

* `enqueue` to insert a new element at the back of the queue

* `dequeue` to remove the first element from the front of the queue

* and `isEmpty` to check whether the queue is empty.

Remember that a functional queue data structure differs from a queue
that is implemented in an imperative style in that the enqueue and
dequeue operations do not mutate the state of the queue object
in-place, but instead return new queue objects that capture the
updated state of the queue after the operation has been performed. The
state of the original queue object remains unchanged. Such data
structures are therefore also called *persistent*. This design choice
suggests the following signature for our `Queue` class:

```scala
abstract class AbstractQueue[T] {
  def enqueue(x: T): AbstractQueue[T]

  def dequeue: (T, AbstractQueue[T])

  def isEmpty: Boolean
}
```

This class is *generic* in the type parameter `T`. This type parameter
abstracts from the type of the elements stored in the queue. Note that
`dequeue` returns a pair consisting of the dequeued element and a
queue object that represents the new state of the queue holding the
remaining elements of the queue.

The keyword `abstract` indicates that the class `Queue` cannot be
instantiated directly (because we did not actually specify how the
three methods are implemented).

A straightforward implementation of this data structure is to simply
use a `List[T]` to represent the elements of the queue in dequeue
order. Here is an implementation of this idea:

```scala
class Queue[T] private (
    private val queue: List[T]
  ) {
  
  def enqueue(x: T): Queue[T] = new Queue[T](queue :+ x)

  def dequeue: (T, Queue[T]) = {
    require(!isEmpty, "Queue.dequeue on empty queue")
    val x :: queue1 = queue
    (x, new Queue(queue1))
  }

  def isEmpty: Boolean = queue.isEmpty

  override def toString: String = {
    s"Queue${queue.toString.drop(4)}"
  }
}
```

The field `queue` that we use to hold the values stored in the queue
is declared private so that the internal representation of a `Queue`
instance is hidden from its clients. For the same reason, the primary
constructor of class `Queue` is declared private. This is achieved by
adding the keyword `private` in front of the parameter list of the
class. As a consequence, only instances of class `Queue` and its
companion object can construct `Queue` instances directly.

In the implementation of `enqueue`, the expression `queue :+ x`
creates a new list by appending `x` at the end of the list `queue`,
leaving `queue` itself unchanged. The implementation of `dequeue`
first ensures that the queue is non-empty and then decomposes `queue`
into its head `x` and tail `queue1` using pattern matching and then
returns `x` and the new `Queue` instance holding `queue1`. The
implementation of `isEmpty` simply delegates the emptiness check to
the list `queue`.

The companion object then provides two factory methods for
constructing queues to hide the queue's internal representation:

```scala
object Queue {
  def empty[T]: Queue[T] = new Queue(Nil)

  def apply[T](xs: T*): Queue[T] = new Queue(xs.toList)
}
```

The type `T*` of the parameter `xs` in method `apply` indicates that
`apply` can be called with a variable length list of parameters of
type `T`. Here is how a client can use this data structure:

```scala
scala> val q = Queue(1, 2, 3, 4)
q: Queue[Int] = Queue(1,2,3,4)

scala> val (d, q1) = q.dequeue
d: Int = 1, q1: Queue[Int] = Queue(2,3,4)

scala> q.enqueue(5)
res1: Queue[Int] = Queue(1,2,3,4,5)
```

Observe that the data structure is indeed persistent: the call
`q.dequeue` does not modify the state of `q` so that the
subsequent call `q.enqueue(5)` adds the new element at the end of the
original queue.

For a generic class such as `Queue[T]`, `Queue` itself is not a
type. Only the instances `Queue[Int]`, `Queue[String]`, ... of
`Queue[T]` obtained by instantiating the type parameter `T` with
another type are themselves types. Instead, `Queue` is called a *type
constructor*. This is different to generics in Java where a generic
class `C[T]` gives rise to a type `C` that is referred to as a *raw
type*.

### Type Erasure and Specialization

How are generics implemented by the Scala compiler? The Scala compiler
uses a technique called *type erasure*. That is, the type parameter
annotations in generic classes and methods are only needed at compile
time when the program is type checked. Once the compiler has
determined that all generic classes are implemented and used
correctly, these annotations are erased from the program. Each value
of some generic type `T` is simply treated as a value of type `Any`,
i.e., as belonging to some arbitrary object type. Essentially, the
type parameters go away and the compiler generates exactly the same
byte code that we would get if we wrote the generic class using type
`Any` instead of using a generic type parameter.

The advantage of this approach is that the compiler can generate a
single byte code version of a generic class that can be shared by all
its instantiations. In particular, if we call the `enqueue` method on
a `Queue[String]` and a `Queue[Int]` in our program, the exact same
byte code is executed by these two method calls at run time. 

One disadvantage of type erasure is that generic classes cannot create
instances of their generic type parameters. E.g., in our queue
implementation we would not be allowed to write something like `new
T()`. For this to work, the compiler would need to dynamically
dispatch the constructor call to create the `T` instance. However,
constructors are not dynamically dispatched.

Another disadvantage of type erasure is that the generated byte code
uses the type `Any` to represent the values of the instantiated types
uniformly for all instantiations. This is OK for actual object types
since they are uniformly represented as pointers to heap allocated
objects. However, for types that represent primitive values such as
`Int` and `Double` which are allocated on the stack, the compiler
needs to perform an implicit type conversion whenever a value of the
type parameter is passed to a method of the generic class or retrieved
from it. For instance, when we call `q.enqueue(x)` on a queue `q` of
type `Queue[Int]` and an argument `x` of type `Int`, then the generic
byte code generated for `enqueue` expects a 64-bit pointer to a heap
allocated object as argument. However, `x` stores an `Int` value,
which only takes 32 bits.

To perform the type conversion, the compiler uses a technique called
*auto boxing*. Before passing `x` to `enqueue`, the compiler generates
auxiliary glue code that creates a new heap allocated object and
stores the value of `x` in a field of that object. Instead of passing
`x` directly to `enqueue`, it instead passes the pointer to the
wrapper object. Similarly, when we call `q.dequeue`, there will be
auxiliary glue code that retrieves the wrapper object from the queue
and *unboxes* it by extracting the contained `Int` value. While these
conversions are opaque to the programmer, they incur a constant time
and space overhead per conversion that you should be aware of.

If you write performance critical applications, you may want to avoid
the additional cost of auto-boxing that is incurred by using type
erasure on generic classes instantiated by value types. For instance,
suppose we want to use our `Queue` data structure in performance
critical code that instantiates `Queue` for the value types `Int` and
`Double`. To avoid the performance overhead, we can tell the compiler
to specialize the generated byte code for these two types by prefixing
the type parameter in the generic class with the `@specialized`
annotation as follows:

```scala
class Queue[@specialized(Int, Double) T] private (
    private val queue: List[T]
  ) {
  ...
}
```

Instead of generating a single byte code implementation of `Queue`
that is shared by all types `T`, the compiler will now generate three
versions: one for type `Int`, one for type `Double`, and one generic
version for all remaining types. Since we have specialized byte code
versions of `Queue` for types `Int` and `Double`, the values of these
types can be passed to their `Queue` versions directly without
auto-boxing, thus eliminating the performance overhead. All remaining
value types (`Boolean`, `Char`, `Byte`, ...) will still fall back to
the generic byte code version and require auto-boxing as before.

The disadvantage of specialization is that compilation time will
increase. Moreover, if we are instantiating the generic class for
different specialized types within the same program, then the JVM will
have to load the byte code for each of those versions into memory at
run-time. This incurs a constant space overhead. However, this
overhead is usually negligible compared to the space overhead caused
by auto-boxing. Therefore, most of the generic classes provided by the
Scala API are specialized for all primitive value types.

Type erasure and specialization are common techniques used by
compilers to implement parametric polymorphism in programming
languages. For instance, both Java and C# compilers use type erasure
to implement generics in the respective languages. On the other hand,
functors in languages in the ML family and C++ templates, which is the
programming feature that provides parametric polymorphism in C++, are
implemented using specialization. That is, in OCaml and C++ each
instantiation of a functor or template will yield a separate native
code version that is specialized for the particular instantiation. The
Scala compiler takes a middle way between the two approaches.

### Variance

Next, we will study how generics relate to subtyping. For instance,
suppose we have the types `Duck`, `Sparrow`, and `Bird` where `Duck`
and `Sparrow` are subtypes of `Bird`. If we have a queue instance of
type `Queue[Duck]`, can we also use it in situations where we need an
instance of type `Queue[Bird]`? That is, is `Queue[Duck]` a subtype of
`Queue[Bird]`? If that's the case, then for example the following code
would be OK:

```scala
val ducks: Queue[Duck] = Queue(new Duck)
val birds: Queue[Bird] = ducks.enqueue(new Sparrow)
```

More generally, suppose we have a generic class `C[T]` that
parameterizes over a type parameter `T`. Suppose further that we have
two concrete types `A` and `B` such that `A` is a subtype of `B`, `A
<: B`. The question is: what does this tell us about the subtype
relationship between `C[A]` and `C[B]`. We refer to this relationship
as the *variance* of the type constructor `C` with respect to its type
parameter `T`. We distinguish three cases:

* `C[T]` is *covariant* in `T`: if `A <: B`, then `C[A] <: C[B]`. That
  is, the subtype relationship between the argument types is preserved
  by the instantiation of `C`.

* `C[T]` is *contravariant* in `T`: if `A <: B`, then `C[B] <:
  C[A]`. That is, the subtype relationship between the argument types
  is inverted by the instantiation of `C`.

* `C[T]` is *invariant* in `T`: neither `C[A] <: C[B]` nor `C[B] <:
  C[A]` holds in general if `A <: B`. That is, there is no subtype
  relationship between the instantiations regardless of how `A` and
  `B` are related.

Whether `C` is covariant, contravariant, or invariant in its type
parameter `T` depends on the implementation details of `C`. Covariant
and contravariant generics provide additional flexibility to clients
compared to generics that are invariant. For instance, if `C[T]` is
covariant in `T`, clients can use a `C[A]` whenever a `C[B]` is
expected. On the other hand, if `C[T]` is invariant in `T`, then this
is not possible.

Since variance depends on implementation details, it is the
programmer's responsibility to decide whether this property should be
exposed to the clients of a generic class. The implementer of the
generic class might decide against exposing the fact that the
implementation is co- or contravariant to retain more flexibility for
future changes to the implementation. For instance, a future
optimization of the implementation might turn a covariant
implementation into an invariant one, breaking client code that relied
on the variance of the implementation. Functional implementations tend
to be more resilient in terms of modifications that affect variance
whereas implementations that expose state to the client are typically
invariant.

By default the compiler treats all type parameters of a generic class
as invariant. If the programmer decides that a particular type
parameter should be treated covariantly, they can express this with a
variance annotation by preceding the binding occurrence of the type
parameter in the head of the class declaration with the symbol
`+`. Likewise, if the type parameter is to be treated contravariantly,
then it should be annotated with the symbol `-`. The compiler will 
statically check whether these variance annotations are indeed
compatible with the implementation of the class.

Let us study these issues in more depth using our example of the
functional queue `Queue[T]`. First, is the implementation of
`Queue[T]` covariant, contravariant, or invariant in `T`? It is easy
to see that it can't be contravariant. For otherwise, clients would be
allowed to do the following:

```scala
val q: Queue[Any] = Queue[Any](1) // OK because Int <: Any
val qs: Queue[String] = q // OK because String <: Any and Queue is
                          // supposedly contravariant, i.e. Queue[Any] <: Queue[String]

val (s: String, _) = q1.dequeue // OK because q1 is of type Queue[String]
s.charAt(0) // bzzzz! would try to call method charAt on an Int
```

On the other hand, we can argue formally that `Queue` is covariant in
`T`. Suppose we have `A <: B` and a `Queue[A]` instance `q`. To see
that `q` can be used in a context that views `q` as a `Queue[B]`, we
have to analyze the ways in which that context can interact with `q`.
There are only two possible interactions that we need to consider:

1. the context calls `q.dequeue`: this call will return a pair `(x,
   q1)` where `x` is of type `A` and `q1` is the remainder of the
   queue, which is again of type `Queue[A]`. Since the caller views
   `q` as a `Queue[B]` it expects a pair of type `(B,
   Queue[B])`. Since `A <: B` we know that `x` is also a `B`
   instance. Moreover, by our assumption that `Queue` is covariant, we
   can again conclude that `q1` is also of type `Queue[B]` (formally,
   this argument uses a proof technique called coinduction). So the
   return value of the call is indeed of type `(B, Queue[B])` as
   expected by the context.

1. the context calls `q.enqueue(y)` for some `y` of type `B`: first
   note that our implementation of `enqueue` does not modify the state
   of `q`, so `q` itself is unaffected by the fact that we pass a `B`
   instead of an `A`. Second, `enqueue` returns a new queue that
   stores `y` together with the elements stored in `q`. Since `q` is
   of type `Queue[A]`, all elements stored in `q.queue` are of type
   `A`. Further, since `A <: B`, all those elements are also `B`
   instances. Thus, all elements in `queue` of the new queue are `B`
   instances and hence the new queue is of type `Queue[B]`. Therefore,
   this case is also OK.

As we can see, it is a non-trivial task to formally prove the
correctness of the variance annotations in the interface of a generic
class to ensure that the specified interface between the class' client
and implementation code is consistent. Fortunately, the type checker
in the Scala compiler automates this task for us.

Thus, let us see what the type checker thinks about our claim that
`Queue[T]` is covariant in `T`. We can annotate the binding occurrence
of `T` in `Queue` with a covariance annotation as follows:

```scala
class Queue[+T] private (
    private val queue: List[T]
  ) {
  ...
}
```

Running this through the type checker yields the following type error:

```scala
Error:(8, 15) covariant type T occurs in contravariant position in type T of value x
def enqueue(x: T) = new Queue[T](queue :+ x)
            ^
```

It seems that we missed something in our proof (attempt)! Before we
analyze this problem further, let us first look at a simpler example
to better understand the reasoning that is done by the type checker to
prove the correctness of variance annotations.

### Covariant and Contravariant Occurrences of Type Parameters

Consider the following simple generic class `Cell[T]` that describes
objects providing read and write access to a mutable field `content`
of type `T`:

```scala
class Cell[T](private[this] var content: T) {
  def read: T = content
  
  def write(x: T): Unit = {
    content = x
 }
}
```

Note that the access modifier `private[this]` specifies that `Cell`
instances only have access to their own `content` field. That is
neither the companion object of `Cell` nor other cell instances can
access the `content` field of `this`.

We distinguish between covariant and contravariant type annotations
within the implementation of a generic class (relative to the client
interface). A type annotation is covariant if it describes values that
may flow from an instance of the class to its client (e.g. the return
type of a public method). An example of a covariant occurrence of `T`
in `Cell` is the use of `T` in the return type of the method
`Cell.read`. If on the other hand, the type annotation describes
values that flow from the client to an instance of the class (e.g., a
parameter type of a public method), then this annotation is
contravariant. An example of a contravariant occurrence of `T` in
`Cell` is the use of `T` in the type of the parameter `x` of method
`Cell.write`.

You may also think of covariant occurrences as describing values that
are read by the client from the instance and contravariant occurrences
as describing values that are written by the client into the instance.

Intuitively, covariant type annotations express guarantees that the
instances of the generic class promise their clients. From the
clients' perspective, a covariant type annotation `T` can therefore be
viewed as expressing lower bounds on types with respect to subtyping:
the clients may safely assume that the values returned by method calls
are an instance of any supertype of `T`. From the perspective of the
implementation of the class, they express upper bounds with respect to
subtyping: the method body must return an instance of some subtype of
`T`. E.g. consider the types `Mallard <: Duck <: Bird`. If the return
type of a method is `Duck`, the implementation may return a `Mallard`
as a special kind of `Duck`, but it can't return an arbitrary
`Bird`. Likewise, the client may treat the return value as an
arbitrary `Bird`, but it can't assume that it is a `Mallard` if the
method only promises to return a `Duck`.

On the other hand, contravariant type annotations express assumptions
that the instances of the generic class make about values provided by
the clients. Hence, they express lower bounds with respect to
subtyping from the perspective of the implementation of the class.
From the clients' perspective, they can be viewed as expressing upper
bounds: the clients are required to provide values of the specified
parameter types or any of their subtypes in method calls. E.g. if a
method expects a parameter of type `Duck`, the client can provide a
`Mallard` but not an arbitrary `Bird`. On the other hand, the method
body may treat the parameter as an arbitrary `Bird` if the type of the
parameter is `Duck` but it can't assume that the argument is a
`Mallard`.

A covariant type parameter of a generic class is only allowed to occur
covariantly in type annotations within the class and dually for
contravariant type parameters. For instance, if we tried to make 
the type parameter `T` of `Cell` covariant, the
compiler would produce the following type error:

```scala
error: (4, 12) covariant type T occurs in contravariant position in type T of value x
def write(x: T): Unit = {
          ^
```

To see why using covariant type parameters in contravariant positions
is problematic, consider the following client code. This code would be
well typed if we allowed `Cell` to be covariant in `T`:

```scala
val c = new Cell("Hello")
val c1: Cell[Any] = c // OK because String <: Any and Cell is covariant
c1.write(1) // OK because c1 is of type Cell[Any]
c.read.charAt(0) // OK because c: Cell[String] and hence c.read: String
```

However, if we executed this code, then the third line would modify
the contents of cell `c` since `c` and `c1` alias the same
instance. Thus, we would be calling `charAt` on an `Any` instance
in the last line, a method that does not exist in `Any`.

If we remove the method `write` from `Cell` so that values of type `T`
can only be read but not written, then `Cell` becomes covariant in
`T`. Thus, the following code compiles:

```scala
class Cell[+T](private[this] var content: T) {
  def read: T = content
}
```

Likewise, if we keep `write` but remove `read` from `Cell` so that
values of type `T` are only written but not read, then `Cell` becomes
contravariant in `T`:

```scala
class Cell[-T](private[this] var content: T) {
  def write(x: T): Unit = {
    content = x
  }
}
```

If we keep both `read` and `write`, then `Cell` is invariant in `T`.

Note that covariance and contravariance annotations can also be used
in combination within the same generic class (i.e. if a class has
multiple type parameters, some of them may be covariant and others
contravariant). 

If a type parameter `T` of a generic class `C1` occurs below some
other generic type constructor `C2[T]` within the implementation of
`C1`, then whether that occurrence is considered covariant or
contravariant depends on both the variance of the occurrence of
`C2[T]` in `C1`, and the variance of the type parameter of `C2`. If
`C2` is covariant in its type parameter, then the occurrence of `T`
below `C2[T]` has the same variance as the occurrence of `C2[T]` in
`C1`. On the other hand, if `C2` is contravariant in its type
parameter, then the occurrence of `T` below `C2[T]` has the opposite
variance as `C2[T]` in `C1`. If `C2` is invariant, then the occurrence
of `T` is also invariant, in which case `T` must be invariant in `C1`
as well.

For instance, consider again our implementation of `Queue`:

```scala
class Queue[T] private (
    private val queue: List[T]
  ) {
  ...
}
```

Here, `C1` is `Queue` and `C2[T]` is `List[T]`. The type `List[T]`
occurs in the type annotations of the private `val` field of
`Queue[T]`. Since `val` fields can only be read but not written, these
occurrences are covariant. Moreover, since `List` is also covariant in
its type parameter
(see
[here](https://www.scala-lang.org/api/2.12.4/scala/collection/immutable/List.html)),
the occurrence of `T` in `List[T]` is also covariant.

### Lower Bounds on Type Parameters

Coming back to our implementation of `Queue[T]`, we now understand why
the compiler does not allow us to make the type parameter `T`
covariant: the `enqueue` method takes an argument of type `T` and
hence `T` occurs in a contravariant position.

However, we may wonder why the variance mismatch is problematic in
this case. After all, unlike the case of `Cell[T].write` where the
class relies on the fact that the client passes a `T` object to
`write`, when a client calls `Queue[T].enqueue`, the queue instance
does not really care whether the object being passed to `enqueue`
belongs to a supertype of `T` instead of a subtype of `T`, as we
argued in our earlier proof attempt.

The flaw in our reasoning is that we have ignored the fact that
classes are open to extension by inheritance. That is, even though
`Queue[T].enqueue` can be called safely with arguments that belong to
a supertype of `T`, this may not be true for subclasses of `Queue`
that override `enqueue`. As an example, suppose we added a method to
the companion object of `Queue` that created a subclass of
`Queue[String]` overriding `enqueue` with a new implementation that
depends on `enqueue` being provided with a `String` value:

```scala
object Queue {
  ...
  
  class SillyQueue extends Queue[String](Nil) {
    override def enqueue(x: Int): Queue[String] = {
      println(x.length) // relies on x being of type String
      super.enqueue(x) // delegate actual work to superclass
    } 
  }
  
  def silly: Queue[String] = new SillyQueue
}
```

Now, if `Queue` was allowed to be covariant in `T`, the following
client code would compile:

```scala
val q: Queue[Any] = Queue.silly // OK because Queue.silly returns a
                                   // Queue[String] and Queue is covariant
q.enqueue(1) // OK because q: Queue[Any], 1: Int, and Int <: Any
```

At run-time, the call `q.enqueue(1)` would go to the
overridden version of `enqueue` in the class `SillyQueue`. This
version of `enqueue` would then try to access field `length` of an
`Int` value. However, `Int` values do not have such a field.

In order to make `Queue` covariant in `T`, we need to modify the type
signature of `Queue[T].enqueue`. As we observed, the implementation of
`enqueue` does not actually rely on the fact that the provided
argument is of type `T`. So we simply need to make this explicit in
the type signature of `enqueue`. The overriding implementations of
`enqueue` in the subclasses of `Queue[T]` then can no longer rely on
their parameter being of type `T` either. We achieve this by making
`enqueue` itself generic in its type parameter:

```scala
class Queue[+T] ( ... ) {
 ...
 
 def enqueue[U](x: U): Queue[?] = new Queue[?](queue :+ x)
}
```

We have replaced the type `T` of the parameter `x` in enqueue with a
new type parameter `U` that is instantiated for each call to
`enqueue`. Hence, `U` is a type parameter of the method `enqueue`.

Now `T` no longer occurs contravariantly. However, one question
remains: what should be the element type `?` of the queue returned by
`enqueue`? It can't be `T` because `x` is of type `U`, which may not
be a subtype of `T`. The only other sensible option is `U`. However,
for this to work, the client needs to guarantee that `U` is a
supertype of `T` for otherwise the list `queue :+ x`
will not be of type `List[U]`. We can express this constraint by
adding a lower bound on the type parameter `U` of `enqueue`:

```scala
class Queue[+T] ( ... ) {
 ...
 
 def enqueue[U >: T](x: U): Queue[U] = new Queue[U](queue :+ x)
}
```

The notation `U >: T` in the declaration of the type parameter `U`
expresses that `enqueue` can only be called with values of types `U`
that are supertypes of the element type `T` of the queue
instance. Note that the types occurring in lower bound constraints are
treated covariantly (why?).

With these modifications, we obtain a covariant implementation of
`Queue`. In particular, the following client code will now compile and
work as expected:

```scala
class A {
  override def toString: String = "A"
}

val q = new Queue(new A) // creates Queue[A]
val q1 = q.enqueue("Hello") // OK because there exists a common supertype of A and String: AnyRef
                            // That is, the compiler will infer type Queue[AnyRef] for q1
val (o, _) = q1.dequeue // o will now point to the A instance created earlier
println(o.toString) // OK because o: AnyRef and AnyRef has a toString method
```

### Upper Bounds on Type Parameters

Similar to lower bounds, we can also add constraints to type
parameters that express upper bounds with respect to subtyping. This
is useful in cases where a generic class has certain minimal
requirements on the capabilities of the types used for instantiating
its type parameters. As an example consider a simple class hierarchy
consisting of an abstract class `Animal` and concrete subclasses
`Cat`, `Dog`, and `Lion`:

```scala
abstract class Animal {
 def makeNoise: Unit
}

class Cat extends Animal {
  override def makeNoise = println("Meow!")
}

class Dog extends Animal {
  override def makeNoise = println("Woof!")
}

class Lion extends Animal {
  override def makeNoise = println("Roar!")
}
```

Each subclass overrides the abstract method `makeNoise` of `Animal`
with an implementation specific to that animal.

Suppose now that we want to implement a container for animals such that
the container itself provides a method `makeNoise`, which simply calls
`makeNoise` on all the contained animals. The following code realizes
such an implementation:

```scala
class Animals[A <: Animal](private val animals: List[A]) {
  def makeNoise: Unit = for (a <- animals) a.makeNoise
}
```

The notation `A <: Animal` expresses an upper bound on the type
parameter `A` of the generic container class `Animals`. This upper
bounds guarantees that the call `a.makeNoise` in the implementation of
`Animals[A].makeNoise` is always safe because we know that `a` must be
an instance of some class that is a subtype of `Animal`.

The following client code then works as expected (the type annotations
are only added for documentation - they would be automatically
inferred by the compiler):

```scala
val dogs: Animals[Dog] = new Animals(List(new Dog))
  
dogs.makeNoise // prints Woof!
  
val cats: Animals[Cat] = new Animals(List(new Cat))
  
cats.makeNoise // prints Meow!
  
val zoo: Animals[Animal] = new Animals(List(new Dog, new Cat, new Lion))
  
zoo.makeNoise // prints Woof! Meow! Roar!
```
