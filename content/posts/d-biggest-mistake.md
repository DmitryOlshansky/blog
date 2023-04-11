---
title: "D Biggest Mistake"
date: 2023-04-11T15:12:52+03:00
---

# DLang’s biggest mistake - transitive qualifiers

Make no mistake - thread-local by default is great! But shared should have been _storage_ declaration qualifier with static/plain globals meaning thread-local storage. 

With that clarified let’s get to the reasons for me insisting that transitive qualifiers are the single biggest reason for most of the D programming language fundamental problems. 

Transitive qualifiers taken together with “wildcard” inout qualifier are easily the single largest breaking feature D1->D2  with _huge_ implementation costs that are still being paid as it provides a steady source of compiler bugs, albeit mostly sorted out by now. The ripple effects of transitivity are not only apparent in user code bases, it has the same if (not worse) proliferation of changes in the compiler.

Being touted as a requirement to support functional programing and guarantee purity of functions. Fine, let’s deconstruct this. First, I’ve programmed Scala, often used as Haskel “substituite” on the JVM, with many simillar idioms readily translatable.  See Cats, Scalaz and related projects for puristic FP on Scala. There, as in other languages, that I however do not know sufficiently well, the mutability problem is solved very pragmatically:
- val is default variable type designator means shallow const (non-assignable)
- var is the mutable/reassignable conterpart, heavily shunned by the tooling ecosystem (specially highlighted in editors, etc.) 
- collections provide immutability by interface - there is simply no write/remove operations in List’s API
- Mutable collections provide superset of immutable API
- Structures by construction consist of val or var, but case classes - the daily driver shorthand notation for dumb data classes consist of vals only

If we throw in explicit shared storage qualifier into this mix, the final nail in the coffin of concurrency bugs - globals would be mostly taken care of. In Scala it would be awkward to introduce shared mostly because it needs to seamlessly blend with Java/JVM and introducing poorly translatable storage qualifiers  / behavior will introduce impendance mismatch. D is not so confined.


Not eliminated mind you, but with easy and correct defaults in place and the rest of the language and tooling pushing in the right direction it gets 90% of the job done without hurting any of the niche cases. Such as, you know, lock-free programming which is quite painful to write with transitive shared in D - there is no escaping the casting to thread-local mutable problem. The problem is easy to state - many lock-free algorithms hinge on the idea of taking ownership of something via CAS operation (or commiting your _local_ changes). By the very definition it transitions shared to non-shared. However nobody notified our compiler friend of such arrangements and I see no solid plans (nor any message on if they are feasible) in that direction as well.

But what about locking, I hear you say. Same deal basically - you grab a lock on shared object and... nothing happens it’s still shared object with CAS as the only safe operation. For all of its complexity the D type system is not smart enough to learn the ownership transfer and insert “smart casts” (a-la Kotlin) or something like that.

With these 2 niches just visited there is little evidence that transitive shared is able to prevent enough of harmful designs to prove its worth yet hinders the most straight-forward patterns.

Message passing by itself (e.g. std.concurrency library) doesn’t fix the issue with transitivity - you still either cast stuff on the receiving end (after all - the sender should be done with, right? Ownership - again not tracked, so can’t be used).

Now with transitive shared being largely disappointing let’s move on to designs that are enabled by transitive immutability. Maybe that’s the treasure trove we have suffered so much to get to.

Alas, no, it’s largely the same - the solution in search of a problem. And it creates a ton of its own:
1. A copy of immutable(T) is passable as (implicitly) head-mutable, but only if’s a pointer or slice. User-defined types are out of luck and this makes writing generic const-qualifier correct code a pleasure to suffer through.
2. Generic code is expected to use const(T) as super type of T and immutable(T). Sadly that still has the problem #1.
3. Templates without extreme precautions and Unqual!T all around will get instantiated up to 3x(!) times. Might be the reason compile times are not fastest in the market (but among fastest still, a lost opportunity right there)
4.  inout. That’s it. I’ve been with DLang for 2012-2018 and it was a great wild ride but I. Never. Fully. Understood. How inout is supposed to work beyond the simplest (the intended?) cases. Keep in mind it also mixes nicely with problem #3. 
5. About the only types properly and easily working with immutable are pointers and array thanks to #1. And truth be told a wrapper type that disallows modification easily solves this use-case, covering 80+% of immutable uses in the wild.
6. Making your types immutable/const friendly is royal pain the ass and it won’t be long before your users will try const blah = YourType(...); and _will_ complain laudly that you *do not support constness*! It’s not pretty to implement elaborate UDT that works with transitive const and importantly *it puts the pressure in the wrong place*. Users _do_ need to be pressured to use const but not by making library author’s life much harder.

Unlike transitive const, shallow const (~ Java’s final) is trivially applicable to any user-defined type with next to no extra work on the library author’s side. Moreover now author can clearly provide types that are  immutable via interfaces (collections! strings!). 

Lastly. There is one(!) great interaction with transitivity, which is (true) pure functions.  However D prides itself on pragmatism and I bet you anything that “non-true” (logical immutability) pure functions in Scala/Haskel/any other functional language are just fine and much easier to write.

What is the way out?

I do not know nor have any power to change the direction of D language. It is a promising modern systems programming language but without dealing with wart of this scale it would be hard to grow beyond its current (way higher than zero, btw, impressively so) presence. 

My suggestion (putting a D Language BDFL hat on)  is ..

Drop it! And burn with fire.

The steps are:
1. Immutable checks are shallow but. compiler warns about breaking transitive const/immutable. With preview option you get no such warnings.
2. Shared is storage qualifier and preview option warns about inappropriate usage are issued, silenced with a flag. Such usage is ignored for a couple of releases.
3. 2&4 at their own cadence are graduated to defaults with explicit options to revert.
4. No more options we are done with “turtles all the way down”, and hopefully for good.