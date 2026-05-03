# Timely skill

This repo contains Claude Code skills for writing timely dataflow and differential dataflow operators.

## Writing style

* Use a concise, technical writing style.
* Avoid filler words, use active voice.
* Only make claims based on evidence.
  If there's no evidence, ask what to do.

## Markdown formatting

* Use the asterisk `*` for lists, not dash `-`.
* Put each sentence on its own line.
* Capitalization of headers: first word upper case, and after colon.
  Uppercase proper nouns, but no other words.
* When changing code, do not drop comments.

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
