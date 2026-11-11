# Resolution: structured-document and musical-time representation research

The research question is answered at decision-support fidelity. Read the [cited research report](../../research/structured-score-formats.md) for evidence and tradeoffs.

JSON plus schema can express the agreed scope, but cross-event musical consistency needs semantic validation. YAML introduces loader-profile choices. MusicXML separates notation from sounding behavior; MNX offers relevant structured concepts but is a draft. Exact duration, retained notation identity, schema/version policy, and validation boundaries remain human decisions. No syntax was selected.

No parser benchmarks, consumer interoperability tests, or hardware measurements were performed. One MNX detail-page retrieval failed; no normative claim depends on that page. These limitations do not block the representation-options question.

- Evidence branch: `research/structured-score-formats`
- Evidence commit: `8e8114d495119a8c9bcd43e5202e9d92dd2856a5`
- Report in this checkout is an exact copy of the committed asset.
- Newly sharp questions fit the existing [document shape](../../issues/xd-dsl-0004-document-shape.md) and [musical semantics](../../issues/xd-dsl-0005-musical-semantics.md) tickets; no additional child is needed.

Resolved for ikolomiets by the score-format research agent; integrated by the coordinating session.
