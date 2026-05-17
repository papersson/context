Source: https://www.youtube.com/watch?v=SxdOUGdseq4

# Simple Made Easy

*A talk transcript, cleaned for readability*

---

## Introduction

All right — who's ready for some more category theory? You're all in the wrong room.

This talk, I hope, seems deceptively obvious. One of the things that's great about this conference is that it's a pretty cutting-edge crowd. A lot of you are adopting new technologies; a lot of you are doing functional programming. You may be nodding along through parts of this, and if some of it's familiar, that's great. On the other hand, I'd hope you'd come away with some tools you could use to conduct a similar kind of discussion with other people you're trying to convince to do the right thing.

So I'll start with an appeal to authority: "Simplicity is a prerequisite for reliability." I certainly agree with this. I don't agree with everything Dijkstra said — I think he may have been very wrong about proof in particular — but I think he's right about this. We need to build simple systems if we want to build good systems, and I don't think we focus on that enough.

---

## Word Origins: Simple and Easy

I love word origins. They're tremendous fun. One reason is that words eventually come to mean whatever we all accept them to mean — whatever's commonly understood is what they mean. But it's often interesting to say, "I wish we'd go back to what this really means and use it that way." There are a couple of words I'll use in this talk whose origins I'd love for you to come away knowing, and to use more precisely, especially when talking about software.

The first word is **"simple."** Its roots are *sim* and *plex*, meaning *one fold* or *one braid* or *one twist*. And what does one twist look like? No twists, really. The opposite of the word is **"complex,"** which means braided or folded together. Being able to think about software in terms of whether or not it's folded together is the central point of this talk.

The other word we frequently use interchangeably with "simple" is **"easy."** The derivation runs through French; the last step is actually speculative, but I bought it because it serves this talk really well. It comes from the Latin root of "adjacent," which means *to lie near*, *to be nearby*. The opposite is **"hard,"** whose root has nothing to do with lying near or far — it means *strong*, or *tortuously so*.

So if we want to apply "simple" to the kind of work we do, we start with this concept of having one braid and look at it along a few dimensions. When we look for simple things, we want things that have one of something: one role, one task or job, one objective. They might be about one concept, like security, or about one particular dimension of the problem. The critical thing is that something simple has *focus* in these areas — you don't want to see it combining things.

But we can't get too fixated on "one." Simple doesn't mean there's only one of them, and it doesn't mean an interface with only one operation. It's important to distinguish *cardinality* — counting things — from actual *interleaving*. What matters for simplicity is that there's no interleaving, not that there's only one thing. That's very important.

The other critical thing about simple, as described, is that whether something is interleaved or not is an *objective* property. We can go and look: I don't see any connections; I don't see anywhere this twists with something else. So simple is an objective notion. That's also very important in distinguishing simple from easy.

---

## The Three Aspects of Easy

Let's look at easy. This notion of nearness is really cool, and there are several ways something can be near.

The first is *physical* nearness — it's right there. That's probably where the root came from: this is easy to obtain because it's nearby; I don't have to take a horse to the next town to get it. We don't have quite the same notion of physicality in software, but we do have our own hard drive, our own toolkit, and the ability to make things physically near through installers and the like.

The second notion is being near to our *understanding* — not as a capability, but literally near to something we already know. The word here is about being *familiar*.

I think we are collectively infatuated with these two notions of easy, and it's hurting us tremendously. All we care about is, "Can I get this instantly and start running it in five seconds?" It could be a giant hairball — all you care about is whether you can get it. And we're fixated on, "Oh, I can't read that." I can't read German. Does that mean German is unreadable? No — I just don't know German. That approach is not helpful. In particular, if you want everything to be familiar, you'll never learn anything new, because it can't be significantly different from what you already know.

There's a third aspect of easy we don't think about enough, and it's critical here: being near to our *capabilities*. We don't like to talk about this because it makes us uncomfortable. With easy in the context of violin or piano playing or mountain climbing, I don't feel bad if I don't play the violin well, because I don't play the violin at all. But our work is conceptual work, so when we start talking about something being outside our capability, it tramples on our egos. Due to a combination of hubris and insecurity, we never really discuss whether something is outside our capabilities. It turns out it's not so embarrassing after all, because we don't have tremendously divergent abilities in that area.

