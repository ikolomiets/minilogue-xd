# Resolution: XD MIDI capabilities and playback constraints

The research question is answered at the documented-capability and source-comparison level. Read the [cited capability report](../../research/xd-midi-capabilities.md) for the inventory, exact discrepancies, and evidence limits.

In-scope control spans channel messages and program SysEx. Edit-buffer loads, stored writes, and onboard sequences have distinct effects. Documentation and source disagree on several mappings and dump details; none of those discrepancies has been established as a hardware bug. Wire-occupancy calculations constrain scheduling discussions but do not establish device latency or jitter guarantees.

No hardware measurements or new implementation tests were performed. Runtime performance, firmware-specific compatibility, and disputed encodings remain unresolved evidence obligations, now explicitly carried by the [encoding evidence decision](../../issues/xd-dsl-0013-encoding-evidence.md) and the existing scheduling and acceptance tickets.

- Evidence branch: `research/xd-midi-capabilities`
- Evidence commit: `3a78b2e06e97087da788320c45b3ecd4759a4dd4`
- Report in this checkout is an exact copy of the committed asset.
- Graduated compatibility fog into [Choose the evidence standard for disputed MIDI encodings](../../issues/xd-dsl-0013-encoding-evidence.md), which blocks sound/patch semantics until its evidence policy is settled.

Resolved for ikolomiets by the XD MIDI research agent; integrated by the coordinating session. No human design decision was resolved.
