- Start Date: 2025-07-12
- RFC PR: (leave this empty)
- React Issue: (leave this empty)

# Summary

Selectors for Context are an in-demand feature, but with a slightly different interface, can be made much more universally useful.

Following from functional programming techniques, this RFC proposes adding a:

- .map to Contexts, of type: `<T, U>(this: Context<T>, fn: (data: T) => U) => Context<U>`
- .apply to Contexts, of type: `<T, U>(this: Context<T>, fn: Context<(data: T) => U>) => Context<U>`

Every call, without respect to the identity of the given argument (function / context of functions), returns a distinct context.
To be usable within renders, existing user-land techniques like WeakMap can be used to stabilize the identity of the returned context to the identity of the given argument.

# Basic example

If the proposal involves a new or changed API, include a basic code example.
Omit this section if it's not applicable.

```js
// User can set either of these for our API
const ViewportReadonly = createContext(null);
const ViewportWritable = createContext(null);
// Now ViewportRead will grab either, with a preference for ViewportWritable's
const ViewportRead = ViewportReadonly.apply(ViewportWritable.map((writableViewport) => (readableViewport) => writableViewport ?? readabelViewport));

function IntegrateLibrary({ viewport }) {
  return (
    // User can choose which to provide
    <ViewportReadonly.Provider value={viewport}>
      <Library />
    </ViewportReadonly.Provider>
  );
}

function Library() {
  // Library will pick up either, 
  const viewport = useContext(ViewportRead);
}
```


# Motivation

Motivation for deriving Contexts from other Contexts is already well-established.

Motivation for this approach is that these .map and .apply functions are drawn from the Functor and Applicative Functor (respectively) from Haskell.
This style of approach is thoroughly treaded ground, being used with great success in Haskell for many years.

To briefly describe the purpose of .apply in addition to .map (this explanation is no different from the many articles on Functor vs Applicative out in the wild):

- With just .map we can refine one Context into any number of others, which can themselves be refined, creating a tree of ever more specific (and less informative) Contexts.

- With .apply however, we can merge any number of Contexts into one context.
Now we can create a DAG, which is so much more expressive than just a tree.

While I believe this interface is more pleasant in both ergonomics, efficiency and theory than wrapping `useContext`,
what I believe is particularly powerful about it is **its ability to change over time**.
If some code depends on a Context, that Context can later be rewritten in terms of other Contexts.
**The consumption is isolated from the production**: the producers can change without necessitating a change in the consumers.

# Detailed design

This is the bulk of the RFC. Explain the design in enough detail for somebody
familiar with React to understand, and for somebody familiar with the
implementation to implement. This should get into specifics and corner-cases,
and include examples of how the feature is used. Any new terminology should be
defined here.

# Drawbacks

Why should we *not* do this? Please consider:

- implementation cost, both in term of code size and complexity
- whether the proposed feature can be implemented in user space
- the impact on teaching people React
- integration of this feature with other existing and planned features
- cost of migrating existing React applications (is it a breaking change?)

There are tradeoffs to choosing any path. Attempt to identify them here.

# Alternatives

Unfortunately Applicative Functors are fairly more awkward in JavaScript than they are in Haskell (Haskell curries by default, where JS doesn't).

The Applicative Functor interface could be made available in some other forms:

## Via liftA2

Instead of a type like `.apply`'s:

> `<T, U>(this: Context<T>, fn: Context<(data: T) => U>): Context<U>`

a function called something like `.with` could be exposed:

> `<T, U, R>(this: Context<T>, other: Context<U>, combine: (a: T, b: U) => R): Context<R>`

Note this is the same as `.map`'ing `other` by `combine` into `(data: T) => R`, and similarly `.apply` could be defined in terms of `.with`.

**I recommend this approach**.
I think it would be more natural for JavaScript users, and for the typical use of combining two Contexts, only requires creating the output Context,
where `.apply` requires creating the intermediate Context from `.map`'ing `other`.
I didn't want to open the RFC with this, as Applicative Functors are instead usually introduced with `.apply`.

I don't love the name `.with` either.
For reference in Haskell `.with` is called `liftA2`.

## Via sequence

Promises are also valid Applicative Functors. While .then does have all the power of .apply (and then some), there is a particularly Applicative-y util for Promises: `Promise.all` (_handling of TypeScript tuples omitted for brevity_):

> `<T>(promises: Array<Promise<T>>): Promise<Array<T>>`

This pattern is called `sequence` in Haskell. Exposing a function like:

> `<T>(contexts: Array<Context<T>>): Context<Array<T>>`

would give the same power to users as `.apply`.

However, there are a number of reasons this interface isn't that great:
- combining more than two or three Contexts in not that common, for which the Array is needless overhead
- there is no Context namespace, so it would need to be exposed as a plain function from 'react'

However, I figured mentioning `Promise` would be useful for precedence.

# Adoption strategy

If we implement this proposal, how will existing React developers adopt it? Is
this a breaking change? Can we write a codemod? Should we coordinate with
other projects or libraries?

# How we teach this

What names and terminology work best for these concepts and why? How is this
idea best presented? As a continuation of existing React patterns?

Would the acceptance of this proposal mean the React documentation must be
re-organized or altered? Does it change how React is taught to new developers
at any level?

How should this feature be taught to existing React developers?

# Unresolved questions

Optional, but suggested for first drafts. What parts of the design are still
TBD?
