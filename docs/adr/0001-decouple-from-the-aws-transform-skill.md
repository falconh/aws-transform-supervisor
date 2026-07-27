# Decouple from AWS's shipped `aws-transform` skill

AWS ships an official `aws-transform` agent skill (v1.0.0) that already performs repo
inspection, TD matching, local and remote execution, and monitoring — so this plugin
deliberately owns only the two things that skill does not do: supervising a run and
planning the leftovers. We do **not** call it at runtime. Its text opens with a mandatory
"Greet and Wait" section ("introduce AWS Transform with this exact text… Do NOT inspect
any files, run any commands… until the user responds") and a rule to confirm with the user
before every execution; loading that mid-loop would both stall an autonomous run on every
iteration and inject another agent's control-flow imperatives into ours. Instead we derive
current CLI facts from `atx custom def exec --help` on the machine, which is authoritative,
free, always current, and contains no instructions aimed at an agent.

## Consequences

AWS's skill remains valuable as design-time reference and is where much of this plugin's
knowledge came from. Because we no longer inherit its rules, its mandatory `--telemetry`
flag becomes our choice rather than an obligation; we include it by default and document
the opt-out.

**Handoff is exempt (added in 0.1.4).** This decision bars calling AWS's skill *inside* our
run: its gates stall an unattended loop, and its imperatives contaminate our control flow.
Neither hazard exists when we are handing work off and stopping — there is no loop left to
stall, and its greet-and-wait is exactly what a user beginning a fleet migration should meet.
So `run-migration` preflight may reach it via `retrieve_skill` when declining fleet work. The
rule is about the loop, not the skill.
