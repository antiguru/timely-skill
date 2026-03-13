# timely-skill

[Claude Code](https://claude.com/claude-code) skills for writing [timely dataflow](https://github.com/TimelyDataflow/timely-dataflow) and [differential dataflow](https://github.com/TimelyDataflow/differential-dataflow) operators.

## Skills

* **writing-timely-operators** — Covers operator construction patterns, input draining, capability handling, frontier-driven processing, total vs. partial orders, yielding, data exchange, tee costs, container batching, and extension traits.
  Targets timely v0.27.
* **writing-differential-operators** — Covers the Collection API, arrangements, traces, cursors, reduce/join/iterate, core operators (reduce_trace, join_core, arrange_core), delta joins (dogsdogsdogs), and columnar data representation.
  Targets differential-dataflow v0.20.

## Installation

Add the plugin and install:

```
/plugin marketplace add antiguru/timely-skill
/plugin install timely-skill@timely-dataflow-skill
/plugin install differential-skill@timely-dataflow-skill
```

Skills are then available as `/timely-skill:writing-timely-operators` and `/differential-skill:writing-differential-operators`.

### Manual installation (development)

Add the skill directory to your Claude Code settings.
In `~/.claude/settings.json` or your project's `.claude/settings.json`:

```json
{
  "skills": [
    "/path/to/timely-skill/skills/writing-timely-operators",
    "/path/to/timely-skill/skills/writing-differential-operators"
  ]
}
```

## License

Licensed under either of [Apache License, Version 2.0](LICENSE-APACHE) or [MIT License](LICENSE-MIT), at your option.
