<!--
.. title: The Yesod Transformer Library 
.. slug: yesod-transformer-library-announced
.. date: 2020-12-28 19:01:36 UTC+01:00
.. tags: 
.. category: 
.. link: 
.. description: 
.. type: text
-->

# The Yesod Transformer Library

This post assumes a familiarity with monad transformers.

As part of my job, I've been spending lots of time in the [Yesod][0] ecosystem - `yesod-core` (and `yesod-*`), `persistent`, and the `shakespeare` templating languages. These libraries are particularly easy to contribute back to: the efforts of Michael Snoyman and Matt Parsons, as well as everyone else who puts hard work into those communities, mean that interaction is pleasant, considerate, and meaningful. Their ambitious nature also means that small contributions are relatively easy to make - just fill any utility hole in the vast APIs that already exist.

However, Yesod's design does come with some pain points. Notably, `yesod-core` is in a complicated relationship with [monad transformers][1]: it used to support them, and now it doesn't.

## Monad transformers in Yesod

Yesod is based around two monads: `HandlerFor site`, which is essentially a request handler for some *foundation site* type `site`; and `WidgetFor site`, which is a specialised handler for building webpages. Specifically, `WidgetFor` lets you use do-notation to build a page imperitavely - you can write to the page piece by piece, appending HTML, CSS, and JS directly to the head or body.

Yesod *also* has two monad classes: `MonadHandler`, which can be implemented by any monad that can run a `HandlerFor`; and `MonadWidget`, which has the same property for `WidgetFor`. Each base monad type implements its respective monad class. But these classes also have lifting instance for loads of monad transformers. For example, if `MonadHandler m`, then `MonadHandler (ReaderT r m)`.

[`HandlerFor` and `WidgetFor` used to be transformers][2]. `yesod-core` still contains some references to `WidgetT` and `HandlerT`, even in non-deprecated APIs. But some years ago, they were changed to be just monads, and the result is an interesting limbo - some code cares about `mtl`-style `MonadWidget`s and `MonadHandler`s, and other code just uses the base monad.

This change wasn't baseless - Snoyman has written in the past about how overused he thinks full-application monad stacks are, and the value of the ["ReaderT" pattern][4], where monad state is limited to a reader type and (typically) the `IO` monad. In fact, `HandlerFor` and `WidgetFor` are exactly that - `IO` monads which can read the foundation site.

### What's so bad about that?

For the most part, nothing. Concrete types can even add value themselves - error messages from an unexpected type are often much more readable than instance resolution failures in `mtl`-style code.

But the pain points I encountered came from two particular places where `yesod-core` interacted with code for my application:

  * when running the website, and
  * when trying to make a custom widget

#### Running the website

Yesod's design is very tied to the idea of a foundation site - as can be best seen in [the `Yesod` class][5]. `Yesod` is implemented for foundation site types - it describes how the corresponding site performs certain actions. For an example, take the `errorHandler` method:
```haskell
errorHandler :: ErrorResponse -> HandlerFor site TypedContent
```
`errorHandler` defines how the server takes some kind of error and returns it to the client as content. Crucially, that response *must* be a `HandlerFor` - it cannot be a generalized `MonadHandler`, and it must be runnable (presumably because, otherwise Yesod could not run it). Similar constraints, using `WidgetFor` and `HandlerFor`, pepper the whole class.

These choices are not a consequence of the dropping of support for monad transformers - the same design existed well before then. But they do have the same consequences - that `site` must do everything. In the above example, there is the constraint (from the type) that the `site`, as well as some internal Yesod `HandlerData`, must be enough to decide how to give an error response. You cannot return a `ReaderT r (... TypedContent)`, even if you need some additional context `r` not found in the `site`. Similarly, you cannot use a `WriterT w (... TypedContent)` to track some error metrics `w`, without having to store them in the `site`.

This is not an obstacle most of the time - after all, it's easy to put stuff in the site. But I was interested in making a handler that would change the output of logging on my website, without needing to change all of my code. In `mtl` style, this would have been a monad transformer, to modify the monad stack for my handlers. In Yesod, that change isn't so easy.

#### Making custom widgets

Widgets, like handlers, are required to be `WidgetFor`s at the boundary where your application code touches Yesod's. This is more sensible - rarely do you want hidden outside context on all your widgets. But that doesn't preclude transformers appearing *inside* some widget code: like running widgets which all need the same DB data in the same `ReaderT`.

For a more concrete example, consider a problem I had last week: when visualising some submitted data to the user who submitted it, some of it would be obviously erroneous - data could be missing, values would be zero where non-zero was expected, etc. The goal was to let the user know that these anomalies existed in a little report table, *and* have that table link to each anomaly in the view.

