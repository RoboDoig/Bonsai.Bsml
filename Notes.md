# Bonsai State Machine Language Notes

## General structure
The BSML consists of:

1. A tool that extracts available subjects from a Bonsai workflow with their output types, as well as individual state definitions from Bonsai.
2. A state machine description language (BSML) that is aware of the types generated in 1. Can be used to define initial state, transitions to other states, conditionals based on probability, Bonsai subjects, timers etc.
3. A compilation tool that converts the BSML into a dynamic state machine at runtime in Bonsai.

### Guarantees
- All states return Unit
- All states are defined as observables in the Bonsai workflow
- Every state transition produces an event `StateTransition (Unit)`
- Valid streams return a built-in type
- A Bonsai controller that manages state dictionary, current state, transition process between states

## What does the Bonsai workflow look like?
We have a Bonsai workflow called StimulusResponse.bonsai

We have a number of subscribable data stream subjects:

`CameraCapture --> Grayscale --> Threshold --> Sum --> Area (int)`

`Behavior --> PokeEvent --> PokeCount (int)`

`Behavior --> BeamBreak --> BeamBroken (bool)`

`Subscriber --> UserEvents (NetMQMessage) --> DecodedMessage (string)`

`Timer --> SelectMany --> StateTimer` (resets on `StateTransition`)

And some potential states defined as observables with a uniform output type (Unit):

`PreTrial` (hangs indefinitely, never returns Unit)

`TrialStart` (loads resources, configures some settings, returns Unit after it's done)

`StimulusA` (plays a stimulus and then returns Unit)

`StimulusB` (plays a stimulus and then returns Unit)

`WaitForPoke` (returns Unit on a poke event)

Which is run through a tool to produce a pseudo-package called StimulusResponse.foo

## What does the BSML look like?    

```
using StimulusResponse.foo

initialState: PreTrial

stateMachine: 
- {name: PreTrial, transitionsTo: {name: TrialStart, withCondition: DecodedMessage == "start"}}
- {name: TrialStart, transitionsTo: {name: WaitForPoke, withCondition: complete}}
- {name: WaitForPoke, transitionsTo: [{name: StimulusA, withCondition: complete}, {name: stimulusB, withCondition: StateTimer > 2}]}
- {name: StimulusA, transitionTo: {name: TrialStart, withCondition: complete}}
- {name: StimulusB, transitionTo: {name: TrialStart, withCondition: complete}}

```

### Writing BSML
We can import a .foo file with the description of the available states and data streams, along with the output types of the data stream.

We define an initial state for the state machine to start on.

We define the state machine. Each line is a transition definition. Each transition definition includes the current state it applies to, and then a single or array of transitions for each state the current state can transition to. Each transitionTo also includes a condition for the transition. The most simple is `completed` which causes the state to transition when it returns Unit. We can also produce transitions based on available data streams. These transitions in BSML are context-aware of the type. So for example a condition based on data stream with a numeric type can use ==, >=, <=, != etc. A boolean or string condition can only use ==.

N.B. need to think about how probabilistic transitions work. Do they use a probability provider in Bonsai?

A nice extension would be to do combined conditionals across the same or multiple data streams e.g. `StateTimer > 2 && StateTimer < 4`

There could even be some python extensibility to allow for expressions for more complex conditionals.

## How does the compilation tool work?

We already have the states themselves implemented in Bonsai. They are stored in a named dictionary so they can be looked up by the state-machine controller when it's time for a transition. We have a soft guarantee that the BSML description will not transition to a non-existent state because of the .foo file we generated.

What we need to generate is the conditional transitions. For the `complete` condition the state controller doesn't need to do much, it just needs to wait for `Unit` to return and then look up the appropriate transition. For data stream based transition (e.g. `StateTimer > 2`), the controller needs to create a parallel observable that subscribes to the data stream and defines a conditional transition.

## Possible alternate approaches
Every state is an IncludeWorkflow stored in extensions. The state machine is a single operator (also an extension) that is created via code generation like in sgen.