The last and critical thing distinguishing easy from simple: **easy is relative.** Playing the violin and reading German are hard for me; they're easy for certain other people. Unlike simple — where we can go look for interleaving — easy is always "easy for whom" or "hard for whom." The fact that we throw these terms around casually — "I like that technology because it's simple," when I really mean *easy*, and when I mean *easy* I really mean *I already know something that looks like it* — is how the whole discussion degrades, and we can never have an objective conversation about the qualities that matter in our software.

---

## Constructs and Artifacts

One critical area where we have to distinguish these two things has to do with **constructs and artifacts**. We program with constructs — programming languages, libraries — and those things have certain characteristics when we look at the code. But we're in the business of artifacts. We don't ship source code, and the user doesn't look at our source and say, "Oh, that's so pleasant." They run our software, and they run it for a long time. We keep glomming more stuff on; the running of it, the performance, the ability to change it — all of that is an attribute of the *artifact*, not the original construct.

Yet we still focus so much on our experience of *using* the construct: "Look, I only had to type sixteen characters! No semicolons!" This whole notion of programmer convenience — we are infatuated with it, and not to our benefit.

It gets worse: our employers are also infatuated with it, with those first two meanings of easy. If I can get another programmer in here who finds your source code familiar and already knows the toolkit, then it's near at hand and I can replace you. It's a breeze — especially if I ignore the third notion of easy, whether anybody can actually *understand* your code. They don't care about that; they just care that someone can sit in your seat and start typing. That focus on the first two aspects of easy makes programmers replaceable.

Contrast this with the impacts of long-term use. Does the software do what it's supposed to? Is it high quality? Can we rely on it? Can we fix problems when they arise, and change it given new requirements? These things have very little to do with the construct as we typed it in, and a lot to do with the attributes of the artifact. We have to start assessing our constructs based on the artifacts they yield, not the look and feel of typing them in or the cultural aspects of that.

---

## The Limits of Understanding

Let's talk about limits. This stuff is pretty simple logic: how can we possibly make reliable things we don't understand? It's very difficult. There's going to be a trade-off — as we make things more flexible, extensible, and dynamic, for some kinds of systems we'll trade away our ability to understand their behavior and ensure they're correct. But for the things we *do* want to understand and ensure are correct, we're going to be limited — limited in our understanding, which is itself very limited.

There's the whole notion of how many balls you can keep in the air, how many things you can keep in mind at once. It's a small number. We can only consider a few things, and when things are intertwined, we lose the ability to take them in isolation. Every time I pull out a part of the software I need to comprehend, and it's attached to another thing, I have to pull that other thing into my mind too, because I can't think about one without the other. That's the nature of intertwining. Every intertwining adds this burden, and the burden is combinatorial in the number of things we can consider. Fundamentally, complexity — this braiding together of things — limits our ability to understand our systems.

---

## Changing Software and Reasoning About Programs

So how do we change our software? I heard today that agile and extreme programming have shown refactoring and tests let us make a change with zero risk. I never knew that — and I still don't, because that's not actually a knowable thing. It's phooey. If you're going to change software, you're going to need to analyze what it does and make decisions about what it ought to do. At the very least you have to ask: what is the impact of this potential change, and what parts of the software do I need to go to in order to affect it? I don't care if you're using XP or Agile or anything else — if you can't reason about your program, you can't make these decisions.

Let me be clear, because as soon as people hear "reason about," they think I'm saying you have to *prove* programs. I am not; I don't believe that's an objective. I'm talking about *informal* reasoning — the same kind we use every day to decide what to do. We don't take out category theory to reason; thank goodness, we can reason without it.

What about the other side? There are two things you do with the future of your software: add new capabilities, and fix the ones you didn't get quite right. I like to ask: what's true of every bug found in the field? It got written. And here's a more interesting fact: it passed the type checker. And it passed all the tests. So now what do you do?

I call this world we live in **guard-rail programming.** It's really sad: "I can make changes because I have tests." Who drives their car around banging against the guard rails to get to the show on time? And do guard rails help you get where you want to go? Do they guide you anywhere? No — they're everywhere, and they don't point your car in any particular direction. So we're still going to need to think about our program. When all the guard rails fail us, we'll need to reason: "Through ordinary logic, this couldn't be in that part of the program — it must be over here. Let me go look there first."

