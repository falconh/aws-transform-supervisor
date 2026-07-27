# AWS Transform Supervisor

Supervises a single AWS Transform (`atx`) migration of one application: publishes a
Recipe the team owns, watches the Attempt, intervenes only in two sanctioned ways,
and converts the Leftovers ATX reports into a written remediation plan.

## Language

**Recipe**:
The transformation instructions this plugin owns, publishes and version-controls.
The same object is addressed as a *Transformation Definition* by the `atx` CLI.
_Avoid_: playbook, template, transformation package

**Exit Criterion**:
A concrete, checkable statement of done declared inside a Recipe. ATX validates
against these and reports which ones it did not meet.
_Avoid_: acceptance test, requirement, definition of done

**Target**:
The application repository being transformed.
_Avoid_: project, codebase, source

**Disposable Clone**:
The scratch copy of the Target in which every Attempt runs, so that a fully
trusted agent never executes against the user's working copy.
_Avoid_: workspace, sandbox, checkout

**Base Commit**:
The Target commit recorded before the first Attempt. Every Attempt branches from
it, so attempts are siblings rather than a chain.
_Avoid_: baseline, original, starting point

**Attempt**:
One execution of a Recipe against the Target, on its own branch off the Base
Commit. Attempts are never reset and never discarded.
_Avoid_: run, try, iteration, retry

**Conversation**:
ATX's unit of execution state, identified by a conversation id and resumable for
30 days. It is the key that correlates an Attempt to its logs and artifacts.
_Avoid_: session, job

**Agent Minutes**:
ATX's unit of billable agent work. A budget ceiling can be placed on a
Conversation, and reaching it ends the Conversation rather than failing it.
_Avoid_: runtime, duration, wall clock

**Nudge**:
A fresh Attempt carrying adjusted plan context, and one of only two sanctioned
interventions. A Nudge never edits the Recipe.
_Avoid_: retry, refine, correction

**Lesson**:
An improvement ATX extracts from an Attempt and applies automatically to later
ones. Lessons belong to ATX, never to this plugin. The `atx` CLI addresses them
as *knowledge items*.
_Avoid_: learning, feedback, training

**Leftover**:
An Exit Criterion that ATX reported as unmet. Leftovers are the unit of work this
plugin plans remediation for.
_Avoid_: gap, failure, remainder, TODO
