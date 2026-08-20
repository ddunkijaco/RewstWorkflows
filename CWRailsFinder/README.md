# CW Rails Finder (Rewst Workflow)

Reports every task in a Rewst tenant that calls a ConnectWise Manage "rails" endpoint instead of a
documented `v4_6_release` REST endpoint.

Rails endpoints are the CW Manage web UI's internal action processor. They are unversioned and
undocumented, and can change on any ConnectWise release.

The workflow reads from the Rewst GraphQL API only. It sends no requests to ConnectWise and writes
nothing.

## Import and run

Import `workflow-01a01f74-40ca-7df9-a0e5-aeb32b527c25_20260820_141207.bundle.json` under
**Automations → Workflows → Import**, then run it.

No configuration is required. Both action IDs are resolved by `ref` at run time and detection does
not match on a hostname, so the workflow runs unmodified in any tenant.

## Inputs

| Input | Type | Default | Description |
|---|---|---|---|
| `task_limit` | integer | `1000` | Maximum tasks fetched per action. If `truncated` returns `true`, increase and re-run. |

## Outputs

| Output | Type | Description |
|---|---|---|
| `findings_json` | string | JSON array of findings |
| `rails_tasks` | integer | Number of rails calls found |
| `workflows_affected` | integer | Number of distinct workflows containing them |
| `writes` | integer | How many findings write rather than read |
| `truncated` | boolean | `true` if a sweep reached `task_limit`, meaning results are incomplete |

The workflow does not format a report. `findings_json` is the output to consume.

## Finding fields

| Field | Description |
|---|---|
| `workflow` | Name of the workflow containing the task |
| `workflow_id` | Workflow ID |
| `task` | Task name |
| `task_id` | Task ID |
| `org_id` | Org that owns the workflow |
| `via_action` | `cw_manage.generic_request` or `core.http_request` |
| `cw_module` | CW module from the endpoint path, e.g. `Contact`, `System`, `Service` |
| `cw_action` | CW action class from the endpoint path, e.g. `UpdateCompanyTeamTabAction` |
| `method` | HTTP method configured on the task |
| `shape` | `absolute URL`, `relative traversal`, or `relative path` |
| `writes` | `true` unless `cw_action` begins with a read verb |
| `fails_loud` | `false` when the task has `require_2xx_status` disabled |
| `loops` | `true` when the task runs under `with items` |
| `endpoint` | The configured URL, whitespace-normalised |

`writes` is derived from `cw_action`, not from `method`. Every rails call is a POST, so the HTTP
method does not distinguish reads from writes. `GetMultilineDataAction` is a read;
`SaveServiceTicketFinanceBillingPodAction` is a write.

## Tasks

| Task | Action | Publishes |
|---|---|---|
| `START` | `core.noop` | — |
| `resolve_cw_action` | `rewst.generic_graph_request` | `cw_actions` |
| `fetch_cw_tasks` | `rewst.generic_graph_request` | `cw_tasks` |
| `resolve_http_action` | `rewst.generic_graph_request` | `http_actions` |
| `fetch_http_tasks` | `rewst.generic_graph_request` | `http_tasks` |
| `detect_rails` | `transforms.set_variable` | `audit` |

The two `resolve_*` tasks look up the action IDs for `cw_manage.generic_request` and
`core.http_request` by `ref`. The two `fetch_*` tasks retrieve every workflow task using those
actions, including each task's `input` and `with` block. `detect_rails` classifies the results and
publishes the `audit` object the outputs are drawn from.

Both actions are swept because a rails call can be issued through either.

## Detection

The CW PSA generic action prepends `https://<site>/v4_6_release/apis/3.0` to `url_path`. Reaching a
rails endpoint requires escaping that prefix, which takes one of two forms:

| Shape | Example |
|---|---|
| Absolute URL | `https://<site>/v4_6_release/services/system_io/actionprocessor/<Module>/<Action>.rails` |
| Relative traversal | `/../../services/system_io/actionprocessor/<Module>/<Action>.rails` |

A task is reported when its URL contains `.rails`, `system_io`, or `actionprocessor`. The module and
action class are then parsed from the path segments.