Everybody's going to moan, "But I have all this speed! I'm agile and fast! This easy stuff makes my life good because I have speed." So — what kind of runner can run as fast as possible from the very start of a race? Only someone who runs really short races. But we're programmers and apparently smarter than runners, because we know how to fix that: we just fire the starting pistol every hundred meters and call it a new sprint. I don't know why runners haven't figured that out.

It's my contention, based on experience, that if you ignore complexity you will slow down — invariably, over the long haul. Of course, if you're doing something really short-term, you don't need any of this; you could write it with ones and zeros. Here's my very scientific graph — notice there are no numbers on the axes, because I completely made it up; it's experiential. If you focus on ease and ignore simplicity, you'll go as fast as possible from the beginning, but no matter what technology or sprints or starting pistols you use, the complexity will eventually kill you. It will make every sprint accomplish less; most sprints will be about redoing things you've already done, and the net effect is you're not moving forward in any significant way. If you start by focusing on simplicity, why can't you go as fast at the beginning? Some simple tools are as easy to use as tools that aren't. Where you can't go quite as fast is when you have to apply some simplicity work to the problem before you start — that's the ramp-up.

---

## Incidental Complexity

One problem we have is the conundrum that some things that are *easy* actually *are* complex. There are constructs with complex artifacts that are very succinctly described — some of the most dangerous things to use are so simple to describe and incredibly familiar. If you're coming from object orientation, you're familiar with a lot of complex things; they're very available, and they're easy to use. By all conventional measures you'd look at them and say, "This is easy."

But we don't care about that. The user isn't looking at our software and doesn't care how good a time we had writing it. What they care about is what the program does, and whether it works well will be related to whether the output of those constructs was simple — what complexity they yield.

When there is complexity there, we'll call it **incidental complexity.** It wasn't part of what the user asked for; we chose the tool, it had inherent complexity, and it's incidental to the problem. *Incidental* is Latin for "your fault." You really have to ask yourself: are you programming with a loom? You're having a great time throwing the shuttle back and forth, and what comes out the other side is a knotted mess. It may look pretty, but you have a problem — you've built a knitted castle. Do you want a knitted castle?

What do we get from simplicity? Ease of understanding, almost by definition. Ease of change and easier debugging. And on a secondary level: increased flexibility, and the ability to change policies or move things around — because as we make things simpler we get more independence of decisions, since they're not interleaved. I can make a location decision orthogonal to a performance decision. Now ask the agilist: is having a test suite and refactoring tools going to make changing the knitted castle faster than changing the Lego castle? No way — completely unrelated.

---

## How Do We Make Things Easy?

The objective here isn't just to bemoan the software crisis. So what can we do? Look at the aspects of easy.

There's the *location* aspect — making something at hand, in our toolkit. That's relatively simple: we install it. Maybe it's a little harder because we need someone to approve it. Then there's making it *familiar* — a learning exercise. Get a book, take a tutorial, have someone explain it, try it out. Both of these are in our hands; we're driving.

Then there's the *mental capability* part, which is always hard to talk about — maybe because the fact is we can learn more things, but we can't get much smarter. We're not going to move our brain closer to the complexity. We have to make things near by *simplifying* them. And the truth isn't that there are super-bright people who can do amazing things while everyone else is stuck. The juggling analogy is close: the average juggler can do three balls; the most amazing juggler in the world maybe nine or twelve — not twenty, not a hundred. We're all very limited compared to the complexity we can create; statistically we're all at about the same point in our ability to understand it, which is not very good.

So we have to bring things toward us, and because we can only juggle so many balls, we have to decide: how many of those balls do you want to be incidental complexity? How many extra balls do you want someone throwing at you — "Oh, use this tool" — *whoa*, more stuff. Who wants that?

---

## The Parentheses Example

Let me look at a complaint I've been on the other side of, because the analysis is quick and has nothing to do with usage — it's purely about the programmer experience.

"Parens are hard." They're *not at hand* for most people who haven't used them — they don't have an editor that does paren matching or structural editing, or they have one and never loaded the mode. Given. Nor are they *familiar* — everyone's seen parentheses, but not on *that* side of the method! That's crazy. But this is your responsibility to fix as a potential user.

