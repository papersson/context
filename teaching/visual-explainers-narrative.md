# How to Make Exceptional Visual Technical Explainers

## The central answer

The best technical explainer is not the one that transfers the most information per minute. It is the one that **causes the right sequence of changes in the learner's mental model**.

That distinction turns out to reconcile much of what is excellent about 3Blue1Brown, Andrej Karpathy, Sebastian Lague, Ben Eater, Bartosz Ciechanowski, Veritasium and Steve Mould, while also explaining why the work of Andy Matuschak, Michael Nielsen, Bret Victor and Paul Graham is unusually relevant. Grant Sanderson explicitly describes deep understanding as the foundation of his goal for 3Blue1Brown; Karpathy teaches by constructing systems from first principles; Ben Eater structures networking around the problem each layer solves; Ciechanowski repeatedly lets a phenomenon appear before supplying the machinery needed to explain it; Derek Muller has experimentally studied the value of confronting alternative conceptions rather than merely presenting the right answer. citeturn24view6turn25view1turn25view0turn25view3turn27view0

The most useful synthesis I can give you is:

> **A great technical explainer is a guided investigation in which the viewer repeatedly notices a gap, makes or anticipates a prediction, acquires the minimum new idea needed to resolve the gap, and then uses that idea somewhere new.**

The word *narrative* is important, but I would not think of narrative primarily as character, drama, historical anecdote, or even story in the conventional sense. For technical exposition, the strongest source of narrative tension is usually **epistemic tension**: *Why does this happen? How could this possibly work? Why did the obvious approach fail? What rule would make all these observations fit together?* That is the common underlying move in the puzzle-driven work of 3Blue1Brown, the build-from-scratch structure of Karpathy and Ben Eater, Ciechanowski's progression from observed behaviour to mechanism, and Muller's misconception-and-resolution approach. This is partly my synthesis rather than an experimentally established universal law, but it is unusually consistent across these very different creators. citeturn25view2turn25view1turn25view0turn25view3turn27view0

There are also three distinctions worth keeping in mind from the outset.

**Engagement is not learning.** Guo, Kim and Rubin analysed 6.9 million viewing sessions across four edX courses and found strong relationships between production choices, length and viewing behaviour. But they explicitly treated engagement as a necessary, *not sufficient*, prerequisite for learning. A retention graph can therefore tell you where people leave; it cannot tell you whether the people who stayed formed a usable mental model. citeturn26view3

**Understanding is not recognition.** A viewer can follow every sentence, nod throughout, and still be unable to reconstruct the explanation tomorrow. Karpicke and Roediger found that repeated retrieval had large benefits for delayed recall in their experiments even when further studying did not, and participants were poor at predicting their later performance. That is a warning against judging an explainer entirely by how smooth it feels while watching. citeturn26view1

**Passivity is not inevitable, even in a linear video.** Matuschak's criticism of conventional books and lectures is that their implicit model is often transmission: the author says the idea, the learner receives it, therefore learning has occurred. His alternative is to design the medium around what the learner must *do mentally*. Bret Victor makes a related case for explanations that encourage readers to question assumptions, vary examples and generalise. Experimental work on retrieval and self-explanation gives these intuitions a stronger empirical foundation. citeturn24view1turn24view3turn26view1turn23search2

So I would optimise an explainer for a hierarchy like this:

| What you want | A good test |
|---|---|
| **Attention** | Did they keep watching? |
| **Comprehension** | Can they explain what each part means immediately afterwards? |
| **Mental model** | Can they predict what happens when something changes? |
| **Transfer** | Can they use the idea in a case they have not seen? |
| **Retention** | Can they reconstruct the important parts later? |
| **Taste / motivation** | Do they now notice interesting questions they previously would not have seen? |

The first item is easiest to measure and therefore easiest to over-optimise. The deeper items are much closer to what the experimental literature on transfer, conceptual change, self-explanation and retrieval actually attempts to measure. citeturn20view0turn20view1turn27view0turn26view1turn23search2

My strongest overall recommendation is therefore: **do not set out to make a video that “explains topic X”. Set out to produce one durable change in what the viewer can see, predict, or reason about.** Everything else—runtime, examples, definitions, notation, history, jokes, visuals—should be subordinate to that change.

## What the great explainers are actually doing

The creators you named are less alike stylistically than they first appear. That is useful, because their differences reveal several distinct narrative engines you can deliberately choose between.