The code could have been ugly, because this is a clear case of mixed responsibilities - I needed to spot and track errors in the model, but report them in a way that's directly woven into the view. But there was a more elegant possibility: the view could report anomalies as they were displayed. Once the view was finished rendering, the anomaly report would already be complete. Since the view itself did the reporting, it could even generate a unique ID per report, and make that ID link to the displayed anomaly - when the report was rendered, each link's target would already be in place on the page.

The resulting approach called for a custom widget type; one which could display basic widgets (`WidgetFor site`, which did not report anomalies) alongside their error-reporting cousins. This custom type was a `WriterT` transformer on `WidgetFor` - and the writer type was the anomaly report itself.

But this transformer, too, ran into problems. Yesod's `whamlet` quasiquoter lets you include HTML snippets as widgets in your Haskell code - and those snippets can themselves embed other widgets. For example, the snippet:
```haskell
[whamlet|
  <h1>A foreboding title
    ^{anInnerWidget}
  |]
```
defines a widget whose layout is the `<h1>` header, followed by the layout of the `anInnerWidget` widget. What's the type of `anInnerWidget`? Yesod requires that it is `WidgetFor`, *even if* the outer widget type is not! In other words, custom widgets cannot use this syntax to nest each other, even though Yesod's `WidgetFor`s can.

## Custom foundation sites

Of course, Yesod still affords you plenty of control over how your handlers and widgets will run. Specifically, to allow for configuration options, DB connections, and other runtime values that would be relevant to handlers and widgets, Yesod allows your foundation site to be absolutely anything: the only thing it has to do is implement the relevant classes, like `Yesod`.

This flexibility on the foundation site type is much more like the polymorphism that `mtl` was written to take advantage of. And since Yesod code often restricts us to using `HandlerFor site` and `WidgetFor site`, there's a neat alternative to monad transformers for Yesod - **site transformers**.

### Site transformers

A site transformer is largely what it sounds like - a wrapper for a `site` type which, like a monad transformer augments a monad, augments the underlying `site`. The transformed type still needs to implement relevant classes, like `Yesod` - but it is free to do those by delegating to the base class, or defining its own behaviour, as it wants.

The power of site transformers is: while your handlers and widgets are required to depend only on your `site` type, and no additional context, they can depend on the site type *as much as they want*. Morever, because access to the Yesod internals lets you change the site type for just a snippet of a handler, that snippet can depend on a different site type to the rest of the code. When running any snippet, you then only have to provide the additional context to use the modified site type.

To talk about this in more concrete terms, consider this example:
```haskell
data ReaderSite r site = ReaderSite r site
```
This `ReaderSite` is a site transformation that adds additional reader context to handlers and widgets. It doesn't look exactly like a `ReaderT`, because the `site` itself is already part of a reader (remember, handlers are "Reader IO"s). Any handler with site type `ReaderSite r site` can read the transformed site type - so it can access the value with type `r`. This means that, by transforming a `site` to a `ReaderSite r site`, we can *add reader context* to a handler - without needing `ReaderT`.

For example, we can define:
```haskell
ask :: HandlerFor (ReaderSite r site) r
```
which readas the reader context from the site. To *run* a `ReaderSite`, we have to supply the reader context - just like for `ReaderT`:
```haskell
runReaderSite :: r -> HandlerFor (ReaderSite r site) a -> HandlerFor site a
```

In fact, this concept goes all the way to essentially reproducing an `mtl`-style design in Yesod - by defining, using, and running site transformers, it's possible to do many of the things `mtl` would otherwise let you do: read additional data, write to additional outputs, run parts of your monad code with a modified state, etc. All the while, your code remains compatible with `Yesod` itself, because the foundation site type is yours to play with.

## Announcing YTL

That gets me to my main point: based on the above concepts, I've written a new utility library for writing Yesod code: [`ytl`, the Yesod Transformer Library][6]. Like `mtl` for monads, `ytl` describes site transformers: how to define them;  how to lift handlers and widgets to transformed variants; and how to run transformed handlers and widgets with the underlying site.

The library itself is already being put into use for [`yesod-katip`, a logging bridge I wrote between Yesod webservers and Katip scribes][7]. But the power of the library is in its extensibility - it provides the tools to define new transformers easily, and integrate them into existing code automatically. Hopefully, others will find that `ytl` is just what they've been looking for.

 [0]: https://www.yesodweb.com/
 [1]: https://hackage.haskell.org/package/mtl
 [2]: https://github.com/yesodweb/yesod/commit/47ee7384ea123135d090c4e931657cb11c583b94
 [4]: https://www.fpcomplete.com/blog/2017/06/readert-design-pattern/
 [5]: https://hackage.haskell.org/package/yesod-core-1.6.18.8/docs/Yesod-Core.html#t:Yesod
 [6]: https://github.com/ivanbakel/ytl
 [7]: https://github.com/ivanbakel/yesod-katip
