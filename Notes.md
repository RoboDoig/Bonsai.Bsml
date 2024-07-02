# Bonsai State Machine Language Notes

## General structure
The BSML consists of:

1. A tool that extracts available subjects from a Bonsai workflow with their output types, as well as individual state definitions from Bonsai.
2. A state machine description language (BSML) that is aware of the types generated in 1. Can be used to define initial state, transitions to other states, conditionals based on probability, Bonsai subjects, timers etc.
3. A compilation tool that converts the BSML into a dynamic state machine at runtime in Bonsai.

## What does the Bonsai workflow look like?
We have a number of subscribable data stream subjects:

`CameraCapture --> Grayscale --> Threshold --> Sum --> Area (int)`