| Creator | Dominant narrative engine | What I would steal |
|---|---|---|
| **3Blue1Brown** | A problem becomes intelligible after a change of perspective | Make the *representation itself* part of the story: “Here is why the old way of seeing this is awkward; here is the viewpoint from which it becomes natural.” citeturn24view6turn25view2 |
| **Andrej Karpathy** | Construction from primitives | Do not merely describe the finished abstraction. Recreate the sequence of needs that makes the abstraction worth inventing. His Zero to Hero course literally builds from backpropagation towards modern language models in code. citeturn25view1 |
| **Ben Eater** | Each layer solves a concrete engineering problem | Ask, “What breaks if we stop here?” His Internet series explicitly proceeds one layer at a time, explaining the problem each layer solves and how it contributes to end-to-end communication. citeturn25view0 |
| **Sebastian Lague** | Project as expedition | The “Coding Adventure” framing turns exposition into progress towards an artefact: attempt, obstacle, refinement, result. The viewer learns because each concept becomes necessary to advance the project. This is an interpretive description of his published format rather than an experimental claim. citeturn14search1turn14search19 |
| **Bartosz Ciechanowski** | Observe first, decompose afterwards | Begin with the thing you ultimately want to explain, then construct progressively simpler sandboxes. His Moon article starts with familiar lunar phenomena, moves into a simplified orbital playground, and only then introduces equations and additional mechanisms. citeturn25view3 |
| **Veritasium** | Intuition collides with evidence | Let viewers commit to an intuitive model before replacing it. Muller's research found that a dialogue containing alternative conceptions produced more reported mental effort and higher post-test scores than a standard lecture-style treatment in one experiment. citeturn24view7turn24view8turn27view0 |
| **Steve Mould** | A strange physical phenomenon creates the question | Let wonder precede terminology: the unusual behaviour is not decoration around the explanation; it *creates the need for the explanation*. His work on phenomena such as the chain-fountain/Mould effect is a representative example of this style. citeturn12search38turn12news46 |
| **Michael Nielsen / Andy Matuschak** | Remembering and using are part of the medium | Treat the end of exposure as the beginning, not the end, of learning. Nielsen's *Augmenting Long-term Memory* focuses on spaced repetition; Matuschak's current work explicitly aims to help readers understand, remember and use what they read. citeturn28search0turn25view5 |
| **Bret Victor** | Explanation as a thing to think *with* | Ask which assumptions or variables the learner ought to manipulate mentally—or literally. Victor defines active reading in terms of questioning assumptions, generating examples and exploring consequences. citeturn24view3 |
| **Paul Graham** | Removal of linguistic friction | Put the intellectual complexity in the *idea*, not in the sentence describing it. Graham deliberately uses ordinary words and simple sentences so readers spend their energy on the ideas. citeturn24view2 |

There is an especially important lesson in Grant Sanderson's reflection on his “windmill” problem. Once a solution is familiar, it becomes extremely difficult to recover the experience of *not* seeing it. Sanderson says that in mathematics, judging what another person will find difficult can be harder than the mathematics itself. This is the expert's fundamental pedagogical hazard: your brain has compressed a chain of ten once-difficult inferences into one obvious-looking chunk. citeturn25view2

That means the job of an explainer is not simply to discover the shortest proof or cleanest finished explanation. Often you need to **reconstruct the missing intermediate states of mind**.

Consider the difference:

> “TCP provides reliable byte-stream delivery using sequence numbers and acknowledgements.”

versus:

> “We can now send packets across the network. But suppose packet 7 disappears. How would the receiver even know? We need some way to say what arrived, what did not, and what order it belonged in. Numbering the pieces gives us the first ingredient…”

The second is longer in words but often shorter in *cognitive distance*. Ben Eater's layer-by-layer description of the Internet exemplifies precisely this problem-driven progression: an abstraction appears because the previous system has exposed a concrete deficiency. citeturn25view0

Karpathy uses a related move in a different medium. Instead of saying, “Here are the components of a neural network training system”, the learner implements increasingly sophisticated pieces. His course starts from basic backpropagation and progressively builds towards modern neural networks; he describes the opening lecture as an unusually step-by-step treatment and assumes only Python plus introductory mathematics. citeturn25view1

Ciechanowski does the same conceptually. In the Moon article, the equation for gravitational force does not arrive as an obligatory definition at the beginning. The reader first moves bodies around, sees trajectories change, observes that greater mass increases force and greater distance weakens it, and only then encounters the compact equation describing those relationships. citeturn25view3

This leads to a principle I would make almost sacred:

> **Never introduce an abstraction before the learner has experienced the job that abstraction is being hired to do.**

Not every topic permits a literal historical reconstruction of how the concept was invented, nor should you force one. But almost every technical concept has some explanatory, predictive, compressive or engineering purpose. Give the viewer that purpose before—or at least simultaneously with—the name.

