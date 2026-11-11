# Local Markdown Issue Tracker

This repository has no configured external tracker. This directory is the local-Markdown fallback requested by Wayfinder. Run `/setup-matt-pocock-skills` if a different tracker should be configured.

The canonical map is [Find the way to an XD control DSL and real-time adapter contract](issues/xd-dsl-0001-xd-control-dsl.md). Its body is an index, not a duplicate specification.

See the [decision dependency diagram](dependencies.md) for the charting snapshot.

## Wayfinding operations

- **Issues:** `issues/*.md`; the JSON front matter is valid YAML and can be parsed with Python's standard library. `id` is immutable; `title` is the human-facing name. Always link using titles.
- **Labels and state:** use `wayfinder:map`, `wayfinder:research`, `wayfinder:prototype`, or `wayfinder:grilling`; task tickets use `wayfinder:task`. States are `open` or `closed`.
- **Children:** query `parent` for the map identity. Never list open tickets in the map body.
- **Blocking:** this file tracker has no native dependencies. `blocked_by` is authoritative; maintain the matching named links under `Blocked by`. Create issues before adding edges. Never create cycles.
- **Claims:** first set `assignee` to `ikolomiets` and add a `worker` field for the agent/session. Re-read before changing an issue to avoid replacing another session's claim. Open, assigned issues are not available to other sessions.
- **Frontier:** open children whose dependencies are all closed and whose assignee is null, sorted by id. Run the query below from the repository root.
- **Resolution:** append a new Markdown resolution comment under `comments/<issue-id>/`, linking evidence and research branches; then close the issue. Add a named link and one-line gist to the map. Do not paste the resolution into the issue body.
- **Research:** use isolated worktrees on `research/<name>` branches. Link the report, branch and commit from the resolution comment. Retain a report copy in this checkout so local links work; identify its source commit.
- **Concurrent edits:** workers own their claimed issue and report. The coordinating session updates the shared map and validates dependencies after merging their findings. Do not use the main checkout for concurrent branch switches.
- **Resumption:** read the map, query the frontier, claim one ticket, then invoke its named skills. A HITL ticket requires the user's live participation. Charting resolves no HITL tickets.

## Query the frontier

```sh
python3 - <<'PY'
from pathlib import Path
import json
issues = {}
for path in Path('docs/wayfinder/issues').glob('*.md'):
    meta = json.loads(path.read_text().split('---', 2)[1])
    issues[meta['id']] = (meta, path)
for key, (meta, path) in sorted(issues.items()):
    if (meta['parent'] == 'xd-dsl-0001' and meta['status'] == 'open'
        and meta['assignee'] is None
        and all(issues[d][0]['status'] == 'closed' for d in meta['blocked_by'])):
        print(f"[{meta['title']}]({path})")
PY
```
