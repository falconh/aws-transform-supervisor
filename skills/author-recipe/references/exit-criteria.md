# Writing Exit Criteria

Exit Criteria are the statements of done that ATX validates against and reports on. Under
this plugin's design they are the only lever on Leftover quality (ADR 0003), so they get
disproportionate care.

## The test

**Could a script decide whether this criterion holds?** If yes, it is a good criterion. If
deciding requires taste, judgement or a conversation, rewrite it until it does not.

| ❌ Vague | ✅ Checkable |
|---|---|
| Code is modernised | `mvn clean install` completes with zero errors |
| Logging is migrated | No file imports `org.apache.log4j`; all logger fields are `org.slf4j.Logger` |
| Tests still pass | `mvn test` passes with no new failures against the base commit |
| UI is converted | Every `.frm` file has a corresponding `.razor` component |
| Dependencies updated | No `com.amazonaws` imports remain; `pom.xml` declares `software.amazon.awssdk` |
| Error handling improved | No `On Error` statements remain; all replaced with `try`/`catch` |

The right-hand column is what makes a Leftover report actionable: ATX can say *"criterion 3
unmet — 4 files still import log4j"*, and that is already a work item.

## Shape

Cover the migration's whole surface, not just its headline. A useful set usually spans:

1. **Builds** — the project compiles. Almost always criterion one.
2. **Tests** — the suite passes, or no *new* failures relative to the Base Commit.
3. **Completeness** — the old pattern is gone. Usually the most valuable, and the one most
   often forgotten. This is what catches code that compiles fine but was never migrated.
4. **Correctness of the new pattern** — the replacement is used the intended way, not merely
   present.
5. **Explicit non-goals** — what the migration deliberately leaves alone. Stops ATX
   wandering, and stops untouched code being reported as a Leftover.

## Worked example

For a Java 8 → 17 upgrade with an AWS SDK v1 → v2 migration:

```markdown
## Exit criteria

CRITICAL: The transformation is complete only when all of the following hold.

1. `mvn clean install` completes with zero errors under Java 17.
2. `mvn test` passes with no test failures that were not already failing at the base commit.
3. No source file imports from `com.amazonaws.*`. All AWS SDK imports are
   `software.amazon.awssdk.*`.
4. `pom.xml` declares no `aws-java-sdk-*` artifacts; the v2 BOM
   `software.amazon.awssdk:bom` is present.
5. All SDK clients are built with the v2 builder pattern (`S3Client.builder().build()`),
   not v1 constructors or `AmazonS3ClientBuilder`.
6. `maven.compiler.source` and `maven.compiler.target` are `17`.
7. NON-GOAL: do not change application logic, method signatures, or public API surface.
   Only the SDK call sites and build configuration change.
```

Seven criteria, each decidable, plus one explicit non-goal.

## Pair each criterion with a way to check it

Where a criterion maps to a command, say so in the Recipe. It gives ATX a deterministic way
to self-check, and it feeds Continual Learning:

```markdown
Verify criterion 3 with: `rg -l 'import com\.amazonaws' --type java` — expect no matches.
```

## Common mistakes

- **Only checking the build.** Code can compile perfectly and still be unmigrated. Always
  include at least one completeness criterion.
- **Criteria that restate the objective.** "Migrate to SDK v2" is the goal, not a criterion.
  The criterion is the observable state that proves it happened.
- **No non-goals.** Without them ATX may rewrite adjacent code, and the diff becomes
  unreviewable.
- **Too many.** Twenty criteria produce a Leftover report nobody reads. Aim for five to ten
  that genuinely span the surface.