Veritasium suggests a second principle: sometimes the obstacle is not absence of knowledge but the presence of a plausible wrong model. Muller's 2008 study involved 272 students across experiments on Newton's laws. In the first experiment, students watching a dialogue containing alternative conceptions reported greater mental effort and achieved higher post-test scores than those watching a standard lecture-style presentation; interviews suggested the alternatives prompted more active attempts to understand. citeturn27view0

That changes what “clear explanation” means. Suppose the viewer believes heavier objects must fall faster. A beautifully animated explanation of \(F=ma\) may coexist peacefully with that intuition. A stronger narrative makes the old intuition produce a prediction, lets reality put pressure on it, and *then* supplies a replacement model. The misconception becomes the antagonist—not the learner.

Matuschak and Nielsen then add the part most YouTube exposition omits: a correct mental model that exists only during playback is fragile. Matuschak argues that conventional books and lectures leave the learner responsible for doing the metacognitive work of questioning, summarising and checking understanding. Nielsen's work on memory systems, and Matuschak and Nielsen's experiments with mnemonic media, are attempts to build remembering into the medium itself. citeturn24view1turn28search0turn28search1

The great opportunity for a visual explainer, therefore, is to combine these traditions:

**Sanderson's perspective shifts + Karpathy/Eater's constructive necessity + Lague's forward momentum + Ciechanowski's concrete-to-mechanism progression + Muller's conceptual conflict + Nielsen/Matuschak's active remembering + Graham's linguistic simplicity.**

That combination is more interesting to me than imitating the superficial style of any one creator.

## What learning science adds to the craft

The research does not hand us a recipe called “the scientifically optimal YouTube explainer”. Studies use different subjects, learner populations, durations and outcomes, and many were conducted in courses or laboratory-like settings rather than voluntary public-media viewing. It is therefore more defensible to extract robust design constraints than to pretend there is a single experimentally proven format. Guo et al.'s large study measured engagement; Mayer and Chandler studied multimedia transfer in a very short lightning animation; Muller studied conceptual change in physics; retrieval studies often use text or vocabulary materials. citeturn26view3turn20view0turn27view0turn26view1

Within those limits, several findings matter enormously for narrative explainers.

**Make the learner retrieve rather than merely re-hear.** Karpicke and Roediger's 2008 experiment found that repeated testing after initial learning produced a large improvement in delayed recall whereas repeated studying after learning did not in that paradigm. A related literature shows benefits of retrieval practice across educational materials, and Karpicke and Blunt found retrieval practice could outperform elaborative concept mapping on measures of meaningful learning. citeturn26view1turn6search13turn6search17

For video, the right translation is not “interrupt every ninety seconds with a multiple-choice quiz”. It is to build *retrieval beats* into the narrative:

> “Before I show you the next step, what should happen?”

> “Could you reconstruct the rule we just derived?”

> “If I double this quantity, which part of the behaviour should change?”

> “We saw three mechanisms. Which one explains this new case?”

Even a linear video can leave two or three seconds of genuine silence before answering. The important cognitive act is that the viewer attempts to produce the model rather than merely recognises it when you say it. This recommendation is an application of retrieval-practice findings rather than something those studies tested specifically in YouTube videos. citeturn26view1turn6search17

**Ask for self-explanation.** Chi and colleagues' work found that eliciting self-explanations can improve understanding, building on earlier observations that stronger learners spontaneously generated more explanations while studying worked examples. For a technical video, “Why must this step be true?” can be more valuable than another thirty seconds of narration from you. citeturn23search2turn23search6

This suggests a useful distinction between two kinds of rhetorical question. A decorative rhetorical question—“So what happens next?” immediately followed by the answer—mainly creates cadence. A pedagogical question gives the viewer enough information, enough time and enough reason to make an actual attempt. Only the latter turns the video into something approaching active practice.

**Surface important misconceptions instead of silently avoiding them.** Muller's physics work is unusually relevant here because it is directly about multimedia explanation. The alternative-conception dialogue in his 2008 experiment produced higher post-test scores than the standard lecture-style condition and appeared to induce more active processing. citeturn27view0

There is a subtle implication: the smoothest possible narrative is not always the best learning experience. Some useful intellectual friction can force the viewer to reconcile competing models. Muller framed this as increasing *useful* cognitive load rather than treating all mental effort as undesirable. citeturn27view0

