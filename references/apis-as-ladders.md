# APIs as ladders

*January 2022*

Developers have opinions on what a good API[^1] is, but rarely have a shared vocabulary to describe what makes them good. This essay puts forward one set of considerations (out of many) that we started using at Stripe in 2019 to discuss API design[^2]. You can find some of these concepts interspersed in this excellent blogpost by Michelle Bu.

The hard part of an API is not to use it, but to learn it. After learning how the API works, typing out the commands is not hard. But the first time you encounter an API, a million questions pop into your mind: What is this object for? Is there a parameter for that? Can the API do X? Learning requires effort, but the more the developer learns, the more problems they can solve. We can imagine the developers climbing a ladder where, for every step they take up the "learning ladder"[^3], the more problems they can solve.

Developers don't want to learn your API, they want to solve a problem and move on. As such, you should try to minimize the amount of learning they have to do[^4]. In a simple domain, you can have an ideal ladder with only one step. Once developers learn that step, the API helps them solve every problem in its domain. The worst ladder makes them learn a lot upfront but then helps them solve comparatively little.

We'll look at three groups of people that are at different stages of the ladder:

- **Beginners** are trying to learn enough to use the API for the first time.
- **Novices** are trying to learn enough to solve their second problem[^5].
- **Experts** are using the API to solve very complicated problems.

And for each of them, different properties of the ladder matter:

- The more upfront learning beginners need, the more likely they'll get discouraged. This is represented by the **height of the first step** in the ladder.
- If a novice learns a little more and unlocks a lot of new solutions, they'll be encouraged to learn further. This is represented by the **steepness** of the ladder.
- Experts care about the total number of problems they can solve with the API. This is represented by **how far to the right** the ladder goes.

We are going to evaluate how APIs help at different stages of the ladder:

- In order to get started, beginners need an API to be **convenient**.
- In order to take the next step, novices need the API to be **gradual**.
- In order to solve most problems, experts need the API to be **flexible**.

But for reasons that will become clear later, you should design your APIs in the opposite order: make them **flexible first, gradual second, and convenient third**.

## Flexible APIs can solve a lot of problems

I'll use the Stripe Billing API as an example because it has a rich data model, meaning it has many objects and allows for many combinations of them. This is a simplified view:

> Solid lines represent necessary dependencies / relationships, dotted lines represent optional relationships

It is easier to discuss the problems of poorly-abstracted APIs than to define exactly what makes an API abstracted. Here are some failure modes:

- **Hidden relationships and dependencies**
  - Imagine if when you changed `Price` from `digital_good` to `physical_good`, the `Customer` could no longer pay from their `CustomerBalance`.
- **Overly simplistic modeling**
  - Imagine if a `Customer` could only have one address.
- **Unnecessary or overly restrictive coupling**
  - Imagine if a `Customer` always needed a shipping address to pay for `Subscriptions`, even digital ones.

But when is an API flexible? The tautological definition is "when it lets you do what you want". I don't have a recipe[^6] on how to make a flexible API but I've found that APIs that follow the closure property tend to be flexible. Loosely, an API holds the closure property when every operation returns a data type that can be fed into other operations. This means that different operations in your API compose with each other. For example, most string-manipulation libraries take in strings and return strings:

```js
"ape".replace("e", "i").uppercase()
```

The closure property makes it easy to combine multiple operations to get the desired result. This is only aspirational though—very few domains behave as cleanly as strings do in the example.

If your API is not flexible, you'll find yourself saying "I'm sorry, I know our API should be able to do that but for reasons that you don't care about and are within our control, we can't".

## Gradual APIs are easy to learn

When a developer wants to add one piece of functionality, how many new concepts do they have to learn? And when they have to add another piece of functionality, what else do they have to learn? And after that? Ideally, they can gradually learn and implement what they need.

When choosing a minimal set of concepts to teach first, you can only go as far as the API's structure will let you. As you try to remove concepts, you can only remove concepts that are optional dependencies, not hard dependencies. For example, in Stripe Billing's current model, you can't have a `Subscription` without a `Customer`:

> The wider the "breadth" of concepts to take at once, the more confused newcomers will be.

To create the minimal Billing integration, you need the following:

```bash
curl https://api.stripe.com/v1/products \
  -d name = "Netflix" \
  -d name = "Stream your favorite movies and shows"

curl https://api.stripe.com/v1/prices \
  -d product = "prod_123" \
  -d unit_amount = 1000 \
  -d currency = "usd" \
  -d "recurring[interval]" = "month"

curl https://api.stripe.com/v1/customers \
  -d email = "jenny.rosen@gmail.com"

curl https://api.stripe.com/v1/subscriptions \
  -d "items[][price]" = "price_123" \
  -d customer = "cus_123" \
  -d default_payment_method = "pm_visa"
```

It'd be better if we could use the API with fewer concepts, ideally introducing one at a time[^7]. Let's try to change the Billing API so that we can teach gradually. First, notice that `Product` and `Price` are there to generate an amount for the `Subscription` to charge. If we let the developer pass an amount directly, then we could make `Product` and `Price` optional.

We now only have three objects to learn upfront: `Customer`, `Subscriptions`, `Invoices`. And if the developer needs the additional functionality that `Product` and `Price` bring in, they can learn those concepts later.

> By hiding `Product` and `Price`, the initial "breadth" of concepts to learn is much smaller.

The code is now:

```bash
curl https://api.stripe.com/v1/customers \
  -d email = "jenny.rosen@gmail.com"

curl https://api.stripe.com/v1/subscriptions \
  -d "items[][description]" = "Netflix" \
  -d "items[][unit_amount]" = 1000 \
  -d "items[][currency]" = "usd" \
  -d "items[][recurring][interval]" = "month" \
  -d customer = "cus_123" \
  -d default_payment_method = "pm_visa"
```

The example illustrates how relaxing constraints between the concepts makes it easier to gradually learn them and implement them[^8]. You can see now why it pays off to invest in the API's flexibility first.

Having separate layers that the developer can gradually learn is often a good recipe. In the example, we split one layer of five concepts into two layers with fewer concepts each:

- **Layer 1** uses `Subscriptions` and `Invoices` to charge `Customers`.
- **Layer 2** uses `Products` and `Prices` to track what was sold.

This is not always easy to do. A subtle problem arises when adding Layer 2 changes the semantics of Layer 1. In the previous example, we are starting with a `Subscription` with a simple amount, and then adding a `Product` and a `Price`. The `Subscription`'s amount can be updated after the `Subscription` was created. But what if when using `Prices`, the `Price`, and thus the final amount, couldn't be updated after creation? This would be a total surprise to the developer who climbed this ladder:

1. I can update the `Subscription`'s amount.
2. I can replace amount with `Product` and `Price` to get additional functionality.
3. But I can't update `Product` and `Price` to change the `Subscription`'s calculated amount? This contradicts what I learned in step 1!

When layering, keep the semantics of each layer stable and if you can't, question if you should be layering in the first place.

## Convenient APIs are easy to get started with

For the most prototypical use-case, how easy is it for beginners to get started?

Consider `create-react-app`. It requires no learning to get started: just install and run `npm start`, and now you have a web server and a single page app running! But what if you want something slightly different? To go one step further, one must "eject" with `npm run eject`.

Suddenly, you learn that there is a tool called Webpack, another called Babel, another called ESLint, and they all have a thousand options for you to handle. `create-react-app` is conveniently-packaged but it is terrible at revealing the next layer. Despite this, `create-react-app` is still hugely popular. Convenience is king.

In the previous section, we saw we needed five concepts to get started with Billing, each concept doing a different job. This is good because it shows the developer where the different functionality is. But it is also many new concepts to learn upfront. Could we make Billing more convenient to use?

We could create an SDK function to do multiple API calls at once:

```ruby
subscription = Stripe::Subscription::Simple.create(
  product_name: "Netflix",
  recurring_interval: "monthly",
  unit_price: 1000,
  currency: "usd",
  payment_method: "pm_visa",
  email: "jenny.rosen@gmail.com",
)
```

which in turn creates the following objects under the hood:

> `SimpleSubscription` takes a few arguments and creates many objects.

