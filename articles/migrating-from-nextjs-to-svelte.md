# Migrating form NextJS to Svelte-Kit

**Author: Daniel Einars**

**Date Published: 18.04.2023**

### 1. Why, What & Who


**Why:**

I've decided to migrate my [website](https://dle.dev) form NextJS to Svelte. My brother has been going on about Svelte for some time and with the [release of SvelteKit v1.0](https://svelte.dev/blog/announcing-sveltekit-1.0) back in December 2022, I finally thought that it's time to take the plunge.

**What:**

I'll be showing how I migrated the intro animation on my home page over to svelte. Beyond that I will highligth the features I was most impressed by when using svelte. These are
  * The included component transitions
  * The included page transitions
  * The simplicity of global state
  * The handling of Environment variables
  * The awesomeness that is data-fetching, including streaming promises

Finally, I'll compare PageSpeed Score with NextJS.

**Who:**

This article is intended for readers who are curious about Svelte, how much work a migration would take, pitfalls I've worked around and anything else that might pop up along the way. With that being said, this article assumes two things

1. You have heard of [NextJS](https://nextjs.org/) and it's [ISR](https://nextjs.org/docs/basic-features/data-fetching/incremental-static-regeneration) feature
2. You know some [React](https://react.dev/)
3. Knowledge of Tailwind is useful *but not required*

It helps if you've dabbled in [react-spring](https://www.react-spring.dev/) and written data-fetching code in React.


**TL;DR:**

Once all was said and done, Svelte saved me a _ton_ of animation code. Although it's currently still missing the ISR feature for non-vercel platforms (WIP), the tools provided by Svelte such as component animations and page transitions, the included state management the excellent documentation made it a breeze to pick up compared to learning NextJS. I was a bit miffed about not being able to keep multiple components in one file, but in hindsight this has kept me from mixing atoms with organisms. The `Reactive` syntax took some getting used to and I'm still not sure 100% of the time if and why I need a reactive block, but this will come with experience, similar to learning about using react hooks correctly.

PageSpeed Score: Svelte Wins!


### 2. Migrating simple components (atoms)

*keep all examples relevant to the intro animation*

*record short gif of what I'm going to be doing (side by side with result?)*
### 3. Migrating composed components (collection of atoms, some internal state)
*keep all examples relevant to the intro animation*

*record short gif of what I'm going to be doing (side by side with result?), split up into multiple steps if necessary.*
### 4. Migrating complex components (layouts, pages, error) 
*keep all examples relevant to the intro animation*

*record short gif of what I'm going to be doing (side by side with result?), split up into multiple steps if necessary.*
### 5. Conclusion

  * Developer Experience (win svelte, minus points for intellij and lack of LSP integration)
  * Ease of use of animations (win svelte)
  * ISR (currently win NextJS)
  * Performance Comparison (win svelte)