**Segment complex explanations and give the learner places to breathe.** Mayer and Chandler compared narrated animations about lightning formation. In one experiment, learners who were allowed to control the pace of a segmented presentation before seeing the whole presentation performed better on transfer than learners receiving the same presentations in the reverse order; in another, learner-controlled segmented presentations again improved transfer relative to uninterrupted whole presentations. Their material was only about 140 seconds long and the segments about ten seconds, so this is evidence for *pacing and segmentation*, not evidence that YouTube chapters should be ten seconds long. citeturn20view0turn20view1

Narratively, that means a long explanation should not feel like one fifty-minute sentence. Give it natural points of closure: “We now know X. The remaining mystery is Y.” Those sentences are not just chapter markers. They help the viewer compress what has just happened before you open the next unresolved question. This is a design inference from segmentation research and the chaptered construction seen in creators such as Eater and Karpathy. citeturn20view0turn25view0turn25view1

**Do not confuse active learning with unguided discovery.** Matuschak's critique of passive transmission is valuable, but it should not be interpreted as “make beginners discover everything themselves”. Cognitive-load research finds that novice learners can benefit substantially from explicit guidance and worked examples, while the same instructional support can become redundant or counterproductive as expertise rises—the “expertise reversal effect”. citeturn22search4turn22search20

This is an important design principle for your audience. A beginner explainer should usually provide a strong rail: carefully chosen examples, explicit intermediate steps, constrained predictions. An expert-oriented explainer can omit more scaffolding and let the viewer infer more. You cannot meaningfully optimise an explanation without specifying **who already knows what**. citeturn22search20turn25view2

And this gives us perhaps the most useful distinction of all:

> **The learner should do the thinking; the teacher should do the instructional design.**

You do not need to make the learner rediscover calculus. You *do* want them to make the crucial prediction, notice the contradiction, articulate the causal link or reconstruct the concept at the point where doing so strengthens the model.

## How long your explainer videos should be

There is **no scientifically defensible universal optimal length**.

The often-repeated “six-minute educational video” rule comes largely from Guo, Kim and Rubin's 2014 analysis of 6.9 million edX watching sessions. They found that shorter videos were substantially more engaging and recommended planning MOOC material as chunks shorter than six minutes. But their outcome was viewing engagement—how long learners watched and whether they attempted post-video assessment—not durable conceptual understanding. The authors explicitly say engagement is necessary but insufficient for learning. citeturn26view3

This matters enormously. “A six-minute MOOC clip has better normalised viewing engagement than a thirty-minute MOOC lecture” does **not** imply “a six-minute Veritasium-style explanation teaches more than a twenty-minute one”.

The context also matters. Karpathy's Zero to Hero syllabus includes individual videos lasting approximately 56 minutes, 75 minutes, 1 hour 55 minutes, 1 hour 57 minutes and 2 hours 25 minutes. Those videos are not comparable to general-audience explainers: they are deliberate, high-intent, build-along technical instruction. Their existence certainly does not prove that long videos cause better learning, but it demonstrates why treating a single runtime threshold as a universal property of educational media is conceptually wrong. citeturn25view1

The stronger conclusion from the evidence is:

> **Optimise the length of the cognitive arc, then segment the presentation. Do not optimise total minutes in isolation.**

Mayer and Chandler's work strengthens that interpretation. Learner control and segmentation improved transfer in their multimedia experiments, suggesting that a complex whole can be easier to learn when it arrives in manageable, learner-paced parts. Again, their experiment was not a YouTube-runtime study, so the conclusion should be about structure rather than a magic number. citeturn20view0turn20view1

For your kind of work, I would use the following **design priors**, not rules:

| Format | My starting runtime prior | Narrative requirement |
|---|---:|---|
| One surprising fact or misconception | **4–8 min** | One question, one model change, one test |
| Focused visual concept explainer | **8–15 min** | One central mental model; very little branching |
| Rich mechanism / mathematical idea | **12–25 min** | Several internal acts, each producing partial closure |
| Deep conceptual investigation | **20–40 min** | Strong chapter structure and recurring central question |
| Coding / hardware / mathematical build-along | **30–150+ min** | Viewer is actively following; checkpoints and resumable chapters are essential |

Those ranges are my synthesis of the engagement evidence, segmentation research and the formats represented by the creators in your question; they are **not experimental estimates of optimum learning duration**. Guo supports a preference for shorter chunks in low-commitment course video; Mayer and Chandler support segmentation; Karpathy shows why high-intent tutorial viewing can rationally occupy a completely different duration regime. citeturn26view3turn20view0turn25view1

For a channel aspiring to the territory between **3Blue1Brown, Ciechanowski, Veritasium and Sebastian Lague**, my practical default would be approximately **10–20 minutes**, with permission to go shorter whenever the idea genuinely fits and longer whenever the conceptual arc genuinely requires it. I would be suspicious when a script crosses roughly twenty minutes without containing independently satisfying internal chapters. That is a craft recommendation derived from the evidence above rather than a published threshold. citeturn26view3turn20view0turn20view1

