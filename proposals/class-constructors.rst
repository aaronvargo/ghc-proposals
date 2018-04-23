.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.

.. trac-ticket:: Leave blank. This will eventually be filled with the Trac
                 ticket number which will track the progress of the
                 implementation of the feature.

.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.

.. highlight:: haskell

This proposal is `discussed at this pull requst <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_. **After creating the pull request, edit this file again, update the number in the link, and delete this bold sentence.**

.. contents::

Class Constructors
==================

It's currently impossible to abstract over the construction of type class
instances. In addition to code duplication, this lack of abstraction makes it
impossible to change existing class hierarchies without breaking existing code.
This proposal enables such abstraction by adding the new notion of *class
constructors*.

A class constructor is a mapping from a set of parameters to a set of instances.
For example, one may define a class constructor which, given implementations of
``(>>=)`` and ``pure`` as arguments, constructs instances of ``Functor``,
``Applicative``, and ``Monad``.

Under this proposal, instance declarations instantiate class constructors,
rather than instantiate classes directly. This introduces a new layer of
abstraction which can allow class hierarchies to be changed in a backwards
compatible manner.

Motivation
------------

Consider the following code, which instantiates ``Functor``, ``Applicative``,
and ``Monad`` for a type constructor ``Foo``, given implementations of ``(>>=)``
and ``pure``::

  instance Functor Foo where
    fmap = liftM

  instance Applicative Foo where
    (<*>) = ap
    pure = fooPure

  instance Monad Foo where
    (>>=) = fooBind

The above is a common pattern for deriving these instances, yet there's no way
to abstract over it without resorting to template haskell or preprocessing. It
must be repeated every time one wishes to derive ``Functor``, ``Applicative``
and ``Monad`` instances from ``(>>=)`` and ``pure``. While this amount of
boilerplate may seem inconsequential, the amount increases with the number of
super classes. This can discourage more fine-grained class hierarchies,
resulting in what Edward Kmett has called `Procrustean Mathematics
<https://www.schoolofhaskell.com/user/edwardk/editorial/procrustean-mathematics>`_.

Worse than the boilerplate itself though is the fact that it makes the code
fragile. Consider how the above instances would need to be modified if the
``(<*>)`` method were split out into its own ``Apply`` super class::

  class Functor f => Apply f where
    (<*>) :: f (a -> b) -> f a -> f b

  class Apply f => Applicative f where
    pure :: a -> f a

Every instantiation of ``Applicative`` would then need to be modified to add an
extra line of boilerplate instantiating the new ``Apply`` class, making this a
breaking change.

Proposed Change Specification
-----------------------------

Add a new namespace for class constructors to reside in. Have instance
declarations instantiate class constructors, rather than instantiate classes
directly. That is, in an instance declaration::

  instance C T where
    param = ...

``C`` will refer to a *class constructor* ``C``, rather than a class ``C``, and
``param`` will refer to a named parameter of the class constructor ``C``, rather
than a class method.

Every class declaration additionally introduces a new class constructor, called
a *primary* class constructor, which by default has the same name as the class.
Primary class constructors take parameters corresponding to the methods of their
corresponding class, and instantiate said class.

Thus far, this proposal has simply described an alternative way of thinking
about how the language currently works. No code will be broken.

Add a new language extension, ``-XClassConstructors``. When it's enabled, a
primary constructor may be given a different name than its corresponding class.
Let the following syntax declare a class ``Foo`` with primary constructor
``Bar``::

  class Foo a = Bar where

Given the above, one needs to write ``instance Bar a`` rather than ``instance
Foo a`` to instantiate the ``Foo`` class.

Allow the user to define additional, *secondary* class constructors. A secondary
constructor instantiates any number of other class constructors, which may be
either primary or secondary. For example the following defines a secondary
constructor named ``BindPure`` (NB: it does *not* define a new class)::

  class constructor BindPure f where
    bindArg :: f a -> (a -> f b) -> f b
    pureArg :: a -> f a

    instance Functor f where
      fmap = liftM

    instance Applicative f where
      (<*>) = ap
      pure = pureArg

    instance Monad f where
      (>>=) = bindArg

The ``BindPure`` constructor takes a type parameter, ``f``, and two named
parameters, ``bindArg`` and ``pureArg``, and instantiates ``Functor f``,
``Applicative f``, and ``Monad f``. It can be utilized like so::

  instance BindPure Foo where
    bindArg = fooBind
    pureArg = fooPure

Where the above has the same semantics as if the user had written::

  instance Functor Foo where
    fmap = liftM

  instance Applicative Foo where
    (<*>) = ap
    pure = fooPure

  instance Monad Foo where
    (>>=) = fooBind

