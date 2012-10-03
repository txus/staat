# Staat

Staat is a State Machine library in C.

## Basic concepts

A state machine is just a series of states (let's say `Sleeping`, `Awake`,
`Eating`) and a set of possible transitions among them (`Sleeping->Awake`,
`Awake->Eating`, etc...).

Additionally, transitions in a state machine may have guards, which are like
preconditions that get executed before transitioning, and have the ability to
do something and ultimately decide if the transition should be aborted or
carried out successfully.

Staat implements these concepts in an easy to use way for your C programs!

## Usage

State Machines in Staat are modeled as blueprints with their states and
transitions, normally as static variables.

For example, let's assume we want to model different cats with the same state
machine, defining states such as Sleeping, Awake, and Eating, and transitions
between those. Ideally we model the Cat state machine **once**, and then, we
__instantiate__ different cat state machines, each of those with their own
state at a given moment, but sharing the transition behavior.

That's why the Cat State Machine will live in a static variable and be
ideally initialized **at load time**, and then each different cat will be born
and die **at runtime**.

Let's start modeling the possible states of a cat:

```c
typedef enum {
  Sleeping,
  Awake,
  Eating
} State;
```

Now we set a static Cat State Machine blueprint (from which we'll initialize
all the cats of the program) and configure it:

```c
#include <staat/blueprint.h>
static StaatBlueprint *CatBlueprint = NULL;

/*
 * At load time, we model the Cat State Machine with the StaatBlueprint_new
 * macro and all possible states, then add the transitions, and then any
 * needed guards.
 */

CatBlueprint = StaatBlueprint_new(Sleeping, Awake, Eating);

StaatBlueprint_add_transition(CatBlueprint, "wake_up", Sleeping, Awake);
StaatBlueprint_add_transition(CatBlueprint, "eat", Awake, Eating);
StaatBlueprint_add_transition(CatBlueprint, "sleep", Awake, Sleeping);
StaatBlueprint_add_transition(CatBlueprint, "sleep", Eating, Sleeping);
```

Let's also add a guard that will be executed every time we hit the "sleep"
transition. The guard will be called "purr" and will make the cat purr before
going to sleep only if it was eating.

To do that we will have previously defined the purr guard function, which
receives a **from** parameter (from which state it came), a **to** (to which
state it is transitioning), and an **object** (which is the object that we'll
pass to the state machine constructor later on, namely the container of the
state machine, we'll see that in a moment).

Remember that guards must return 0 if everything went well and they allow the
transition, or 1 otherwise (and that will prevent them from transitioning).

```c
int purr_guard(int from, int to, void *object) {
  if (from == Eating) {
    ((Cat*)object)->purr();
  }
  return 0;
}

StaatBlueprint_add_guard(CatBlueprint, "sleep", purr_guard);
```

Good. Now we're ready to instantiate our first cat state machine. Our program
will consist of Cat objects (structs) with various properties, one of which
will be their state (an instantiated cat state machine). Let's see how we
model the cats:

```c
#include <staat/machine.h>

typedef struct {
  StaatMachine *state;
  /* The two things below are irrelevant */
  int mood;
  CatFn purr;
} Cat;

Cat* Cat_new() {
  Cat *cat = calloc(1, sizeof(Cat));
  cat->purr = default_purr_fn; // whatever
  cat->mood = 0;

  /*
    HERE!
    We initialize a Cat State Machine with an initial "sleeping" state **and a
    reference to the cat object**, so that guards have access to it and can
    manipulate / query it.
  */
  cat->state = StaatMachine_new(CatBlueprint, cat, Sleeping);

  return cat;
}
```

Now we have a way to create cats that start sleeping. Let's try it out!

```c
Cat *cat = Cat_new();
int current_state = cat->state->current;        // Sleeping
StaatMachine_transition(cat->state, "wake_up"); // returns 0 = OK
current_state = cat->state->current;            // Awake
StaatMachine_transition(cat->state, "eat");     // returns 0 = OK
current_state = cat->state->current;            // Eating
```

But what if the transition doesn't work? For example, let's try transitioning
to the current state (nonsense):

```c
StaatMachine_transition(cat->state, "eat");     // returns 1 = wrong!
```

Okay, now let's go to sleep:

```c
StaatMachine_transition(cat->state, "sleep");   // returns 0 = OK
```

As we defined in the guard, the `cat->purr()` function will be called before
transitioning. Guards are useful to define preconditions to the transition,
and abort it altogether if at least one of them fails.

## Contributing

1. Fork it
2. Create your feature branch (`git checkout -b my-new-feature`)
3. Commit your changes (`git commit -am 'Added some feature'`)
4. Push to the branch (`git push origin my-new-feature`)
5. Create new Pull Request

## Who's this

This was made by [Josep M. Bach (Txus)](http://txustice.me) under the MIT
license. I'm [@txustice](http://twitter.com/txustice) on twitter (where you
should probably follow me!).