I would design those chapters so that roughly every few minutes the viewer arrives at some new stable state:

> “Now we can explain the first mystery.”

> “So that handles the simple case—but it immediately gives us a new problem.”

> “We have built the mechanism. Now let's see whether it predicts the strange behaviour we started with.”

This creates a rhythm of **open loop → reasoning → local closure → new loop**. It gives you the engagement benefits of shorter conceptual units without requiring every subject to be chopped into disconnected six-minute uploads. That recommendation is consistent with segmentation research and with the layered explanatory structures visible in Eater, Ciechanowski and Karpathy. citeturn20view0turn25view0turn25view3turn25view1

A useful editing test is therefore not “Can I make this twelve minutes?” but:

> **Is there any thirty-second stretch in which the viewer cannot tell what question the current material is helping answer?**

If so, the problem may not be length. It may be lack of narrative function.

And another:

> **If I remove this section, does the central mental model become weaker?**

If the answer is no, you have probably found a tangent, however interesting it is.

That idea parallels Paul Graham's approach to prose: he writes quickly and then spends considerable time editing, much of it cutting, with the aim of minimising the work imposed by the words themselves. citeturn24view2

## A narrative architecture for technical explainers

I would build most of your explainers around a seven-stage arc. It is not presented as a validated psychological model; it is a practical synthesis of the strongest patterns above.

**Begin with the phenomenon, capability, or puzzle.**

Do not begin with “Today we're going to learn about eigenvectors.” Begin with something eigenvectors make intelligible. Do not begin with the taxonomy of internet protocols. Begin with two computers trying—and failing—to communicate reliably. Ciechanowski's Moon article opens by showing the Moon's changing, sometimes puzzling behaviour and explicitly promises to explain those effects later; Ben Eater's Internet series is organised around the problem each layer must solve. citeturn25view3turn25view0

The opening should cause the viewer to want a model.

A good cold open therefore often takes one of four forms:

| Opening | Example structure |
|---|---|
| **Anomaly** | “This seems to violate what you expect.” |
| **Capability** | “By the end, we will have built this from almost nothing.” |
| **Puzzle** | “There is an embarrassingly simple question here that turns out to be hard.” |
| **Conflict** | “Two perfectly reasonable ways of thinking about this give different predictions.” |

These correspond closely to the phenomenon-first work of Mould and Ciechanowski, the construction narratives of Karpathy and Eater, the mathematical puzzles of 3Blue1Brown and the alternative-conception approach studied by Muller. citeturn12news46turn25view3turn25view1turn25view0turn25view2turn27view0

**Get the viewer to commit to a prediction.**

Before explaining, ask what they expect. This is especially powerful when the topic contains a common misconception. Muller's conceptual-change work suggests that explicitly bringing alternative conceptions into the learning experience can promote more active engagement with the underlying physics; retrieval and self-explanation research also supports making learners produce an answer rather than merely receive one. citeturn27view0turn26view1turn23search2

The prediction does not have to be correct. In fact, a wrong prediction can create the precise intellectual tension the rest of the section resolves.

The important thing is not to humiliate the viewer. Veritasium's strongest pedagogical mechanism is not “Ha, you were wrong”. It is:

> “That answer makes sense. Here is the hidden assumption inside it. Let's see what happens when reality tests that assumption.”

That formulation is my synthesis of Muller's published work on alternative conceptions and conceptual change. citeturn24view8turn27view0

**Construct the smallest model that can make progress.**

This is the Karpathy/Eater move. Start with primitives. Do not announce the final architecture and then tour its components; build until the current system encounters a limitation, then introduce the next idea because it fixes that limitation. citeturn25view1turn25view0

A particularly strong sentence pattern is:

> “So far, this lets us do X. But it still cannot do Y.”

That one sentence simultaneously performs recap, identifies a limitation and creates the motivation for the next abstraction.

Notice how different this is from:

> “The next concept we need to cover is Y.”

The first sentence has causality. The second has curriculum.

**Alternate concrete and abstract.**

Ciechanowski's Moon article repeatedly moves from manipulable examples and observed motion into mathematical compression, then back out to phenomena. Bret Victor argues that explorable examples can make abstractions concrete and let readers develop intuition by seeing consequences of changes. citeturn25view3turn24view3

I would think of this as breathing:

**example → pattern → abstraction → prediction → example**

rather than:

**definition → definition → notation → theorem → example at the end**

The abstraction is the compression of experience the viewer already partially possesses.