Class constructors can be constrained, e.g. the following constructor can be
used to construct a ``Semigroup`` given an instance of ``Ord``::

  class constructor Ord a => Minimum a where
    instance Semigroup a where
      x <> y | x <= y = x
             | otherwise = y

Primary class constructors are constrained by the superclass constraint of their
corresponding class. E.g. the primary constructor for ``Ord a`` has a constraint
``Eq a``. The following (contrived) constructor is invalid, due to the ``Eq``
constraint of the ``Ord`` constructor not being satisfied::

  class constructor Ord b => OrdBy b a where
    from :: a -> b

    instance Ord a where
      x `compare` y = from x `compare` from y

However, the following two constructors are valid::

  class constructor Ord b => EqOrdBy b a where
    from :: a -> b

    instance Eq a where
      xs == ys = from xs == from ys

    instance Ord a where
      x `compare` y = from x `compare` from y

  class constructor (Ord b, Eq a) => OrdBy b a where

    instance Ord a where
      x `compare` y = from x `compare` from y

Instantiations of class constructors can be polymorphic::

  newtype MyList a = MyList { myListToList :: [a] }

  -- `Ord a` is required to satisfy `Ord [a]`
  instance Ord a => EqOrdBy [a] (MyList a) where
    from = myListToList

The semantics of the above is that the context gets pushed out to each
constructed instance::

  instance Ord a => Eq (MyList a) where
    xs == ys = myListToList xs == myListToList ys
    
  instance Ord a => Ord (MyList a) where
    xs `compare`` ys = myListToList xs `compare` myListToList ys

Note that the above is an undesirable way to derive the ``Eq`` instance for
``MyList`` however, as ``Eq (MyList a)`` should only require ``Eq a``. To
address situations like this, class constructors can also construct polymorphic
instances. For example, the following constructor can construct lexicographical
``Eq`` and ``Ord`` instances for any ``Foldable``::

  class constructor Foldable f => Lexicographical f where

    instance Eq a => Eq (f a) where
      xs == ys = toList xs == toList ys

    instance Ord a => Ord (f a) where
      xs `compare` ys = toList xs `compare` toList ys

Splitting a Class
^^^^^^^^^^^^^^^^^

Given an existing ``Applicative`` class::

  class Functor f => Applicative f where
    (<*>) :: f (a -> b) -> f a -> f b
    pure :: a -> f a

An ``Apply`` superclass can be split out in a backwards compatible matter::

  class Functor f => Apply f where
    (<*>) :: f (a -> b) -> f a -> f b

  class Apply f => Applicative f = ApplicativeSansApply where
    pure :: a -> f a

  class constructor Functor f => Applicative f where
    (<*>) :: f (a -> b) -> f a -> f b
    pure :: a -> f a

    instance Apply f where
      (<*>) = (<*>) -- note the lhs is just the name of the argument to `Apply`,
                    -- while the rhs refers to the parameter of the `Applicative`
                    -- constructor being defined (which shadows the `(<*>)` method).

    instance ApplicativeSansApply f where
      pure = pure

The key in the above is to give the primary constructor of the new definition of
``Applicative`` a different name, and then to define the ``Applicative``
constructor such that no existing ``Applicative`` instances break. Note this
only maintains backwards compatibility if the ``Apply`` class didn't previously
exist.

TODO
^^^^

- Default Definitions

  - Can lead to boilerplate. Required for backwards compatiblity, but can be
    subsumed by using records instead
  - class constructor arguments as records

- Making an existing class a new superclass with minimal breakage
- Type Families - both associated and standalone
- Some constructors require QuantifiedConstraints. Possible work arounds.
- GeneralizedNewtypeDeriving, DeriveAnyClass

  - GND should take a class constructor and derive every instance it would create
  - class constructors can ultimately subsume GND

- Overlapping Pragmas
- Default Signatures
- Importing class constructors
- Alternatives

  - Default superclasses
  - Deriving via


Specify the change in precise, comprehensive yet concise language. Avoid words like should or could. Strive for a complete definition. Your specification may include,

* grammar and semantics of any new syntactic constructs
* the types and semantics of any new library interfaces
* how the proposed change addresses the original problem
* how the proposed change might interact with existing language or compiler features

Note, however, that this section need not describe details of the
implementation of the feature. The proposal is merely supposed to give a
conceptual specification of the new feature and its behavior.

Effect and Interactions
-----------------------
Detail how the proposed change addresses the original problem raised in the motivation. Detail how the proposed change interacts with existing language or compiler features and provide arguments why this is not going to pose problems.



Costs and Drawbacks
-------------------
Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.



Alternatives
------------
List existing alternatives to your proposed change as they currently exist and discuss why they are insufficient.



Unresolved questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.



Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?
