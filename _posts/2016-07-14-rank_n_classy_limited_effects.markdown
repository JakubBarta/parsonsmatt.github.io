---
title: "Rank 'n Classy Limited Effects"
date: 2016-07-14
layout: post
categories: programming
---

## Update from the Future (2019-05-22):

I don't recommend this technique any more.
It's quite complicated, and there's a much simpler formulation of the idea that uses records-of-functions instead of type class instances.
I've written up this formulation at the end of the blog post.


## Original Post

Side effects are awful.
Database access, HTTP requests, file reading, talking to Redis, ah!
Just so much gross IO code to shuffle around.
There's been a lot of effort to make things nicer.
Using monads to track effects in the type is a great start, but it's a little painful to work with without some good abstractions.

The [`mtl`](https://hackage.haskell.org/package/mtl) library does a great job of making abstractions, but it has a big flaw:
every new monad you want to introduce incurs $O(n^2)$ instances that you need to write.
If you're using `Reader`, `State`, `Logger`, `Http`, `Database`, `Email`, etc. (with special instances for testing/production/etc) then eventually this becomes too much of a burden.

More recently, the [free monad](http://www.haskellforall.com/2012/06/you-could-have-invented-free-monads.html) approach and [extensible-effects](https://hackage.haskell.org/package/extensible-effects) on top of it have become more popular.
Free monads solve the $O(n^2)$ instance problem, and they offer the ability to introspect on the computation and perform optimizations on it.
However, they have worse performance and are more complicated to implement.
You either have to build a giant command functor with an equally complex interpreter, or you need to build many small languages and manage their combinations.

I've been working on a very promising pattern at the day job lately, and it's worked out quite well thus far.
It seems to solve the issues involved with a ridiculous proliferation of monad instances *and* the complications involved with free monads, while still giving most of the benefits of both.

# `mtl` style, revisited

If you're unfamiliar, the `mtl` style of documenting effects is to use type classes to specify the effects of functions.
This has two main benefits:

## You don't have to worry about the order of your monad stack.

Concretely, these two functions are incompatible:

```haskell
foo :: StateT Int (Reader Char) Bool
foo = do
    int <- get
    char <- lift ask
    pure False

bar :: ReaderT Char (State Int) Bool
bar = do
    int <- lift get
    char <- ask
    pure True
```

Since we have to specify the `lift`s, we can't use them together.
The `mtl` approach makes this possible:

```haskell
foo :: (MonadState Int m, MonadReader Char m) => m Bool
foo = do
    int <- get
    char <- ask
    pure False

bar :: (MonadState Int m, MonadReader Char m) => m Bool
bar = do
    int <- get
    char <- ask
    pure True
```

The two functions are now easily interoperable! Nice.
We also didn't have to type `lift` a bunch, though truth be told, you just need the `mtl` type classes in scope for that to work.
While this is nice, it's not the *real* benefit of `mtl`.

## Strict specification of effects

When you use an `mtl` type class, you're restricting yourself to the interface that the type class provides.
If your monad is `StateT Int IO String`, then your monad can do any `IO` it wants.
That's no good! But if you know your function is `MonadState Int m => m String`, you know it can only operate on the state.

This lets you swap implementations easily.
The PureScript compiler had an awesome demonstration, where they moved a `WriterT` based logger to an `IO` based instance ([documented here](http://blog.functorial.com/posts/2016-01-31-PureScript-0.8.html)) for big performance gains.

# Mocking Monads

First, we'll define a type class that represents an effect.
We'll use a limited subset of `Http` requests.

```haskell
class Monad m => MonadHttp m where
    get :: Url -> m ByteString
    post :: ToJSON a => Url -> a -> m ByteString

type Url = String
```

We can easily make an instance for `IO`, using `wreq`:

```haskell
instance MonadHttp IO where
    get = fmap (view responseBody) . Wreq.get 
    post url = fmap (view responseBody) . Wreq.post url . toJson
```

Now, wherever we might have been using a function like `makeRequest :: Something -> IO OtherThing`, we can now abstract that `IO` into `makeRequest :: MonadHttp m => SomeThing -> m OtherThing`.
We can make the change transparently, since `IO` will still be inferred and used.
Plus, we have the assurance that we're not going to be accessing the database or printing any output in our `MonadHttp` functions.

Actually running HTTP requests in dev/test is boring. It's slow, annoying, unreliable, etc. and we'd much rather run locally for faster tests and more reliable development.
We can easily create a mock implementation of `MonadHttp` that does static returns:

```haskell
newtype MockHttp a 
    = MockHttp
    { runMockHttp :: ReaderT HttpEnv IO a 
    } deriving (Functor, Applicative, Monad, 
            MonadReader HttpEnv, MonadIO)

type HttpEnv = IORef HttpState
type HttpState = Map String ByteString

instance MonadHttp MockHttp where
    get url = do
        state <- ask >>= liftIO . readIORef
        pure (Map.lookup url state)
    post url body = do
        ref <- ask
        state <- liftIO (readIORef ref)
        liftIO (writeIORef ref (Map.insert url (encode body) state))
        pure "200 OK"
```

Now, with this instance, all of your `MonadHttp` requests will be performed *real* fast (and dumb).

# `RankN` Classy

Now, how can we select which interpretation we want?
We obviously want `IO` for production and `MockHttp` for testing.
Ultimately, what we want is one of a *family* of functions, with the following generalized type:

```haskell
forall app a. (forall m. MonadHttp m => m a) -> app a
```

And here, we see `RankNTypes` come into play.
The two type variables `app` and `a` are both `Rank1` types, since they're introduced at the leftmost part of the function and are mentioned to the right of all the function arrows.
The `m` type variable, on the other hand, is *hidden* -- that's a `Rank2` type variable.
The variables `app` and `a` can both be chosen by the user to be whatever works, but we're forcing the user to provide a value that not only *satisfies* the `MonadHttp` type class, but that it can *do no more* than `MonadHttp`. Consider this other signature:

```haskell
forall app m a. MonadHttp m => m a -> app a
```

It looks really similar, but that `m` is no longer hidden.
The user of the function can easily select `IO` as the implementation, as that satisfies the type class requirements.
The user could then execute arbitrary IO actions, and the types haven't helped much.

The `Rank2` type above forces the user to only use `MonadHttp` functions and actions.

We can safely specialize `app` to `IO` for now, which we'll need in order to read `IORef`s and do HTTP. So the functions we're looking for, then are:

```haskell
mockHttpRequests :: HttpEnv -> (forall m. MonadHttp m => m a) -> IO a
mockHttpRequests env action = 
    runReaderT (runMockHttp action) env

runHttpRequests :: (forall m. MonadHttp m => m a) -> IO a
runHttpRequests action = action
```

# Abstracting the implementations

Now, here's the last bit of the trick: You abstract out the implementations into a record.

```haskell
data Services
    = Services
    { runHttp :: forall a. (forall m. MonadHttp m => m a) -> IO a
    }
```

which you store in your application environment:

```haskell
type Application = ReaderT Services IO
```

Now, when you need to run HTTP requests, you can do:

```haskell
foobar :: Application Int
foobar = do
    service <- ask
    lift $ runHttp service $ do
        page <- get "http://wwww.google.com/"
        post "http://secret-data" (collectData page)
        pure (length page)
```

Then, while initializing your application, you can choose which environment to pass in:

```haskell
runApplicationProd :: Application a -> IO a
runApplicationProd action = runReaderT action (Services runHttpRequests)

runApplicationTest :: Application a -> IO a
runApplicationTest action = do
    ref <- newIORef initialHttpState
    runReaderT action (Services mockHttpRequests)
```

# What about free monads?

Oh this is the best part!

```haskell
reify :: (forall m. MonadHttp m => m a) -> Free HttpF a
reify = unFreeHttp

newtype FreeHttp a = FreeHttp { unFreeHttp :: Free HttpF a }

data HttpF next
    = Get Url (ByteString -> next)
    | forall a. FromJSON a => Post Url a next

instance MonadHttp FreeHttp where
    get url = FreeHttp (liftF (Get url id))
    post url body = FreeHttp (liftF (Post url body))
```

And now you can grab ahold of a Free monad representation of your AST.

# The Theory

Where a free monad seems like a great way to describe the effects of a computation, they seem to be more awkward at implementing *requests*.
I'm inspired by [Tomas Petricek's Coeffects](http://tomasp.net/coeffects/) concept, which describes the *context* or environment of a computation as an indexed comonad.
It seems like this approach allows you to request an environment comonad of interpreters for effects.
By reifying these effects at the value level (a trick similar to [Gabriel Gonzalez's First Class Module Records](http://www.haskellforall.com/2012/07/first-class-modules-without-defaults.html)), we avoid a lot of the problems with type classes and instances, while keeping the niceties of their abstractions.

An environment comonad is a essentially a really complicated way of saying "tuple", and that's left adjoint[^1] to a reader monad.
We get nice syntax sugar for monads and not comonads in Haskell, so `ReaderT Services` provides a nice approach to packaging up your environment's *request* context.

What's next? Well, you might note that the `Services` type was a little restricted. Indeed, the following is a bit nicer:

```haskell
data Services g
    = Services
    { runHttp :: forall a. (forall f. MonadHttp f => f a) -> g a 
    }
```

Which, hey... That's just a natural transformation! Specifically, a monad morphism.
We can reify that type with `ConstraintKinds` to get:

```haskell
type InterpreterFor g eff = forall a. (forall f. eff f => f a) -> g a
```

We can read this as: You choose the `eff`ect you want to interpret, and the monad you want to interpret it to.
But you can't choose the underlying concrete `f`, nor can you introspect on the `a`s to do so.

If you allow `TypeOperators`, then it even reads nicely, and we can replace our `IO` services with:

```haskell
data Services
    = Services
    { runHttp :: IO `InterpreterFor` MonadHttp 
    }
```

(*Not going to lie, that syntax really pleases my inner Rubyist*)

And, in a final act of cutting `IO` out of the program, we can parametrize that, yielding:

```haskell
data Services eff
    = Services
    { runHttp     :: eff `InterpreterFor` MonadHttp
    , runDatabase :: eff `InterpreterFor` MonadDatabase
    , runEmails   :: eff `InterpreterFor` MonadMandrill
    -- etc...
    }
```

along with a final `Application` type that abstracts over that:

```haskell
type Application = forall m. ReaderT (Services m) m
```

([credit to /u/Faucelme on reddit for that!](https://www.reddit.com/r/haskell/comments/4std0v/rank_n_classy_limited_effects/d5c2rcb))

Applications, then, are *just* an environment comonad of monad morphisms.
More plainly, they're a record of effect interpreters.

# Update:

Thanks to [/u/ElvishJerrico on Reddit](https://www.reddit.com/r/haskell/comments/4std0v/rank_n_classy_limited_effects/d5cdon4) who has implemented a `Category` instance for these morphisms!
This is a great way to compose effects. The given example is copied here:

```haskell
{-# LANGUAGE ConstraintKinds            #-}
{-# LANGUAGE DeriveFunctor              #-}
{-# LANGUAGE FlexibleInstances          #-}
{-# LANGUAGE GeneralizedNewtypeDeriving #-}
{-# LANGUAGE MultiParamTypeClasses      #-}
{-# LANGUAGE PolyKinds                  #-}
{-# LANGUAGE RankNTypes                 #-}
{-# LANGUAGE UndecidableInstances       #-}

module Lib where

import           Control.Category
import           Control.Monad.Free
import           Control.Monad.IO.Class
import           Control.Monad.Reader
import           Prelude                hiding (id, (.))
import qualified Prelude                as P

newtype Interpret c d = Interpret (forall n a. d n => (forall m. c m => m a) -> n a)

instance Category Interpret where
  id = Interpret P.id
  Interpret f . Interpret g = Interpret $ \h -> f (g h)

class Monad m => MonadHttp m where
  httpGet :: String -> m String

newtype HttpApp a = HttpApp { runHttpApp :: IO a }
  deriving (Functor, Applicative, Monad, MonadIO)

instance MonadHttp HttpApp where
  httpGet _ = return "[]" -- Should do actual IO

runIO :: Interpret MonadHttp MonadIO
runIO = Interpret $ \x -> liftIO $ runHttpApp x

newtype MockHttp m a = MockHttp { runMockHttp :: m a }
  deriving (Functor, Applicative, Monad)

instance MonadReader r m => MonadReader r (MockHttp m) where
  ask = MockHttp ask
  local f (MockHttp m) = MockHttp $ local f m

instance MonadReader String m => MonadHttp (MockHttp m) where
  httpGet _ = ask

runMock :: Interpret MonadHttp (MonadReader String)
runMock = Interpret runMockHttp

class Monad m => MonadRestApi m where
  getUserIds :: m [Int]

data RestApi a = GetUsers ([Int] -> a) deriving Functor

instance MonadRestApi (Free RestApi) where
  getUserIds = liftF $ GetUsers id

runRestApi :: Interpret MonadRestApi MonadHttp
runRestApi = Interpret $ iterA go where
  go (GetUsers f) = do
    response <- httpGet "url"
    f $ read response

runApplication :: Interpret MonadRestApi MonadIO
runApplication = runIO . runRestApi

mockApplication :: Interpret MonadRestApi (MonadReader String)
mockApplication = runMock . runRestApi
```

# Actually...

Don't use this.
It's complicated and overly boilerplatey.
Here's the `MonadHttp` effect code we ended up developing:

```haskell
type InterpreterFor g eff = forall a. (forall f. eff f => f a) -> g a

class MonadHttp m where
  get :: Url -> m ByteString
  post :: ToJSON a => Url -> a -> m ByteString

data Services eff 
    = Services 
    { runHttp :: eff `InterpreterFor` MonadHttp
    }
```

To create one of these `InterpreterFor`s, we have to make a type and define an instance:

```haskell
newtype MockHttp a 
    = MockHttp
    { runMockHttp :: ReaderT HttpEnv IO a 
    } deriving (Functor, Applicative, Monad, 
            MonadReader HttpEnv, MonadIO)

type HttpEnv = IORef HttpState
type HttpState = Map String ByteString

instance MonadHttp MockHttp where
    get url = do
        state <- ask >>= liftIO . readIORef
        pure (Map.lookup url state)
    post url body = do
        ref <- ask
        state <- liftIO (readIORef ref)
        liftIO (writeIORef ref (Map.insert url (encode body) state))
        pure "200 OK"

instance MonadHttp IO where
    get = fmap (view responseBody) . Wreq.get 
    post url = fmap (view responseBody) . Wreq.post url . toJson
```

Instead, we will create a record-of-functions for the type class, and create two values:

```haskell
data Http m = Http
    { get :: Url -> m ByteString
    , post :: forall a. ToJSON a => Url -> a -> m ByteString
    }

prodHttp :: Http IO
prodHttp = Http
    { get = fmap (view responseBody) . Wreq.get 
    , post = \url -> fmap (view responseBody) . Wreq.post url . toJson
    }

mockHttp :: IORef (Map String ByteString) -> Http IO
mockHttp env = Http
    { get = \url -> do
        state <- readIORef env
        pure (Map.lookup url state)
    , post = \url body -> do
        state <- liftIO (readIORef env)
        liftIO (writeIORef env (Map.insert url (encode body) state))
        pure "200 OK"
    }
```

and we include the record of functions directly into `Services`:

```haskell
data Services eff = Services { http :: Http eff }

type Application eff = ReaderT (Services eff) eff
```

This gives us the same expressive power without having to deal with type classes, instances, and any of that other hassle.
You construct plain values and pass them around.

[^1]: I had initially written "isomorphic," and was corrected by George Wilson who reminded me that tuple and reader form an adjunction, and that the isomorphism is between Kleisli (Reader r) and CoKleisli (Env r)