**Introduce friction before the model feels finished.**

Once the viewer has a provisional understanding, stress-test it. Use an edge case, a changed parameter, an apparent contradiction, or the common misconception you deliberately postponed.

Why? Because understanding that survives only the example used to teach it may be imitation rather than transfer. Mayer and Chandler used transfer measures to distinguish deeper understanding from simple retention; Muller's work similarly aimed at conceptual change rather than mere presentation of correct statements. citeturn20view0turn27view0

This is also where a Sebastian Lague-style “adventure” becomes pedagogically useful rather than merely entertaining. A failed implementation can reveal why a technical constraint matters. But do not preserve every dead end simply because it happened in the real development process. Show failures whose *diagnosis* teaches the model. Lague's “Coding Adventure” framing and Matuschak's interest in showing unfinished process provide the craft inspiration here; the filtering criterion is mine. citeturn14search1turn28search18

**Compress the understanding.**

Near the end, return to the opening phenomenon and explain it again, now almost effortlessly.

This is one of the most satisfying moves available to a technical writer. A phenomenon that needed ten minutes of investigation should now admit a compact explanation because the viewer possesses the right concepts.

Then state the core model in its smallest useful form. Not a generic “So that's how X works”, but something the viewer could carry away:

> “The key is that each node only needs local information; the global behaviour emerges from repeated local updates.”

or:

> “The derivative is not an extra number attached to the function; it describes how the function itself responds to an infinitesimal change.”

The final sentence should compress, not merely summarise.

Paul Graham's philosophy of making the words recede behind the idea is especially relevant here: ordinary words and simple sentences leave more of the reader's effort available for the underlying thought. citeturn24view2

**End with transfer, not repetition.**

Do not finish by replaying the same example. Give the viewer a new case and let them use the model.

> “So what should happen if the planet were twice as massive?”

> “Would this protocol still work if packets arrived out of order?”

> “Can you now see why this apparently unrelated algorithm has the same structure?”

Then pause.

Self-explanation research, retrieval-practice experiments and Mayer's use of transfer measures all point towards the importance of getting beyond immediate re-exposure to active production and application. citeturn23search2turn26view1turn20view1

The complete arc is therefore:

**Phenomenon → prediction → construction → abstraction → friction → compression → transfer.**

When you have a script that feels flat, I would locate where this chain breaks. Very often the problem is that the video goes directly from **topic → information → more information → summary**, so nothing creates the need for the next idea.

## The craft of the script

Once the macro-narrative is right, sentence-level writing becomes extraordinarily important because narration competes for the same limited attention the viewer needs for the technical idea. Paul Graham's dictum is almost perfectly suited to explainer scripts: ordinary words, simple sentences, less energy spent on prose and more available for ideas. citeturn24view2

There are several writing rules I would adopt aggressively.

**Write for the ear, but reason for the mind.**

A sentence can be technically precise and still be impossible to parse in real time. Unlike prose, spoken narration does not let the viewer casually move their eyes backwards three clauses. So split logical dependencies across sentences.

Instead of:

> “Because the gradient, which is calculated with respect to each parameter and whose magnitude depends on the local geometry of the loss surface, points in the direction of steepest increase, we negate it to minimise the loss.”

try:

> “The gradient tells us the direction in which the loss increases fastest. We want the opposite. So we negate it.”

You have not removed intellectual content. You have made its causal sequence audible. This is consistent with Graham's emphasis on ordinary words and uncomplicated sentences; the exact rewriting rule is my application to narrated exposition. citeturn24view2

**Prefer causal sentences to descriptive sentences.**

A weak technical script contains many forms of “X has Y”. A strong one contains *because*, *therefore*, *if*, *so*, *which means*, *but* and *unless*.

Compare:

> “The cache contains recently accessed data.”

with:

> “Memory is slow compared with the processor. So we keep recently used data closer to the processor, betting that we will need it again soon.”

The first gives a fact. The second gives the model that makes the fact reconstructible.

Ben Eater's Internet description—each layer is explained in terms of the problem it solves—is an especially pure version of this explanatory orientation. citeturn25view0

**Introduce names after meanings whenever possible.**

Experts often teach in dictionary order:

> “A monoid is…”

The learner often benefits from invention order:

> “Suppose we want to combine any number of these objects, including none at all, without caring how we parenthesise the combinations. What properties would the operation need?”

Then:

> “That structure has a name: a monoid.”

You have made the name compression rather than burden.

This is not a rule to ban precise definitions; it is a rule about sequencing. Ciechanowski repeatedly lets the reader experience relationships before introducing their compact mathematical expression, while 3Blue1Brown explicitly resists confusing formal symbolism with actual clarity. citeturn25view3turn25view2

