# Bonsai State Machine Language Notes

## General structure
The BSML consists of:

1. A tool that extracts available subjects from a Bonsai workflow with their output types, as well as individual state definitions from Bonsai.
2. A state machine description language (BSML) that is aware of the types generated in 1. Can be used to define initial state, transitions to other states, conditionals based on probability, Bonsai subjects, timers etc.
3. A compilation tool that converts the BSML into a dynamic state machine at runtime in Bonsai.

### Guarantees
- All states return Unit
- Every state transition produces an event `StateTransition (Unit)`
- Valid streams return a built-in type

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