To make an API convenient you can always "package it": draw an arbitrarily large box around the concepts you need for a given use-case and make a new concept that packages them all into one. But don't overdo it. The problems start when the developer needs to peek inside the box: the more you stuff inside the box, the more concepts they'll have to learn at once when they open it, and the more disoriented they will be. In the example above, the developer will be blindsided when they suddenly have to learn about `Invoices`, a concept they haven't seen before, to track something as basic as "did I get paid?".

Another way of making an API more convenient is through defaults. Enumerate the top 5 use cases and see if you can cover them with a set of sensible defaults.

But the most important thing is to have good documentation with a Getting Started guide. Showing developers one or two self-explanatory code snippets that just work after they copy-paste them is the ultimate sign of convenience. In my opinion, much of Stripe's success can be explained by this code snippet:

```ruby
charge = Stripe::Charge.create(
  amount: 1000,
  currency: "usd",
  source: "tok_visa",
)
```

## Examples outside of APIs

These concepts also apply to other products that require considerable learning and have lots of depth:

### Excel

Excel has a pretty smooth ladder, except in that last 10% of cases when you just can't do what you want and you need to learn VisualBasic (Microsoft's scripting language). This either constrains the problems that a user can use Excel for, or it requires the user to go through a huge hurdle to get to the next step. Without any data, my sense is that for every one user that learned VisualBasic, there are hundreds that didn't but would've benefited from it.

### Vim

Vim is very hard to learn. Do you want to type a letter? Press `i` first. Do you want to quit? Figure it out. The steepness of that initial step is criminal. Yet the underlying cursor and text manipulation are so good that once you understand them you can't go back to any other text editor. So, if you put the initial effort upfront, you get huge rewards at the end. Very few get over that initial hurdle.

This was inspired by this excellent post.

## What developers actually want

Earlier, I recommended working on flexibility first, teaching the API gradually second, and convenience third. But that is the opposite of what the developer market empirically cares about:

- If the API is not convenient → beginners don't adopt the API.
- If the API is not gradual → novices find it complicated and don't become experts.
- If the API is not flexible → experts eventually "eject" to something else to solve their problems.

While having a flexible API makes the other two steps easier, the market doesn't care about flexibility at first. It is tempting to start by making the API convenient and ignore its flexibility. When targeting beginners, convenience has the most immediate impact on adoption but starting with it leads to a dead end.

But when targeting developers with an existing project (like a large enterprise) convenience is less important. There, the developer will spend lots of time learning and scoping out an integration. What matters is that they can use your API in the first place. More often than not, your API has one restriction that makes it impossible for them to adopt it. The more flexible your API, the more likely it is to satisfy the project's constraints.

Hopefully these concepts will help you improve your API designs. But please don't take them as the end-all-be-all: there are many other considerations that this essay doesn't cover and could be much more important in your domain.

---

*Thanks to Devon Zuegel, Alexander Thiemann, Jonathan Blow, and Michelle Bu for reading early drafts and providing valuable feedback.*

## Footnotes

[^1]: In this essay, I use "API" (Application Programming Interfaces) to refer to any tool that is used by a program while the program is running. Databases like Mongo and PostgreSQL, services like Twilio and Stripe, and date and string libraries are all APIs.

[^2]: These principles weren't in place when most Stripe APIs were written, so they don't explain Stripe's success as an API company.

[^3]: Those who already know about learning curves probably noticed that the ladders of this essay are learning curves with their axes inverted. An earlier version of this essay used learning curves but the ladder metaphor felt more natural to early readers.

[^4]: As an API designer, it is easy to forget how little people care about learning your particular API. You spend all this time thinking about the domain and understanding its structure that you naturally expect everybody to understand it (or want to understand it) too.

[^5]: This can also be a beginner trying to add something to a project written by someone else.

[^6]: *Simple Made Easy* by Rich Hickey is the best I've found on "how to make APIs flexible?"

[^7]: *The Witness* is the best example I've seen of teaching one concept at a time.

[^8]: This talk by Casey Muratori focuses on how to make APIs gradual by minimizing "integration discontinuities": moments where the requirements expand slightly, but the effort required to learn and use the API is disproportionally big.

Source: https://blog.sbensu.com/posts/apis-as-ladders/
