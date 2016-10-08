# 8 simple, battle-tested rules for a scalable CSS architecture

This is the manifest of things I've learned about managing CSS in large, complex web projects during my 10+ years of professional web development. I've been asked about these things enough times that having a document to point to sounded like a good idea.

I've tried to keep the explanations short, but if you're only interested in the *what* and not the *why*, **there's a tl;dr section at the end** which reiterates the rules more compactly.

## Introduction

If you're working with frontend applications, eventually you'll need to style things. And even though the state-of-the-art of frontend applications keeps blazing ahead, CSS is still the only way to style anything on the web (and lately, in some cases, [native applications too](https://facebook.github.io/react-native/)). There's 2 broad categories of styling solutions out there, namely:

* CSS preprocessors, which have been around for ages (such as [SASS](http://sass-lang.com/), and [many](http://postcss.org/) [others](http://lesscss.org/))
* CSS-in-JS libraries, which are a relatively new approach to styling (such as [free-style](https://github.com/blakeembrey/free-style), and [many others](https://github.com/MicheleBertoli/css-in-js))

The choice between the two approaches is a topic for a whole separate article, and as usual, both have their pros and cons. That said, I'll be focusing on the former approach, and if you've chosen to go with the latter, this article will probably be a bit less interesting.

## High level goals

So we're after a robust, scalable CSS architecture. But what properties does that call for, specifically?

* **Component oriented** - The best way to deal with UI complexity is to split the UI into smaller components. If you're using a sensible framework, the JavaScript side of this will come naturally. [React](https://facebook.github.io/react/), for instance, encourages a high level of componentization and compartmentalization. We want a CSS architecture to match.
* **Sandboxed** - Splitting the UI into components won't help our congnitive load if touching the styles of one component can have unwanted and unpredictable effects on another. Fundamental CSS features such as the [cascade](https://developer.mozilla.org/en/docs/Web/Guide/CSS/Getting_started/Cascading_and_inheritance), and a single, global namespace for identifiers actively work against you in this regard. If you're familiar with the [Web Components spec](https://developer.mozilla.org/en-US/docs/Web/Web_Components), think of this as getting the [style isolation benefits of the Shadow DOM](http://www.html5rocks.com/en/tutorials/webcomponents/shadowdom-201/) without having to care about browser support (or whether or not the spec ever gets serious traction).
* **Convenient** - We want all the nice things, and we don't want to work for them. That is, we don't want to make our developer experience any worse by adopting this architecture. If possible, we want to make better.
* **Err on the side of safety** - Somewhat related to the previous point, we want our everything to be *local by default*, and global only as an exception. We engineers are lazy people, and the path of least resistance always needs to point to the correct solution.

## Concrete rules

### 1. Always prefer classes

This is just to get the obvious out of the way.

Do not target ID's (e.g. `#header`), because whenever you think there can be only one instance of something, [on an infinite timescale](https://twitter.com/stedwick/status/525777867146539009), you'll be proven wrong. One past example of this was when we wanted to weed out any data-binding bugs on a large application we were working on. We started two instances of our UI code, side-by-side in the same DOM, both bound to a *shared* instance of our data model. This was to make sure that all changes in the data model were correctly reflected in both UI's. Any components that you might have assumed to always be unique, such as a header bar, no longer are. This is a great benchmark for surfacing other subtle bugs related to assumptions about uniqueness too, by the way. I digress, but the moral of the story is: there's no situation where targeting an ID would be a *better* idea than targeting a class, so let's just not, ever.

Neither should you target elements (e.g. `p`) directly. It's often OK to target elements that *belong to a component* (see below), but on their own, eventually you'll end up having to [undo those styles](http://csswizardry.com/2012/11/code-smells-in-css/) for a component that doesn't want them. Recalling our high level goals, this also goes against just about all of them (component-orientedness, avoiding the cascade like the plague, and being local by default). Setting things like fonts, line-heights and colors (a.k.a [inherited properties](https://developer.mozilla.org/en-US/docs/Web/CSS/inheritance)) on `body` *can* be the exception to this rule if you so choose, but if you're serious about component isolation, it's completely feasible to forgo even these (see below about working with external styles).

So with very few exceptions, your styles should always target a class.

### 2. Co-locate component code

When working on a component, it helps tremendously if everything related to that component -- its JavaScript, styles, tests, documentation, etc -- live very close to each other:

```
ui/
├── layout/
|   ├── Header.js              // component code
|   ├── Header.scss            // component styles
|   ├── Header.spec.js         // component-specific unit tests
|   └── Header.fixtures.json   // any mock data the component tests might need
├── utils/
|   ├── Button.md              // usage documentation for the component
|   ├── Button.js              // ...and so on, you get the idea
|   └── Button.scss
```

When you're working in the code, simply open your project browser, and all other aspects of the component are at your fingertips. There's a natural coupling between the styles and the JavaScript that produces your DOM, and it's a fair bet you'll be touching one soon after touching the other. Think of this as the [locality of reference principle](https://en.wikipedia.org/wiki/Locality_of_reference) for UI components. I, too, used to meticulously maintain separate mirrors of my source tree under `styles/`, `tests/`, `docs/` etc, until I realized that literally the only reason I kept doing that was because that's how I'd always done it.

As an additional benefit, if you're working in the browser, and you spot a component that's misbehaving, you can right-click-Inspect it, and you'll see for instance:

```html
<div class="myapp-Header">...</div>
```

Noting the name of the component -- `Header` in this case -- you switch to your editor, hit the key combo for "Quick open file", start typing "head", and there you go:

TODO: IMAGE

This strict 1:1 mapping between UI components and files is doubly useful if you're new on the team, and don't know the architecture by heart yet: you don't need to, to be able to find the guts of the thing you're supposed to work on.

### 3. Consistent class namespacing

CSS has a single, flat namespace for class names and other identifiers (such as ID's, animation names, etc). Just like in the PHP days of old, the community has dealt with this by simply using longer, structured names, thus emulating namespaces. We'll want to choose a namespacing convention, and stick with it.

For instance, we could use `myapp-Header-link` as a class name:

* `myapp` to first isolate our app from other apps possibly running on the same DOM
* `Header` to isolate the component from other components in the app
* `link` to reserve a local name (within the component's namespace) for our styling purposes

Whatever namespacing convention we choose, we'll want to be consistent about it. It'll be the map by which we navigate the styles of our project.

As a corollary to the previous rule, we will also lose the magic of the simple 1:1 mapping between components and their code unless we also decide that a single file only contains styles belonging to a single namespace. Say we have a login form, that's only used within the `Header` component. On the JavaScript side, it's defined as a helper component within `Header.js`, and not exported anywhere. It might be tempting to declare a class name `myapp-LoginForm`, and sneak that into both `Header.js` and `Header.scss`. But [this is not 'nam, this is engineering, there are rules](https://www.youtube.com/watch?v=WiQmQhA-OrM). Now the new guy on the team might be tasked to fix a small layout issue in the login form, and inspects the element to figure out the culprit component. But there is no `LoginForm.js` nor `LoginForm.scss` to be found, and the new guy has to resort to `grep` or guesswork to find the relevant source files. If the login form warrants a separate namespace, split it into a separate component. Consistency is worth its weight in gold in projects of non-trivial size.

From now on I'll assume the namespacing scheme of `app-Component-class`, which I've personally found to work really well, but you can of course also come up with your own.

### 4. Prevent leaking styles outside the component

So we've established our namespacing conventions, and now want to use them to sandbox our UI components. If every component only uses class names prefixed with their unique namespace, we can be sure that their styles never leak to their neighbors. This is very effective (see below for the caveats), but having to type the namespace over and over again also gets rather tedious.

A robust, yet still tremendously simple solution to this is to wrap the entire style file into a prefix block. Note how we only have to repeat the app and component names once:

```scss
.myapp-Header {
  background: black;
  color: white;
  
  &-link {
    color: blue;
  }

  &-signup {
    border: 1px solid gray;
  }
}
```

The above example is in SASS, but the `&` symbol -- perhaps shockingly -- works the same across all relevant CSS preprocessors ([SASS](http://sass-lang.com/), [PostCSS](https://github.com/postcss/postcss-nested), [LESS](http://lesscss.org/) and [Stylus](http://stylus-lang.com/)). For completeness, this is what the above SASS compiles to:

```css
.myapp-Header {
  background: black;
  color: white;
}

.myapp-Header-link {
  color: blue;
}

.myapp-Header-signup {
  border: 1px solid gray;
}
```

All the usual patterns play well with this, e.g. having different styles for different component states (think [Modifier in BEM terms](http://getbem.com/naming/)):

```scss
.myapp-Header {

  &-signup {
    display: block;
  }

  &-isScrolledDown &-signup {
    display: none;
  }
}
```

Which compiles to:

```css
.myapp-Header-signup {
  display: block;
}

.myapp-Header-isScrolledDown .myapp-Header-signup {
  display: none;
}
```

Even media queries work very conveniently, as long as your preprocessor supports bubbling (SASS, LESS, PostCSS and Stylus all do):

```scss
.myapp-Header {

  &-signup {
    display: block;

    @media (max-width: 500px) {
      display: none;
    }
  }
}
```

Which becomes:

```css
.myapp-Header-signup {
  display: block;
}

@media (max-width: 500px) {
  .myapp-Header-signup {
    display: none;
  }
}
```

TODO: TOOLING HELPS css-ns

### 5. Prevent leaking styles inside the component

Remember when I said prefixing each class name with the component namespace was a "very effective" way of sandboxing styles? Remember when I said there were "caveats"?

Consider the following styles:

```scss
.myapp-Header {
  a {
    color: blue;
  }
}
```

And the following component hierarchy:


```
+-------------------------+
| Header                  |
|                         |
| [home] [blog] [kittens] | <-- these are <a> elements
+-------------------------+
```

We're cool, right? Only the `<a>` elements inside `Header` get [blued](https://www.youtube.com/watch?v=axHe_BVY_9c), because the rule we generate is `.myapp-Header a { color: blue }`. But consider the layout is later changed to:

```
+-----------------------------------------+
| Header                    +-----------+ |
|                           | LoginForm | |
|                           |           | |
| [home] [blog] [kittens]   | [info]    | | <-- these are <a> elements
|                           +-----------+ |
+-----------------------------------------+
```

The selector `.myapp-Header a` also matches the `<a>` element inside `LoginForm`, and we've blown our style isolation. This can be avoided in two ways:

1. Never target element names in stylesheets. If every `<a>` element in `Header` is `<a class="myapp-Header-link">` instead, we'll never have to deal with this issue. Then again, sometimes you have the perfectly semantic markup set up, with the `<article>`s and `<aside>`s and `<td>`s in all the right places, and you don't want to clutter them with extra classes. In that case:
1. Only target things outside your namespace with [the `>` combinator](https://developer.mozilla.org/en-US/docs/Web/CSS/Child_selectors).

Adjusted for the latter approach, our styles can be rewritten as:

```scss
.myapp-Header {
  > a {
    color: blue;
  }
}
```

Which will ensure our isolation also works depth-wise in the component tree, as the generated selector becomes `.myapp-Header > a`.

If this sounds controversial, let me further bring up your blood pressure by claiming that this is also fine:

```scss
.myapp-Header {
  > nav > p > a {
    color: blue;
  }
}
```

We've been trained to avoid selector nesting (including this stronger form with `>`) like the plague, by [many years' worth of credible advice](http://lmgtfy.com/?q=css+nesting+harmful). But why? The cited reasons boil down to these three:

1. Cascading styles will ruin your day, eventually. The more you nest selectors, the higher the chances of accidentally finding an element which matches the selectors of *more than one compoent*. If you've read this far, you know we've already eliminated this possibility (with strict namespace prefixing, and using strong child selectors where needed).
1. Too much specificity will reduce reusability. The styles written for `nav p a` won't be reusable anywhere else except in that very specific environment. But we specifically *never want this anyway*, in fact, we go out of our way to forbid this method of reusability, as it doesn't play well with our goal of components being isolated from each other.
1. Too much specificity will make refactorings harder. This has some basis in reality, in that if you only had `.myapp-Header-link`, you could freely move the `<a>` around in your component's HTML, and the same styles will always apply. Whereas with `> nav > p > a` you will need to update the selector to match the link's new location within the component's HTML. But given how we want to assemble our UI from small, well-isolated components, this argument is also rather moot. Sure, if you had to consider the HTML & CSS of your entire application when doing a refactoring, this might be scary. But you're operating in a small sandbox which has some tens of lines of styles, and knowing that nothing outside that sandbox needs to be considered, you can quickly rewrite the entire thing if your refactoring needs it.

This is a good example of understanding the rules, so you know when to break them. In our architecture, selector nesting is not only OK, it's sometimes the right thing to do. Go crazy with it.

### An aside for the curious: Prevent leaking styles into the component

TODO: NOT POSSIBLE w/o Shadow DOM or iframes

### 6. Respect component boundaries

Exactly like we styled `.myapp-Header > a`, when we nest components, we may need to apply some styles to child components (the Web Component analogy is again perfect, as then there'd truly be no distinction between how `> a` and `> my-custom-a` work). Consider this layout:

```
+---------------------------------+
| Header           +------------+ |
|                  | LoginForm  | |
|                  |            | |
|                  | +--------+ | |
| +--------+       | | Button | | |
| | Button |       | +--------+ | |
| +--------+       +------------+ |
+---------------------------------+
```

We immediately see that styling `.myapp-Header .myapp-Button` would be a bad idea, and we'd obviously want `.myapp-Header > .myapp-Button` instead. But what styles would we ever want to apply to child components?

Note how the `LoginForm` is docked to the right edge of the `Header`. Intuitively, one might style this as:

```scss
.myapp-LoginForm {
  float: right;
}
```

We haven't violated any of our rules, but we've also made the `LoginForm` a lot harder to reuse: if our upcoming home page wants to repeat the `LoginForm`, but without the right-side float, we're out of luck.

The pragmatic solution to this is to (partially) relax our previous rule of only applying styles to the namespace the current file belongs to. Specifically, we want to do this instead:

```scss
.myapp-Header {
  > .myapp-LoginForm {
    float: right;
  }
}
```

But we also don't want to allow breaching the child's sandbox arbitrarily:

```scss
// COUNTER-EXAMPLE; DON'T DO THIS
.myapp-Header {
  > .myapp-LoginForm {
    color: blue;
  }
}
```

Because we'd lose our safety net of knowing our local changes can never have global repercussions. So where do we draw the line between what's OK and what's a no-no?

We want to respect the sandbox *inside* each child component, as we don't want to rely on its implementation details. It's a black box to us. What's *outside* the child component, conversely, is the sandbox of the parent, where it reigns supreme. The distinction between inside and outside emerges quite naturally from one of the most fundamental concepts in CSS: [the box model](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Box_Model/Introduction_to_the_CSS_box_model).

TODO: image: box-model

My analogies are usually terrible, but here goes: just like *being inside a country* means being within its physical *borders*, we establish that a parent can effect styles on its (direct) children only outside the border of the component. That means properties related to positioning and dimensions (e.g. `position`, `margin`, `display`, `width`, `float`, `z-index` etc) are OK, while properties that reach inside the border (e.g. `border` itself, `padding`, `color`, `font` etc) are no-no.

As a corollary, this is also very obviously forbidden:

```scss
// COUNTER-EXAMPLE; DON'T DO THIS
.myapp-Header {
  > .myapp-LoginForm {
    > a { // relying on implementation details of LoginForm ;__;
      color: blue;
    }
  }
}
```

There are a few interesting/boring edge cases, such as:

* `box-shadow` - The effect is clearly rendered outside the border, then again a subtle shadow might be an integral part of the look-and-feel of the component.
* `color`, `font` and other [inherited properties](https://developer.mozilla.org/en-US/docs/Web/CSS/inheritance) - `.myapp-Header > .myapp-LoginForm { color: red }` reaches into the insides of the child component, but on the other hand is functionally equivalent to `.myapp-Header { color: red; }`, which is OK by our other rules.
* `display` - If the child component uses a [Flexbox](https://css-tricks.com/snippets/css/a-guide-to-flexbox/) layout, it's possibly relying on having `display: flex` set on its root element. However, the parent might choose to hide its child by setting `display: none` on it.

The important thing to realize is that in these edge cases, you're not risking thermonuclear war, just introducing a tiny bit of the CSS cascade back into your styles. As with other things that are bad for you, enjoying the cascade *in moderation* is fine. For instance, taking a closer look at the last example, the [specificity contest](https://developer.mozilla.org/en-US/docs/Web/CSS/Specificity) works out exactly like you'd want it to: when the component is visible, `.myapp-LoginForm { display: flex }` is the most specific rule, and takes precedence. When the owner decides to hide it with `.myapp-Header-loginBoxHidden > .myapp-LoginBox { display: none }` that rule is more specific, and wins.

### 7. Integrate external styles loosely

To avoid repetitive work, you sometimes need to share styles between components. To avoid work altogether, you sometimes want to use styles created by others. Both of these should be achieved without creating any unnecessary coupling into the codebase.

As a concrete example, let's consider using some styles from [Bootstrap](http://getbootstrap.com/css/), as it's a perfect example of an annoying framework to work with. Considering everything we've talked about above, with regard to sharing a single global namespace for styles, and collisions being bad, Bootstrap will:

* Export a ton of selectors (as of version 3.3.7, 2481 to be precise) to that namespace, whether you actually use them or not. (Fun aside: IE's up to version 9 can only process 4095 selectors before silently ignoring the rest. I've heard of people spending *days* debugging and wondering what the hell's going on.)
* Use hard-coded class names such as `.btn` and `.table`. Can't imagine those ever being accidentally reused by some other developer or project. :sarcasm:

Regardless, let's say we want to use Bootstrap as a basis for our `Button` component.

Instead of integrating on the HTML side with:

```html
<button class="myapp-Button btn">
```

Consider [extending](http://sass-lang.com/documentation/file.SASS_REFERENCE.html#extend) the class in your styles:

```html
<button class="myapp-Button">
```

```scss
.myapp-Button {
  @extend .btn; // from Bootstrap
}
```

This has the benefit of not giving anyone (including yourself) any ideas about relying on the presence of the ridiculously named `btn` class on the HTML component. The origin on of the styles that `Button` uses is an implementation detail that need not show on the outside at all. As a consequence, should you ever decide to ditch Bootstrap in favor of another framework (or just writing the styles yourself), the change won't be externally visible in any way (except, uhh, the visible changes in how `Button` *looks*).

The same principle applies to your own helper classes, and there you'll have the option of using more sensible class names:

```scss
.myapp-Button {
  @extend .myapp-utils-button; // defined elsewhere in your project
}
```

Or [forgoing emitting the class](http://sass-lang.com/documentation/file.SASS_REFERENCE.html#placeholder_selectors_) altogether (LESS has a similar mechanism):

```scss
.myapp-Button {
  @extend %myapp-utils-button; // defined elsewhere in your project
}
```

Finally, all CSS preprocessors support the concept of [mixins](http://sass-lang.com/documentation/file.SASS_REFERENCE.html#mixins), which allow you to more or less whatever you want:

```scss
.myapp-Button {
  @include myapp-generateCoolButton($padding: 15px, $withExplosions: true);
}
```

It should be noted that when dealing with more civilized style frameworks (such as [Bourbon](http://bourbon.io/) or [Foundation](http://foundation.zurb.com/)), they'll in fact be doing just this: declaring a bunch of mixins for you to use where they're needed, and not emitting any styles on their own. [Neat](http://neat.bourbon.io/).

### 8. Know the rules, and when to break them

Finally, as mentioned before, when you understand the rules you've laid out (or adopted from a stranger on the Internet), you can make exceptions that make sense to you. For instance, if you feel that there's added value in using a helper class directly, you can do so:

```html
<button class="myapp-Button myapp-utils-button">
```

That added value might be -- for instance -- that your test framework can then be more clever in automatically figuring out what things act as buttons, and can be clicked.

Or you might decide that it's OK to break component isolation when the breach is tiny, and the additional work from splitting components would be too great. While I'll want to remind you that it's a slippery slope, and that consistency is king, bla bla... as long as your team is in agreement, and you get stuff done, you're doing the right thing.

## License

[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/)
