# decisions/ — append-only

To reverse a decision, write a new record and set `superseded-by` on the old
one. **Never edit a record's Decision or Rationale after acceptance**, and never
delete a file — the history of a reversed decision is why it does not get
reversed back.

Amendments that add cost or consequence are fine; mark them with an `Amended:`
line. New records use `../../_templates/tpl-decisions.md`.
