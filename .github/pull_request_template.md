<!--
Title this pull request `[Area] Sentence case description`.
Areas: Connector, Docs, CI, Tests, Release.
Example: [Connector] Reject unsigned webhook deliveries
-->

## Description

<!--
Three to five sentences in plain language: what changed, why, the one risk,
and how you verified it. Describe the behavior a caller sees, not the files
you touched, because the Files tab already shows those.

Worked example:

  Inbound deliveries with a missing signature header were normalized instead
  of rejected, so an unsigned request reached the trigger body. This change
  fails those deliveries closed and returns a typed rejection. The risk is
  that an operator who never configured a signing secret now sees rejections
  where deliveries used to pass, which is the intended behavior and is called
  out in the README. Verified with a new fixture that replays a signed and an
  unsigned delivery through `normalize_inbound`, plus the full package gate.
-->

## Test plan

<!--
What you actually ran and what happened. A passing count is not evidence on
its own: name the check that would have failed if this change were wrong, and
say what remains unverified.
-->
