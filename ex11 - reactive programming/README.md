## Description
### Reactiveness
I've found modelling evolving state as composition of event streams to be intuitive and easy to manage. I do not yet have enough experience with this style of programming to know if it's maintainable or if it scales well.

In this example I'm creating a stream of states tracking which keyboard buttons are pressed down with no successive repeats of the same state. We'll learn some interesting things about PureScript and this style of programming along the way.

We start by creating a pair of an event stream and a function that push events onto that stream. To do this we use `create :: forall a. Effect { event :: Event a, push :: a -> Effect Unit }` from the [FRP.Event](https://pursuit.purescript.org/packages/purescript-event/1.2.4/docs/FRP.Event) module.
```purescript
main :: Effect Unit
main = do
    { event, push } <- create

    pure unit
```
*Note that we haven't told PureScript which types we want to work with. PureScript has a Hindley-Milner type system that infer the types by how we use `event` or `push`. We get the terseness of dynamic languages like Python but high reliability.*

*Further you should notice that this is very similar to [Redux](https://redux.js.org/). Actually, you can implement Redux in PureScript in one or two lines of code.*

Now we need to push keyboard presses into `event`. In order to track if a button is pressed or not we need to track when a key was pressed down and also when it's released. That way we can deduce whether a button is being pressed. I could create a sum type like `data KeyEvent a = Down a | Up a` which might be good for a larger project because it's more apparent what its purpose is but the [Either](https://pursuit.purescript.org/packages/purescript-either/4.1.1/docs/Data.Either#t:Either) type has a lot of useful utility functions that I want to use. We'll use `Left` for presses and `Right` for releses.
```purescript
main :: Effect Unit
main = do
    { event, push } <- create
    
    onKeyDownEvent <- eventListener $ onKey (Right >>> push)
    onKeyUpEvent   <- eventListener $ onKey (Left  >>> push)

    pure unit

    where
        onKey :: (String -> Effect Unit) -> Event -> Effect Unit
        onKey dispatch e = fromEvent e <#> key # maybe (pure unit) dispatch
```
A lot of stuff at once here but it's not as bas as it seems. The `onKey :: (String -> Effect Unit) -> Event -> Effect Unit` is a little juicy but splitting it up seems to make this example cluttered. `onKey` takes the `push` function and maps the [Event](https://pursuit.purescript.org/packages/purescript-web-events/2.0.1/docs/Web.Event.Internal.Types#t:Event) to the keyboard button value and invokes `push` with it.

Next we wire up the event listeners to be fed the key press events with `addEventListener :: EventType -> EventListener -> Boolean -> EventTarget -> Effect Unit`.
```purescript
window <#> toEventTarget >>= addEventListener keydown onKeyDownEvent false
window <#> toEventTarget >>= addEventListener keyup   onKeyUpEvent   false
```
Now we can use `subscribe :: forall r a. Event a -> (a -> Effect r) -> Effect (Effect Unit)` to do stuff based on what's in our event stream. The `Effect (Effect Unit)` part of the signature confused me at first. Why not just use `join :: forall a m. Bind m => m (m a) -> m a` to simplifify it to `Effect Unit`? But it turns out that the inner `Effect Unit` is actually an effect that, when invoked, reverts the subscription to the event. I don't care about that in this demo so I'm going to ignore it.
```purescript
subscribe event logShow <#> const unit
```
*`const :: forall a b. a -> b -> a` ignores the second argument and always returns the first.*
The same result could be achieved with the `do` notation but I'm such a massive clever clog I want to "rEdUcE lInEs Of CoDe!!!1!½" like a complete junior n00b beginner.

Running this application will log to the browser console `(Right "a")` when you press the `a` key, for example, and `(Left "a")` when releasing the key. If you hold a button down we get a repeat of that event. That might seem fine but what if you press another button at the same time? Both are fireing one after the other? So how do you know when reacting to an event that the other button is pressed? Or hold one key for a while and then hold another key (thereby holding two down at the same time)? The first key stops firing. Things get a little funky and difficult to reason about.

What we want is a stream of states tracking which buttons are currently pressed. This state could either be a record of all possible keys with booleans or a collection with currently pressed keys like an [Array](https://pursuit.purescript.org/builtins/docs/Prim#t:Array). For this particular problem I think the [Set](https://pursuit.purescript.org/packages/purescript-ordered-collections/1.6.1/docs/Data.Set) type is perfect. It's an collection of unique items with some very useful functions like `member :: forall a. Ord a => a -> Set a -> Boolean` and `subset :: forall a. Ord a => Set a -> Set a -> Boolean`. `member` checks if a value is in the set. `subset` checks if all values in a set is contained in another set. That means you could very easily check if a very particular combination of keys are pressed at once. Very nice and clean!
```purescript
let keyStates = event # mapKeyStates

subscribe keyStates logShow <#> const unit

where
    mapKeyStates :: forall e. IsEvent e => e (Either String String) -> e (Set String)
    mapKeyStates e = fold (either delete insert) e empty
```
*Alright cool! When running the application we immediately print `(fromFoldable Nil)` to the browser console. Pressing buttons will print things like `(fromFoldable ("a" :  "d" : "s" : Nil))`.*

We could stop here but there are some further interesting things to look at. Holding buttons will fire multiple consecutive identical states which is useless. If you have a big application reacting to these states, it could get very expensive. We shouldn't do any performance optimizations before we know for sure we need it. I'm going to do it anyway to demonstrate something interesting.
```purescript
let keyStates = event # mapKeyStates # filterRepeats

subscribe keyStates logShow <#> const unit

where
    mapKeyStates :: forall e. IsEvent e => e (Either String String) -> e (Set String)
    mapKeyStates e = fold (either delete insert) e empty

    filterRepeats :: forall e. IsEvent e => e (Set String) -> e (Set String)
    filterRepeats e = withLast e # filter (\{ now, last } -> maybe true (_ /= now) last) <#> _.now
```
Let's pretend that `mapKeyStates` is an expensive operation. We might want to invoke `filterRepeats` before we map. However if we turn
```purescript
let keyStates = event # mapKeyStates # filterRepeats
```
into
```purescript
let keyStates = event # filterRepeats # mapKeyStates
```
the types don't line up. Do we need to do something big to fix this? No we don't. It turns out that in `filterRepeats :: forall e. IsEvent e => e (Set String) -> e (Set String)` there is nothing in the implementation that is the least aware that the contents of the event stream is a `Set`. The `_ /= now` is the only part that requires that whatever the stream contains, it has to support equality checking. If when we code we try to make fewer decision and keep things as general as possible, it makes code easier to refactor.

Changing
```purescript
filterRepeats :: forall e. IsEvent e => e (Set String) -> e (Set String)
filterRepeats e = withLast e # filter (\{ now, last } -> maybe true (_ /= now) last) <#> _.now
```
into
```purescript
filterRepeats :: forall e a. IsEvent e => Eq a => e a -> e a
filterRepeats e = withLast e # filter (\{ now, last } -> maybe true (_ /= now) last) <#> _.now
```
allows us to switch the filtering and mapping. Again it may not provide any benefit in our case but it's interesting. We definately want the ability to refactor our code. We might as well generalize `mapKeyStates :: forall e. IsEvent e => e (Either String String) -> e (Set String)`. Looking at the implementation we see that we don't actually require the contents to be strings. If we generalize them we see that the only requirement from `delete :: forall a. Ord a => a -> Set a -> Set a` is that it implements `Ord`.
```purescript
mapKeyStates :: forall e a. IsEvent e => Ord a => e (Either a a) -> e (Set a)
mapKeyStates e = fold (either delete insert) e empty
```
Okay, that was a little all over the place. Let's have a look at the final result.
```purescript
module Main where

import Prelude

import Data.Either (Either(..), either)
import Data.Filterable (filter)
import Data.Maybe (maybe)
import Data.Set (Set, delete, empty, insert)
import Effect (Effect)
import Effect.Console (logShow)
import FRP.Event (class IsEvent, create, fold, subscribe, withLast)
import Web.Event.EventTarget (addEventListener, eventListener)
import Web.Event.Internal.Types (Event)
import Web.HTML (window)
import Web.HTML.Window (toEventTarget)
import Web.UIEvent.KeyboardEvent (fromEvent, key)
import Web.UIEvent.KeyboardEvent.EventTypes (keydown, keyup)

main :: Effect Unit
main = do
    { event, push } <- create
    
    onKeyDownEvent <- eventListener $ onKey (Right >>> push)
    onKeyUpEvent   <- eventListener $ onKey (Left  >>> push)

    window <#> toEventTarget >>= addEventListener keydown onKeyDownEvent false
    window <#> toEventTarget >>= addEventListener keyup   onKeyUpEvent   false

    let keyStates = event # mapKeyStates # filterRepeats

    subscribe keyStates logShow <#> const unit

    where
        onKey :: (String -> Effect Unit) -> Event -> Effect Unit
        onKey dispatch e = fromEvent e <#> key # maybe (pure unit) dispatch

        mapKeyStates :: forall e a. IsEvent e => Ord a => e (Either a a) -> e (Set a)
        mapKeyStates e = fold (either delete insert) e empty

        filterRepeats :: forall e a. IsEvent e => Eq a => e a -> e a
        filterRepeats e = withLast e # filter (\{ now, last } -> maybe true (_ /= now) last) <#> _.now
```
I'm happy. I have this silly fetish for [point free style](https://en.wikipedia.org/wiki/Tacit_programming) that I might grow out of one day. But NOT TODAY! **guttural pirate laugh**
## Instructions
### Setup
1. Install required packages
    ```
    npx psc-package install web-uievents
    npx psc-package install console
    npx psc-package install event
    ```
1. Set up npm scripts in `package.json` for ease of use
    ```json
    "scripts": {
        "postinstall": "psc-package install",
        "build": "pulp --psc-package --watch build --to bundle.js"
    }
    ```
### Usage
1. Download and install all project dependencies
    ```
    npm install
    ```
1. Start the PureScript build process
    ```
    npm run build
    ```