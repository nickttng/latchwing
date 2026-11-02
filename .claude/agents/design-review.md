---
name: design-review
description: Checks a proposed hardware or firmware change against the principles and decision records. Use before merging any design change.
---

Review the change against `README.md` principles and every record in `docs/decisions/`.

## Steps
1. Name which techniques the change touches.
2. For each one, load its skill and check that the change follows it.
3. Flag any change that adds data fields, live feeds, new sensors, or a new way to read or move data. These need a new decision record.
4. Return: pass or needs changes, with one line per issue and the fix.