Now dig deeper. Did the language actually give you something *simple* — free of interleaving and braiding? The answer is no. Common Lisp and Scheme are not simple in their use of parens, because the use of parentheses is overloaded: parens wrap calls, they wrap grouping, they wrap data structures. That overloading is a form of complexity by the definition I gave you. So even if you set up your editor and learned that the parenthesis goes on the other side of the verb, this was still a valid complaint — but for two reasons you *could* solve, plus one reason that was the *language designer's* fault: overloading. And we can fix that: add another data structure. Having more data structures doesn't make Lisp not Lisp — it's still a language defined in terms of its own data structures — but it lets us get rid of the overloading, which makes the remaining issue your fault again, just a familiarity thing you can solve for yourself.

There's an old dig at Lisp programmers. I'm not totally sure what it referred to — I believe it was performance-related, that Lispers consed up all this memory and did all this evaluation and Lisp programs at the time were complete pigs relative to the hardware. They knew the value of all these constructs — the dynamism, the dynamic nature — these things are great and valuable, but there was a performance cost. I'd like to lift that whole phrase and apply it to all of us right now. As programmers we read Hacker News and it's "Oh, this thing has this benefit, great, I'll do that — oh, this has *that* benefit, awesome, that's shorter." You never see in these discussions: was there a trade-off? Any downside? Anything bad that comes along with this? Never — we look only for benefits, and we're not looking carefully enough at the byproducts.

---

## What's in Your Toolkit

So I have two columns: complexity and simplicity — the simplicity column just being *simpler*, not purely simple. I didn't label them "bad" and "good"; I'll leave your minds to do that. Remember: **simple means unentangled, not "I already know what it means."**

State and objects are complex; values are simple and can replace them in many cases. Methods are complex; functions and namespaces are simple. (Methods are there partly because the class is often a very poor namespace.) Vars and variables are complex. Managed references are also complex, but still simpler. Inheritance, switch statements, and pattern matching are complex; polymorphism à la carte is simple. Syntax is complex; data is simple. Imperative loops, and even fold — which seems higher-level but still ties two things together — are complex; set functions are simpler. Actors are complex; queues are simpler. ORM is complex; declarative data manipulation is simpler. And eventual consistency is really hard for programs. Conditionals are complex in interesting ways; rules can be simpler. Inconsistency is almost definitionally complex, because *consistent* means "to stand together," so *inconsistent* means standing apart — and taking a set of things standing apart and trying to think about them all at once is inherently complex. Anybody who's used an eventually consistent system knows that.

---

## Complect

There's a really cool word: **complect.** I love it. It means *to interleave, entwine, or braid*. I don't want to say "braid" or "entwine," because they don't carry the bad connotation that "complect" does — complect is obviously bad. It's archaic, but there's no rule that says you can't use old words again. So what do you know about "complect"? It's bad. Don't do it. This is where complexity comes from: complecting. That's very simple. And it's something you want to avoid in the first place.

Look at this diagram — the first one and the last one. It's the *same stuff* in both, the exact same strips. What happened? They got complected. Now it's hard to understand the bottom diagram from the top, but it's the same stuff. You're doing this all the time. You can write a program a hundred ways. In some, it's just hanging there, all straight: you look at it and say, "Oh, I see, it's four lines." Then you type those four lines in another language or with a different construct and you end up with a knot. You've got to take care of that.

"Complect" means *braid together*; **"compose"** means *place together*. Everybody keeps telling us we want composable systems — to place things together — and there's no disagreement. Composing simple components is the way we write robust software. So it's simple: we make simple systems by making them modular. We're done! I'm halfway through my talk and don't even know if I'll finish.

Obviously that's *not* the key. Who has seen components with this nice characteristic? I'll raise my hand twice, because not enough of you are raising yours. You can write modular software with all kinds of interconnections between modules — they may not call each other, but they're completely complected. We know how to solve this, and it has nothing to do with there being two things; it has to do with what those two things are *allowed to think about*. What do we want them allowed to think about? Some abstractions — that dashed white version of the top of the Lego. That's all we want to limit them to, because now the blue guy doesn't know anything about the yellow guy and vice versa, and they both become simple.

It's very important not to associate simplicity with partitioning and stratification. They don't *give* you simplicity — they're *enabled* by it. If you make simple components, you can horizontally separate them and vertically stratify them. But you can also partition and stratify complex things and get no benefits. So be particularly careful not to be fooled by code organization. There are tons of libraries — "Look, separate classes, they call each other in nice ways" — and then out in the field you find this thing presumes that thing never returns 17. What is that?