**Use one recurring example as home base.**

When an explanation repeatedly changes both the concept *and* the example, the learner must continually rebuild context. A recurring object—a ray hitting a sphere, a tiny neural network, a packet travelling between two machines, a single orbit—lets you go away into abstraction and then return somewhere familiar.

Ciechanowski's interactive articles repeatedly reuse and modify simplified worlds, while Eater's projects maintain a stable artefact that accumulates functionality. citeturn25view3turn25view0

**Distinguish detail from depth.**

Detail means saying more things. Depth means revealing a more generative model.

An explanation can become *deeper* while becoming *shorter* if it replaces twelve isolated facts with one mechanism from which they follow. That is one reason interactive explanations in Victor's sense are so powerful: once the learner can change an assumption and predict consequences, the explanation has become a model rather than a catalogue. citeturn24view3

This is also why I would be ruthless about historical digressions. History belongs when it reveals why a concept had to be invented, shows a productive failure, or sharpens the central question. Otherwise it is probably another video.

**Say explicitly what is being simplified.**

Technical explainers require approximation. The danger is not simplification; it is invisible simplification.

Useful phrases include:

> “This is the two-dimensional version; the three-dimensional case changes the geometry but not the principle.”

> “Real networks have several complications we're deliberately ignoring. None of them changes the mechanism we need here.”

> “This analogy is useful for X, but it breaks down at Y.”

This maintains epistemic trust and prevents the learner from over-generalising the model. It also makes advanced viewers less likely to interpret pedagogical simplification as ignorance.

**Show the reason behind notation.**

Notation becomes much easier to tolerate when every symbol does work.

Instead of saying “Let \(r\) denote the distance”, show that distance keeps changing, that the force depends upon it, and that you now need a compact way to talk about that dependency. Ciechanowski's progression from manipulating distance and mass to displaying the gravitational equation is a good exemplar. citeturn25view3

**Use questions as cognitive operations, not decoration.**

Every question in a script should ideally do one of four jobs:

| Question type | Mental operation |
|---|---|
| “What do you expect?” | Prediction |
| “Why did that happen?” | Causal explanation |
| “What has to be true?” | Derivation |
| “Would this still work if…?” | Transfer |

Those operations align well with research on retrieval, self-explanation and transfer. citeturn26view1turn23search2turn20view1

A script containing twenty questions is not necessarily active. A script containing three questions that genuinely make the viewer stop and think may be.

**Preserve some discovery, but edit reality.**

Matuschak writes approvingly of “working with the garage door up”—showing the process, including problems and incomplete thought. Sebastian Lague's project-based framing also gives viewers a sense of discovery rather than presenting only the polished theorem at the end. citeturn28search18turn14search1

That is valuable because finished technical knowledge can appear inevitable. Seeing one well-chosen wrong turn restores the reason an idea exists.

But pedagogy is not documentary filmmaking. You should compress five uninformative failures into one informative failure. The test is: **does seeing the failure alter the learner's model?** If not, cut it.

**Write the title and opening around the intellectual payoff, not the syllabus label.**

“Understanding Fourier Transforms” names a topic.

“Why every sound can be built from pure tones” names a mystery and implies a transformation in understanding.

The second is usually the stronger narrative promise because it tells the viewer which gap will close. This is a craft inference from the question-, puzzle-, mechanism- and build-driven formats used by the creators above rather than an experimentally established title formula. citeturn24view6turn25view0turn25view3turn24view7

And finally:

**Do not hide the beautiful idea beneath your clever writing.**

Paul Graham's ideal is prose in which the idea seems almost to enter the mind without the reader noticing the language carrying it. For technical narration that principle is even more powerful: the conceptual object should be memorable; the wording that transported it usually does not need to be. citeturn24view2

## How to become unusually good at this

The limiting factor is unlikely to be learning more “presentation tricks”. It is building a better feedback loop between **what you think you taught and what actually changed in the learner**.

Grant Sanderson's observation about expert blindness is the place to start: after you know a solution, it becomes extremely hard to simulate the mind that does not know it. He explicitly notes that people creating mathematics exercises are surprisingly poor judges of their difficulty. citeturn25view2

So before polishing visuals, test the *narrative*.

Give an early script, rough storyboard, audio draft or ugly prototype to people who genuinely resemble the intended audience. Stop at crucial moments and ask them things such as:

> “What do you think is happening?”

> “What do you expect next?”

> “Why did that step work?”

> “What is confusing right now?”

> “Explain what we have established so far without using my wording.”

