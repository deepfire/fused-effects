# Defining new effects

Effects are a powerful mechanism for abstraction, and so defining new effects is a valuable tool for system architecture. Effects are modelled as (higher-order) functors, with an explicit continuation denoting the remainder of the computation after the effect.

It’s often helpful to start by specifying the types of the desired operations. For our example, we’re going to define a `Teletype` effect, with `read` and `write` operations, which read a string from some input and write a string to some output, respectively:

```haskell
data Teletype (m :: * -> *) k
read :: (Member Teletype sig, Carrier sig m) => m String
write :: (Member Teletype sig, Carrier sig m) => String -> m ()
```

Effect types must have two type parameters: `m`, denoting any computations which the effect embeds, and `k`, denoting the remainder of the computation after the effect. Note that since `Teletype` doesn’t use `m`, the compiler will infer it as being of kind `*` by default. The explicit kind annotation on `m` corrects that.

Next, we can flesh out the definition of the `Teletype` effect by providing constructors for each primitive operation:

```haskell
data Teletype (m :: * -> *) k
  = Read (String -> k)
  | Write String k
  deriving (Functor)
```

The `Read` operation returns a `String`, and hence its continuation is represented as a function _taking_ a `String`. Thus, to continue the computation, a handler will have to provide a `String`. But since the effect type doesn’t say anything about where that `String` should come from, handlers are free to read from `stdin`, use a constant value, etc.

On the other hand, the `Write` operation returns `()`. Since a function `() -> k` is equivalent to a (non-strict) `k`, we can omit the function parameter.

In addition to a `Functor` instance (derived here using `-XDeriveFunctor`), we need two other instances: `HFunctor` and `Effect`. `HFunctor`, named for “higher-order functor,” has one non-default operation, `hmap`, which applies a function to any embedded computations inside an effect. Since `Teletype` is first-order (i.e. it doesn’t have any embedded computations), the definition of `hmap` can be given using `coerce`:

```haskell
instance HFunctor Teletype where
  hmap _ = coerce
```

`Effect` plays a similar role to the combination of `Functor` (which operates on continuations) and `HFunctor` (which operates on embedded computations). It’s used by `Carrier` instances to service any requests for their effect occurring inside other computations—whether embedded or in the continuations. Since these may require some state to be maintained, `handle` takes an initial state parameter (encoded as some arbitrary functor filled with `()`), and its function is phrased as a _distributive law_, mapping state functors containing unhandled computations to handled computations producing the state functor alongside any results.

Since `Teletype`’s operations don’t have any embedded computations, the `Effect` instance only has to operate on the continuations, by wrapping the computations in the state and applying the handler:

```haskell
instance Effect Teletype where
  handle state handler (Read    k) = Read (handler . (<$ state) . k)
  handle state handler (Write s k) = Write s (handler (k <$ state))
```

Now that we have our effect datatype, we can give definitions for `read` and `write`:

```haskell
read :: (Member Teletype sig, Carrier sig m) => m String
read = send (Read ret)

write :: (Member Teletype sig, Carrier sig m) => String -> m ()
write s = send (Write s (ret ()))
```

This gives us enough to write computations using the `Teletype` effect. The next section discusses how to run `Teletype` computations.

## Defining effect handlers

Effects only specify actions, they don’t actually perform them. That task is left up to effect handlers, typically defined as functions calling `interpret` to apply a given `Carrier` instance.

Following from the above section, we can define a carrier for the `Teletype` effect which runs the calls in an underlying `MonadIO` instance:

```haskell
newtype TeletypeIOC m a = TeletypeIOC { runTeletypeIOC :: m a }

instance (Carrier sig m, MonadIO m) => Carrier (Teletype :+: sig) (TeletypeIOC m) where
  ret = TeletypeIOC . ret

  eff = TeletypeIOC . handleSum (eff . handleCoercible) (\ t -> case t of
    Read    k -> liftIO getLine      >>= runTeletypeIOC . k
    Write s k -> liftIO (putStrLn s) >>  runTeletypeIOC   k)
```

Here, `ret` is responsible for wrapping pure values in the carrier, and `eff` is responsible for handling an effectful computations. Since the `Carrier` instance handles a sum (`:+:`) of `Teletype` and the remaining signature, `eff` has two parts: a handler for `Teletype` (`alg`), and a handler for teletype effects that might be embedded in other effects in the signature.

In this case, since the `Teletype` carrier is just a thin wrapper around the underlying computation, we can use `handleCoercible` to handle any embedded `TeletypeIOC` carriers by simply mapping `coerce` over them.

That leaves `alg`, which handles `Teletype` effects with one case per constructor. Since we’re assuming the existence of a `MonadIO` instance for the underlying computation, we can use `liftIO` to inject the `getLine` and `putStrLn` actions into it, and then proceed with the continuations, unwrapping them in the process.

Users could use `interpret` directly to run the effect, but it’s more convenient to provide effect handler functions applying `interpret` and then unwrapping the carrier:

```haskell
runTeletypeIO :: (MonadIO m, Carrier sig m) => Eff (TeletypeIOC m) a -> m a
runTeletypeIO = runTeletypeIOC . interpret
```

In general, carriers don’t have to be `Functor`s, let alone `Monad`s. However, sometimes—especially in cases where the carrier is a thin wrapper like this—they can be more convenient to write using (derived) `Monad` instances. In this case, by using `-XGeneralizedNewtypeDeriving`, we can derive `Functor`, `Applicative`, `Monad`, and `MonadIO` instances for `TeletypeIOC`:

```haskell
newtype TeletypeIOC m a = TeletypeIOC { runTeletypeIOC :: m a }
  deriving (Applicative, Functor, Monad, MonadIO)
```

This allows us to use `liftIO` directly on the carrier itself, instead of only in the underlying `m`; likewise with `>>=`, `>>`, and `pure`:

```haskell
instance (MonadIO m, Carrier sig m) => Carrier (Teletype :+: sig) (TeletypeIOC m) where
  ret = pure
  eff = handleSum (TeletypeIOC . eff . handleCoercible) (\ t -> case t of
    Read    k -> liftIO getLine      >>= k
    Write s k -> liftIO (putStrLn s) >>  k)
```
