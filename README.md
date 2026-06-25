# cascade-example-callbacks

A cascade fleet example that exercises per-callback security: secrets opt-in,
least-privilege permissions, OIDC id-token propagation, dependency ordering, and
the retry wrapper. Each callback's callee reports what it received so the
scenario suite can assert the posture took effect at the callee, not just that
it was emitted.

## Callbacks under test

The manifest declares build callbacks with distinct security postures, each
backed by a callee reusable workflow that asserts and reports its posture:

| Callback | Posture | Callee |
| --- | --- | --- |
| `build-secret` | opts in to an explicit secret (`CALLBACK_SHARED_SECRET`) | `callee-secret.yaml` asserts the secret arrived non-empty |
| `build-perms` | explicit least-privilege `permissions` | `callee-perms.yaml` reports the capped token scopes |
| `build-oidc` | `id-token: write` for OIDC | `callee-oidc.yaml` mints an OIDC token and asserts it was issued |
| `build-needs` | `depends_on: build-perms` + `retries: 2` | `callee-needs.yaml` reports it ran after its dependency |
| `build-withheld` | declares a secret that is intentionally not set | `callee-withheld.yaml` refuses and fails (registered negative) |

A `deploy-app` deploy and two environments (`staging`, `prod`) give the repo a
valid promote chain; the postures above ride on the build callbacks, which fire
on every orchestrate run.

## Scenario suite

`scenario-suite.yaml` merges a source change to drive an orchestrate run, then
reads that run's jobs and asserts each callee's conclusion and the needs
ordering and retry-wrapper presence. It then dispatches `callee-withheld.yaml`
standalone and registers that run as an expected failure. Every run the suite
causes is recorded in the fleet ledger, and the final reconcile job fails on any
unregistered run.