This is much more revealing than “Was that clear?” because it externalises the viewer's model. The practice is consistent with the emphasis on self-explanation, retrieval and transfer in the experimental literature, while directly countering the expert-blindness problem Sanderson describes. citeturn23search2turn26view1turn25view2

I would build your production process around four increasingly demanding tests.

| Test | Question | Failure means |
|---|---|---|
| **Followability** | Can the viewer say what each step is doing? | Your dependencies or language are unclear |
| **Prediction** | Can they anticipate a nearby consequence? | They are following surface features, not the mechanism |
| **Transfer** | Can they use the idea on a different example? | Your explanation may be too tied to one case |
| **Delayed reconstruction** | What remains after time has passed? | The experience was fluent but not durable |

Retrieval research directly supports caring about delayed reconstruction, while Mayer and Chandler's experiments illustrate why transfer tests are more demanding than simple retention. citeturn26view1turn20view0turn20view1

This also means **audience-retention graphs should be diagnostic instruments, not your objective function**. Guo et al.'s own paper is unusually clear on this point: their millions of viewing sessions measure engagement, and engagement is only a prerequisite for learning. A moment where viewers leave deserves investigation, but a moment everybody watches is not necessarily a moment everybody understands. citeturn26view3

For each video, I would write a one-sentence “model delta” before writing the script:

> **Before:** the viewer thinks/sees/can do ___.

> **After:** the viewer thinks/sees/can do ___.

Then write three transfer questions that someone who truly achieved the “after” state should be able to answer. Only after that would I outline the narrative. This procedure is my recommendation, but its emphasis on transfer and retrieval follows the learning evidence above. citeturn20view1turn26view1

Next, maintain a **misconception inventory** for every domain you explain. Every time a test viewer makes a plausible mistake, do not merely fix the sentence that confused them. Ask whether the mistake reveals a stable intuitive model. If it does, that model may deserve to appear explicitly in the narrative. Muller's experiments are particularly relevant: alternative conceptions can be pedagogically useful material rather than errors to sweep out of view. citeturn27view0

I would also create a companion layer around important videos. Nielsen and Matuschak point towards a medium in which learning continues over time rather than ending when the page—or video—ends, while retrieval research provides experimental justification for revisiting knowledge through active recall. citeturn28search0turn25view5turn26view1

That companion need not be elaborate. For a major explainer, it could contain a few carefully designed questions:

**Immediately afterwards:** reconstruct the central mechanism without replaying the video.

**Later:** answer a prediction question involving a new case.

**Later still:** explain the core idea in one paragraph or sketch the mechanism from memory.

Those timings should be viewed as a general spaced-practice design rather than an experimentally established schedule for your particular videos; Nielsen's work is explicitly concerned with spaced repetition, while Karpicke and Roediger demonstrate the importance of retrieval for later memory. citeturn28search0turn26view1

Most importantly, study your best videos not by asking *which production choices performed well*, but by reconstructing the learner's sequence of thoughts.

For every major beat, write:

| At this moment… | Ask yourself |
|---|---|
| **What does the viewer currently believe?** | What prior model am I relying on? |
| **What do they want to know?** | Is there an open question pulling them forwards? |
| **What new thing am I introducing?** | Is it one conceptual dependency or five? |
| **Why is it needed now?** | Has the problem that motivates it appeared? |
| **What should they now be able to predict?** | Has their model actually gained power? |
| **How will I find out?** | Is there a prompt, transfer case or later test? |

That, in my view, is much closer to professional instructional design than writing a script and then adding graphics.

The final ideal is a video with an almost paradoxical character: **it feels effortless to watch because enormous effort went into deciding where the viewer should have to think**.

The narration is simple in Graham's sense. The conceptual dependencies are reconstructed carefully enough to overcome Sanderson's expert blindness. The abstractions are earned through problems, as in Eater and Karpathy. The learner encounters the phenomenon before the machinery, as in Ciechanowski. Misconceptions become productive conflicts, as in Muller's research. The viewer is periodically required to retrieve or self-explain rather than merely recognise. And the whole thing is short enough to contain one coherent intellectual journey—but no shorter than that journey genuinely requires. citeturn24view2turn25view2turn25view0turn25view1turn25view3turn27view0turn26view1turn23search2

That suggests a compact doctrine for the kind of visual explainers you are aiming to make:

**Start with something worth explaining. Make the viewer predict. Let each abstraction solve a problem they have already felt. Alternate concrete experience with compression. Confront the most plausible wrong model. Give the learner moments to produce the idea themselves. End by making the model work somewhere new. Cut everything that does not strengthen that journey. And choose runtime only after the journey is designed.**

That is the common ground between excellent exposition and serious pedagogy.
