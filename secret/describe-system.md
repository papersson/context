# Describe a system, anonymized

I'm studying testing and verification techniques (stateful property testing,
deterministic simulation, fault injection, model checking), and I want to
design exercises based on the shape of a real system instead of toy examples.
Please look through the services and repos you can reach here and write me a
description of how this system fits together, in a form that is safe to share
publicly. The shape matters to me, not the specifics: what talks
to what, what state lives where, what each part assumes, and where timing
and failure come in.

## Anonymization comes first

This text will be shared outside this environment, so treat anonymization as a hard
requirement. No single detail should be identifying, and neither should any
combination of details (a domain plus a team size plus an unusual
technology choice can identify an organization even with every name removed).

- Give every service, team, queue, topic and database a generic role name,
  such as "ingest service", "billing processor" or "events topic A". Don't
  write internal names, product names, customer or partner names, the
  organization's name, hostnames, URLs, repo names, ticket ids, people or dates.
- Describe the business domain only in generic terms, such as "a record is
  created, enriched by two other services, then published". If the domain
  itself would be identifying, swap in an analogous neutral domain, and say
  that you did.
- Name generic infrastructure only by category: "a managed message queue",
  "a relational database", "a scheduled batch job". Leave out vendor and
  product names unless one is so common that it identifies nobody, and even
  then prefer the category.
- Give numbers as rough orders of magnitude ("tens of services", "minutes
  to hours of delay"), never exact counts, volumes, money amounts or SLAs.
- Don't paste code, schemas, config, log lines or real payloads. Paraphrase
  them structurally instead, e.g. "the message carries an entity id and a
  status, but not a version".
- Describe incidents in general terms, without dates, customer impact
  figures or anything that matches a public postmortem or status page.

## What to describe

1. The map, kept rough: the main services and what each one owns (data or a
   responsibility), whether this codebase owns it or another team or a vendor does, and
   how they communicate: synchronous calls, fire-and-forget messages,
   events, shared databases, scheduled or batch jobs. A plain-text diagram
   is welcome.

2. One or two flows in depth. Pick the flows that cross the most services
   and matter most. Go step by step: who calls or messages whom, what state
   changes where, and what "success" means once the flow is over. For each
   step, note whether it's synchronous or asynchronous, whether retrying it
   is safe (idempotent), and what happens on a timeout, an error, or a
   timeout after the other side already did its part.

3. The asynchronous fire-and-forget calls, in the most depth. My biggest
   pain is messages that queue up and get processed much later than
   expected, sometimes hours later. For each important one, describe:
   - who sends it, who consumes it, and what triggers it;
   - the delivery guarantees as far as you can tell: can it be duplicated,
     reordered, delayed, or lost, and is there a retry or a dead-letter
     path;
   - what the consumer assumes is still true when the message arrives (the
     entity still exists, its status hasn't changed, the config is the
     same, it's the newest message about that entity), and what happens
     when that assumption is false;
   - what carries ordering or freshness information (a version, a
     timestamp, a sequence number), or that nothing does;
   - how anyone finds out when a message is late, stuck or processed twice.

4. The invariants, as you understand them: statements that should always be
   true across services, such as "every X eventually gets a Y", "a Y never
   exists without its X", or "two services agree on the status of X once
   things are quiet". Note which ones the code actually enforces and which
   ones are only hoped for.

5. How it's tested today, at each level (unit, service with fakes,
   integration, end to end, in production), and what's faked at each one.
   Note especially whether any test controls time or message delivery.

6. Two or three bugs or incidents that escaped to production, or that people
   worry about, in two or three generalized sentences each: what happened,
   why it got past the tests, and which assumption was wrong.

## How to write it

Write plain prose and short lists. Aim for two pages at most, and keep the
depth for sections 2 and 3. Mark how you know each claim: [code] if you saw
it in code or config, [docs] if it came from documentation, [inferred] if
you're guessing. Where the code and the docs disagree, say so; that
disagreement is often the interesting part. If you can't tell something,
say you can't instead of guessing.

The most useful single thing you can give me: for one real late or
duplicated message, a short timeline in generic terms. What the sender knew
when it sent the message, what changed in the meantime, and what the
consumer did when the message finally arrived.

## Before you hand it back

Reread the whole text as an outsider who knows the industry and tries to
figure out which organization, team or product this is. Remove or blur anything
that would help them, including combinations of details. Then list, in
one line each, which kinds of things you removed or generalized (for
example "replaced the domain with an analogous one", "dropped vendor
names"), without repeating the removed details themselves.

It's done when:
- every service and message has a generic role name, with no internal names,
  vendors, exact numbers, code or dates anywhere;
- each important async message has its consumer's assumptions written out
  and marked as [code], [docs] or [inferred];
- at least one invariant spans more than one service;
- someone who has never seen our system could sketch the depth flow as a
  sequence diagram from your text alone, including where the late message
  lands.
