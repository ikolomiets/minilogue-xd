# Decision dependencies

Charting snapshot. Arrows run from prerequisites to the decisions they unblock. Research is complete; human decisions remain open. Use the [tracker query](README.md#query-the-frontier) for current availability after later sessions.

```mermaid
flowchart TD
    xd_dsl_0002["Establish XD MIDI capabilities and playback constraints"]
    xd_dsl_0003["Establish structured-document and musical-time representation options"]
    xd_dsl_0004["Choose the structured score document shape"]
    xd_dsl_0005["Define musical time and note semantics"]
    xd_dsl_0006["Bound the supported performance environment"]
    xd_dsl_0007["Define sound-parameter and patch control semantics"]
    xd_dsl_0008["Define the boundary between scores and onboard sequences"]
    xd_dsl_0009["Define arbitration between score automation and live changes"]
    xd_dsl_0010["Define scheduling and MIDI-output guarantees"]
    xd_dsl_0011["Define the Rust adapter and CLI boundary"]
    xd_dsl_0012["Define the specification handoff and acceptance evidence"]
    xd_dsl_0013["Choose the evidence standard for disputed MIDI encodings"]
    xd_dsl_0003 --> xd_dsl_0004
    xd_dsl_0005 --> xd_dsl_0004
    xd_dsl_0002 --> xd_dsl_0007
    xd_dsl_0013 --> xd_dsl_0007
    xd_dsl_0002 --> xd_dsl_0008
    xd_dsl_0005 --> xd_dsl_0008
    xd_dsl_0007 --> xd_dsl_0008
    xd_dsl_0005 --> xd_dsl_0009
    xd_dsl_0007 --> xd_dsl_0009
    xd_dsl_0002 --> xd_dsl_0010
    xd_dsl_0005 --> xd_dsl_0010
    xd_dsl_0006 --> xd_dsl_0010
    xd_dsl_0008 --> xd_dsl_0010
    xd_dsl_0009 --> xd_dsl_0010
    xd_dsl_0004 --> xd_dsl_0011
    xd_dsl_0007 --> xd_dsl_0011
    xd_dsl_0008 --> xd_dsl_0011
    xd_dsl_0009 --> xd_dsl_0011
    xd_dsl_0010 --> xd_dsl_0011
    xd_dsl_0004 --> xd_dsl_0012
    xd_dsl_0005 --> xd_dsl_0012
    xd_dsl_0006 --> xd_dsl_0012
    xd_dsl_0007 --> xd_dsl_0012
    xd_dsl_0008 --> xd_dsl_0012
    xd_dsl_0009 --> xd_dsl_0012
    xd_dsl_0010 --> xd_dsl_0012
    xd_dsl_0011 --> xd_dsl_0012
    xd_dsl_0002 --> xd_dsl_0013
    xd_dsl_0006 --> xd_dsl_0013
    classDef resolved fill:#d5f5df,stroke:#26834a,color:#123d21
    class xd_dsl_0002 resolved
    class xd_dsl_0003 resolved
```

## Available after charting

- [Define musical time and note semantics](issues/xd-dsl-0005-musical-semantics.md)
- [Bound the supported performance environment](issues/xd-dsl-0006-performance-environment.md)