---

## State

I'm not going to stand up here and tell you state is awesome. I'm not a "functional whatever" guy. Instead, I'll say: I did this, and it sucks. I did years and years of C++, He-Man stateful programming. It's really not fun. It's never simple. Having state in your program is never simple, because it has a fundamental complecting in its artifacts: it complects *value* and *time*. You don't have the ability to get a value independent of time, and sometimes not the ability to get a value in any proper sense at all. But it's a great example of "easy": it's totally familiar, it's at hand, it's in all the programming languages. This complexity is so easy.

And you can't get rid of it through ordinary code organization. Say I have modularity and that assignment statement is inside a method. If every time you call that method with the same arguments you can get a different result, that complexity just leaked right out. It doesn't matter that you can't see the variable — if the thing wrapping it is stateful, and the thing wrapping *that* is stateful (by stateful I mean: ask the same question, get a different answer), you have this complexity, and it's like poison, like dark liquid dropped into a vase — it ends up all over the place. The only time you can really get rid of it is when you put it inside something able to present a *true* functional interface on the outside: same input, same output.

Note: this is *not* about concurrency. It has nothing to do with concurrency — it's about your ability to understand your program. Your program was single-threaded and it didn't work; all the tests passed; it made it through the type checker. Figure out what happened. If it's full of variables, you'll need to recreate the state that was happening at the client when it went bad. Easy? No.

Your shiny new language has `var`, or maybe refs or references. None of these constructs make state *simple* — that's the first primary thing. I won't even say that of Clojure's constructs; they do not make state simple in the sense I'm talking about. But they're not the same: they all *warn* you when you have state. Most people using a language where immutability is not the default, and you have to go out of your way for it, find that their programs end up with orders of magnitude less state — because they never needed all that state in the first place. That's great. But I'll call out Clojure's and Haskell's references as particularly superior here, because they compose value and time: they're little constructs that do two things — provide some abstraction over time *and* the ability to extract the value. That's really important, because that extraction is your path back to simplicity. If I can get a value out, I can continue my program after passing that reference to somebody else. If instead I pass something that goes and finds the mutable variable every time, I'm poisoning the rest of my system. So look at the `var` in your language and ask whether it does the same thing.

---

## Why Things Are Complex

Let's see why these things are complex:

State complects everything it touches. Objects complect state, identity, and value, in a way you cannot extricate. Methods complect function and state, and in some languages they also complect namespaces — derive from two things in Java with the same method name, and it doesn't work. Syntax complects meaning and order, often in a very unidirectional way; Professor Sussman made a great point about data versus syntax, and it's super true — I don't care how much you love your favorite language's syntax, it's inferior to data in every way. Inheritance complects types — that's literally what it means, definitionally. Switching and matching complect multiple pairs of "who's going to do something" and "what happens," all in one place, in a closed way — that's very bad. Vars and variables complect value and time, often inextricably; you can't obtain a value. (We saw that amazing memory in a keynote where you dereference an address and get an object out — I want one of those computers. I called Apple; they said no. The only thing you can ever get out of a memory address is a word, a scalar — the thing everyone derided. Recovering a composite object from an address isn't something any computer we have actually does.) Loops complect what you're doing and how to do it; fold is subtler — it seems like someone else handled it, but it carries an implication about order, that left-to-right bit. Actors complect what's to be done and who does it. Object-relational mapping has — oh my God — complecting going on; you can't even begin to describe how bad it is. (What's the dual of a value — a *co-value*? It's an inconsistency. Who wants that?) And conditionals are interesting and more cutting-edge: we have rules about what our program should do, strewn all throughout the program, complected with the structure and organization of the program. Can we fix that?

---

## Choose Simpler Stuff

If you take away two things from this talk: one, the difference between "simple" and "easy"; two, the fact that we can create precisely the same programs we create now, with these tools of complexity, using dramatically simpler tools. I did C++, Java, C# for a long time. I know how to make big systems in those languages — and you do not need all that complexity. You can write just as sophisticated a system with much simpler tools, which means you'll focus on what the system is supposed to do instead of all the guck that falls out of the constructs.

The first step toward a simpler life is to choose simpler stuff:

For **values**, most languages have something — `final`, `val`, ways to declare something immutable. You'll want some persistent collections, because the harder thing is getting *aggregates* that are values; find a good library or use a language where that's the default. **Functions** — most languages have them, thank goodness; if you don't know what they are, they're like stateless methods. **Namespaces** you really need the language to do for you, and unfortunately it's not done well in a lot of places.

**Data** — please. We supposedly write data-processing programs, but there are all these programs with no data in them, just constructs we globbed on top of data. Data is really simple. There aren't a tremendous number of variations in its central nature: there are maps, sets, and linear sequential things — not a lot of other conceptual categories. We create hundreds of thousands of variations that have nothing to do with the essence of the stuff, and make it hard to write programs that manipulate that essence. We should just manipulate the essence; it's not hard, it's simpler. Same for communications: aren't we all glad we don't use the Unix model on the web, where any arbitrary command string is your argument list and any characters can come back — let's all write parsers? No. Just use data.

The most desirable and most esoteric thing — tough to get, but transformative — is **polymorphism à la carte**. Clojure protocols, Haskell type classes, and similar constructs give you the ability to independently say: I have data structures, I have definitions of sets of functions, and I can connect them — three independent operations. The genericity isn't tied to anything in particular; it's available à la carte. I don't know of many library solutions for languages that lack it.

**Managed references** we've covered. **Set functions, queues** — get them from libraries; you don't need a special communication language. **Declarative data manipulation** — use SQL (or finally learn it), or something like LINQ or Datalog; these are harder, with fewer well-integrated options. **Rules** — instead of embedding conditionals at every point of decision, gather them somewhere else; use rules libraries or languages like Prolog. For **consistency**, use transactions and values.

There are reasons you might have to get off this list, but no reason not to start with it.

---

## Environmental Complexity

There's a source of complexity that's difficult and *not* your fault — call it **environmental complexity.** Our programs run on machines next to other programs and next to other parts of themselves, and they contend for resources: memory, CPU cycles. This is inherent. *Inherent* is Latin for "not your fault" — it's not part of the problem, but it's part of the implementation. You can't tell the customer the thing they wanted is no good because you have GC problems.

There aren't many great solutions. You can do segmentation — "this is your memory, this is your CPU" — but there's tremendous waste, because you pre-allocate and don't use everything and have no dynamic nature. The harder problem, for which I don't have a solution: the policies around this don't compose. If everybody says "I'll just size my thread pool to the number of cores," how many times can you do that in one program and have it still work out? Not many. Splitting that up and making individual decisions isn't actually making things simpler — it's making them complex, because that's a decision that should be made by someone with better information, and we don't have good sources for organizing those decisions in single places in our systems.

---

## Designing Simple Things

There's a long quote that basically says: programming is not about typing, it's about thinking. So how do we design simple things of our own? The first part is just choosing constructs with simple artifacts. But we sometimes have to write our own constructs, so how do we abstract for simplicity?

**Abstract** — here's an actual definition, not a made-up one — means *to draw something away*, particularly to draw away from the physical nature of something. I want to distinguish this from the gross usage where people use "abstraction" to mean *hiding stuff*. That is not what abstraction is, and it won't help you here.

One approach: just do **who, what, when, where, why.** Go through everything you're deciding to do and ask, "What's the *who* aspect of this? The *what* aspect?" This helps you take things apart. The other thing is to maintain the attitude, "I don't know, and I don't want to know." I once said that so often during a C++ course that a student made me a shirt — a Booch diagram where every line just said "I don't want to know." That's what you want.

**What.** What are the operations — what do we want to accomplish? We form abstractions by taking sets of functions and giving them names, using whatever your language provides: interfaces, protocols, type classes. They should be really small — much smaller than what we typically see. Java interfaces are huge, mostly because Java lacks union types, so it's inconvenient to say "this function takes something that does this *and* that." Those giant interfaces make programs much harder to break up. Represent them with your polymorphism constructs; they're *specifications*, not implementations, and they should only use values and other abstractions in their definitions. The biggest problem in this part of design is complecting *what* with *how* — either bluntly, by jamming a concrete function or class in instead of an interface, or more subtly, by having the semantics of the function dictate how it's done. Fold is an example. Strictly separating *what* from *how* is the key to making *how* somebody else's problem: you can tell the database engine or the logic engine, "You figure out how to do this."

**Who.** This is about data or entities — the things abstractions connect to. Build components from subcomponents in a direct-injection style; don't hardwire what the subcomponents are. Take them as arguments for programmatic flexibility. You should probably have many more subcomponents, and much smaller interfaces, than you typically do — usually you have none, then maybe one when you decide to farm out policy. If you do who/what/when/where/why and find five components, don't feel bad — you're winning massively. Just beware of hidden detail dependencies between subcomponents.

**How.** This is the actual implementation, the work. Connect things together using polymorphism constructs — the most powerful tool. You *can* use a switch or pattern matching, but that gloms everything together. With a polymorphism system you have an open policy, especially powerful if it's runtime-open, but better than nothing even if not. Again, beware of abstractions that dictate *how* in some subtle way, because you're nailing down the person downstream who has to implement. The more declarative, the better. These implementations should be islands as much as possible.

**When and where.** Strenuously avoid comparing this with anything. It accidentally creeps in when people design systems with directly connected objects. If your program is architected so thing A calls thing B, you've just complected it: now A has to know *where* B is in order to call it, and *when* it happens is whenever A does it. Stick a queue in there. Queues are the way to get rid of this problem. If you're not using queues extensively, you should be — right after this talk.

**Why.** This is policy and rules. This is hard for us. We put it all over the application, and if you ever have to talk to a customer about what the application does, it's really difficult to sit with them in source code. And if you have one of those pretend testing systems that lets you write English strings so the customer can read them — that's just silly. You should have *code* that does the work and that someone can look at, which means putting this stuff somewhere outside in a declarative or rule system.

**Information.** It is simple. The only thing you can possibly do with information is ruin it — so don't. Objects were made to encapsulate I/O devices: there's a screen I can't touch, so I have an object; there's a mouse I can't touch, so I have an object. That's all they're good for. They were never supposed to be applied to information, and we apply them to information. That's just wrong — and now I can say *why* it's wrong: because it's complex. It ruins your ability to build generic data-manipulation things. If you leave data alone, you can build things once that manipulate data and reuse them everywhere — write once, done. It also ties your logic to representational things — more intertwining. So represent data as data. Please start using maps and sets directly. Don't feel you have to write a class just because you have a new piece of information. That's silly.

---

## Simplifying What Already Exists

The final aspect: sometimes we have to simplify other people's stuff — the problem space, or code someone else wrote. This is a whole separate talk, so I won't get into it now. But the job is essentially **disentangling.** We know what's complex — it's entangled — so we need to disentangle it. You'll need to figure out where things go, follow stuff around, and eventually label everything. That's roughly the process, but a full treatment is its own talk.

---

## Wrap-Up

The bottom line: **simplicity is a choice.** It's your fault if you don't have a simple system. We have a culture of complexity, and to the extent we keep using tools with complex outputs, we're in a self-reinforcing rut, and we have to get out of it. If you're already saying, "I know this, I already use something better — I use that whole right column," then I hope this talk gives you the basis for talking with somebody who doesn't yet believe.

It is a choice, and it requires constant vigilance. We saw that guard rails don't yield simplicity. It requires sensibilities and care. Your sensibilities about simplicity being equal to ease of use are simply wrong — we saw the definitions; easy is not simple. You have to develop sensibilities around entanglement. You have to have *entanglement radar*: look at software and notice not "I don't like your names" or "there's a missing semicolon" (also important, but separate) — start *seeing* complecting. Start seeing interconnections between things that could be independent. That's where you get the most power. All your reliability tools — since they're not about simplicity — are secondary. They're safety nets, nothing more.

So how do we make simplicity easy? Choose constructs with simpler artifacts and avoid constructs with complex artifacts. It's the *artifacts*, not the authoring — as soon as you get into an argument with somebody about "we should be using whatever," get that sorted out, because however they feel about the shape of the code they type is independent from this, and this is the thing you have to live with. Create abstractions that have simplicity as a basis. Spend a little time up front simplifying things before you start, and recognize that when you simplify, you often end up with *more* things. Simplicity is not about counting. I'd rather have more things hanging nice and straight, not twisted together, than a couple of things tied in a knot. And the beautiful thing about keeping them separate is that you'll have far more ability to change it — which is where the benefits lie.

So I think this is a big deal, and I hope you can bring it into practice, or use it as a tool to convince someone else to. I'll leave you with this: this is what you say when somebody tries to sell you a sophisticated type system. Thank you.
