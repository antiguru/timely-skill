# Timely skill

This repo contains Claude Code skills for writing timely dataflow and differential dataflow operators.

## Skill content style

* Skills are API references for the current version. Not migration guides.
* State what IS, not what WAS.
  No "was removed", "no longer", "earlier versions", "replaced by".
* Version references only in the target version line at the top of each skill (e.g., "This skill targets **timely v0.29**.").
  Do not reference version numbers elsewhere in the skill body.

## Updating skills for new releases

* Check crates.io for latest versions of timely, differential-dataflow, and columnar.
* Fetch changelogs or release notes to identify actual API changes.
  Do not blindly bump version numbers — verify what changed.
* Update type signatures, code examples, and descriptive text to match the new API.
* After editing, scan each skill for stale references (old type names, removed traits, outdated generic bounds).
