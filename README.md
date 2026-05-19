# policy-eval

A minimal policy-as-code engine written in the
[Capa](https://github.com/nelsonduarte/capa-language) language.
Takes a JSON-encoded policy (a tree of rules whose `when`
clauses can nest arbitrarily deep) and a JSON subject document
(a list of cloud resources, in the demo), and emits a markdown
verdict report + a JSON summary.

**Why this exists.** Policy-as-code (OPA / Sentinel / Conftest)
is the way 2026 enterprises enforce cloud governance, IaC
guardrails, and now AI-agent boundaries. The pain isn't the
concept, it's the operational glue: writing a policy is
straightforward, but a small team often wants something tiny,
auditable, and trustable instead of bringing in a 50MB OPA
distribution.

`policy-eval` shows the smallest defensible shape of such an
engine: ~700 lines of [Capa](https://github.com/nelsonduarte/capa-language),
5 files, the entire evaluator pure, no network, no daemon, one
deterministic pass over (rules × resources).

It is also a demo of a feature the previous Capa demos didn't
exercise: **recursive sum types**. The policy AST is a
`Condition` whose `AllOf` / `AnyOf` / `NotMatch` variants
carry payloads typed `List<Condition>` and `Condition`. The
evaluator is a textbook tree walk that runs on top of it.

## The problem

Cloud accounts drift. A misconfigured S3 bucket goes public,
an IAM role accumulates AdministratorAccess, a Terraform module
forgets the default security group. Catching these at scale
needs rules-as-data: write the rule once, run it across every
resource in every account on every PR.

`policy-eval` is the smallest possible take. Three inputs (a
policy file, a subject file, an optional fail threshold), two
outputs (markdown + JSON), exit code reflects whether the gate
passed.

### Policy schema

A policy is a `name` plus a list of `rules`. Each rule has
`name`, `severity`, `when` (a `Condition`), and `message`.

Conditions are a small AST:

| Form                                            | Meaning                                    |
|-------------------------------------------------|--------------------------------------------|
| `{"path": "a.b", "equals": x}`                  | drill into `a.b`, compare to `x`           |
| `{"path": "a.b", "not_equals": x}`              | drill, compare, negate                     |
| `{"path": "a.b", "in": [x, y]}`                 | value is in the list                       |
| `{"path": "a.b", "contains": x}`                | value is a list containing `x`             |
| `{"path": "a.b", "exists": true}`               | path resolves (or doesn't, with `false`)   |
| `{"all_of": [cond, cond, ...]}`                 | every sub-condition matches                |
| `{"any_of": [cond, cond, ...]}`                 | at least one sub-condition matches         |
| `{"not": cond}`                                 | the sub-condition does not match           |

Compositors (`all_of` / `any_of` / `not`) nest freely. The
demo policy includes one rule
([`test-not-and-anyof`](data/policy.json#L51)) that exercises
the recursive case directly:

```json
{
  "any_of": [
    {"all_of": [
      {"path": "type", "equals": "rds_instance"},
      {"not": {"path": "config.encrypted", "equals": true}}
    ]},
    {"path": "config.backups", "equals": false}
  ]
}
```

### Subject schema

```json
{
  "environment": "production",
  "snapshot_at": "2026-05-19T08:00:00Z",
  "resources": [
    {"type": "s3_bucket", "name": "data-lake", "config": { ... }},
    {"type": "iam_role", "name": "ci-deploy",  "config": { ... }},
    ...
  ]
}
```

The evaluator drills into the raw JSON envelope of each
resource via dotted paths, so the policy can address any
nested field without the program committing to a fixed schema.

## Run it

Prerequisites: Capa >= 0.8.4 (commit `4c4511b` or newer; the
submodule-resolves-root-siblings loader fix is required).

From the repo root:

```bash
capa --run policy_eval.capa -- data/subject.json
```

Expected output:

```
[INFO] loading policy data/policy.json
[INFO] policy 'default-cloud-baseline': 6 rule(s)
[INFO] loading subject data/subject.json
[INFO] subject 'production': 8 resource(s)
[WARN] [no-public-s3]            s3_bucket/public-assets     Critical: S3 bucket is publicly accessible (ACL allows public read/write)
[WARN] [s3-encryption-required]  s3_bucket/public-assets     High:     S3 bucket is missing server-side encryption
[WARN] [s3-encryption-required]  s3_bucket/logs-archive      High:     S3 bucket is missing server-side encryption
[WARN] [no-admin-iam]            iam_role/ci-deploy          Critical: IAM role grants AdministratorAccess (use a scoped policy)
[WARN] [no-default-sg]           ec2_instance/web-01         Medium:   EC2 instance uses the default security group
[WARN] [no-public-ec2]           ec2_instance/web-01         High:     EC2 instance has a public IP (use NAT or a load balancer)
[WARN] [test-not-and-anyof]      rds_instance/billing-db     Low:      RDS instance must be encrypted and have backups enabled
[INFO] findings: 2 critical, 3 high, 1 medium, 1 low, 0 info
[INFO] wrote ./policy-report-2026-05-19.md
[INFO] wrote ./policy-report-2026-05-19.json
[WARN] policy gate FAILED: at least one finding >= High
policy-eval: FAILING
```

The exit code is non-zero whenever any finding meets or
exceeds the `--fail-threshold` (default `High`).

### Options

```
Usage: policy-eval [FLAGS] [OPTIONS] SUBJECT

Arguments:
  SUBJECT                    path to the subject JSON document (under data/)

Flags:
  -v, --verbose              enable DEBUG-level logging

Options:
  -p, --policy STR           path to policy JSON (default: data/policy.json)
  -o, --output-dir STR       directory for report files (default: cwd)
  -f, --fail-threshold STR   exit non-zero when any finding >= this severity
                             (default: High)
```

## The audit story

`capa --manifest policy_eval.capa` shows the capability shape
of every function:

```
parse_condition          pure        the recursive parser
eval_condition           pure        the recursive evaluator
lookup_path              pure
json_equals              pure
run_policy               pure        drives evaluation across (rules x resources)
render_markdown          pure
render_json              pure
summarise                pure
has_failing              pure
parse_policy             [Fs]
parse_subject            [Fs]
write_text               [Fs]
run                      [Logger, Fs, Fs]
main                     [Stdio, Fs, Env, Clock]
```

The recursive parser and the recursive evaluator are both
provably pure. They can't touch disk, can't print, can't
read the clock. A diff that added `stdio.println("debug")`
inside `eval_condition` would fail to compile because the
function takes no `Stdio` parameter.

The user-defined `Logger` capability hides `Stdio` past the
factory: `run` declares `[Logger, Fs, Fs]`, not
`[Stdio, Fs, Fs]`. The two `Fs`s are attenuated to disjoint
prefixes (`data/` for reads, the output directory for writes)
so no parser can write a file and no writer can read one.

## Repo layout

```
.
├── policy_eval.capa     entry point: CLI, attenuated Fs split, orchestration
├── model.capa           types: Condition (recursive!), Rule, Policy,
│                                Subject, Resource, Finding, Severity
├── parse.capa           JSON -> Condition (recursive), Policy, Subject
├── evaluator.capa       tree-walk eval; lookup_path; json_equals; run_policy
├── render.capa          pure renderers: markdown + JSON
├── data/
│   ├── policy.json      6 rules including a nested any_of(all_of(not(...))) sample
│   └── subject.json     8 cloud resources, 7 of them trigger at least one rule
├── libraries/           vendored Capa seed libraries (capa_cli, capa_datetime, capa_log)
├── LICENSE              MIT
└── README.md
```

## What this exercises in Capa (that earlier demos didn't)

- **Recursive sum types**. `Condition` is defined with
  `NotMatch(Condition)`, `AllOf(List<Condition>)`,
  `AnyOf(List<Condition>)` - the variant payloads reference
  the type being defined. The previous demos
  ([`audit-trail-reporter`](https://github.com/nelsonduarte/audit-trail-reporter),
  [`sbom-watch`](https://github.com/nelsonduarte/sbom-watch))
  used only flat sum types.
- **Recursive function calls**. `parse_condition` calls itself
  through `parse_condition_list`; `eval_condition` calls itself
  through `NotMatch` / `AllOf` / `AnyOf`. Both confirmed
  working in Capa without any opt-in flag.
- **Tree-walk interpreter pattern**. The shape of the program
  is genuinely different from the per-record classification of
  the AML reporter or the cross-source matching of the SBOM
  watcher: a small DSL is parsed into an AST and evaluated
  recursively against data.
- **Heavy `JsonValue` path navigation**. `lookup_path` drills
  into nested objects via `as_object().get()` chains, and the
  parser distinguishes leaf vs combinator nodes by which
  keys are present.

Otherwise the demo uses the same Capa surface as the previous
ones: built-in caps (`Fs`, `Stdio`, `Env`, `Clock`),
user-defined `Logger` via `capa_log`, capability attenuation
(`Fs.restrict_to`), sum types with payloads, `?` for error
propagation, three vendored seed libraries.

## Limitations

This is a v0.1 demo. Out of scope:

- **No streaming / no Rego compatibility**. The policy and
  subject are read whole. A real OPA replacement would
  support streaming evaluation and Rego syntax.
- **Leaf-only JSON equality**. `json_equals` compares scalars
  (Bool, Num, Str, Null). Nested arrays and objects don't
  participate. The demo policy never asks for that.
- **No type coercion**. `{"equals": "true"}` (string) does
  not match `true` (bool); the runtime values must match
  exactly.
- **No PURL-style ecosystem awareness**. The evaluator is
  type-tag agnostic; it doesn't know that `s3_bucket` and
  `S3Bucket` are the same thing.
- **TOCTOU race on `Fs.restrict_to`**. The Capa runtime
  canonicalises paths (via `os.path.realpath`) before
  comparing, so `data/../etc/passwd` and symlinks pointing
  outside the prefix are denied. A symlink swap between the
  `allows()` check and the underlying `open()` is still
  possible; closing it requires open-at-dirfd semantics.

## Companion demos

- [audit-trail-reporter](https://github.com/nelsonduarte/audit-trail-reporter)
  - AML risk classifier; classification-aggregation shape
- [sbom-watch](https://github.com/nelsonduarte/sbom-watch)
  - SBOM operationaliser; cross-source matching shape
- **policy-eval** (this repo) - tree-walk interpreter shape

Three different program shapes, the same language; each one
surfaced a different Capa muscle.

## License

MIT. See [LICENSE](./LICENSE).
