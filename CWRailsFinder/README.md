# CW Rails Finder (Rewst Workflow)

Finds every task in your Rewst tenant that calls ConnectWise Manage **"rails"** endpoints — the
undocumented internal CW Manage action processor — instead of a supported `v4_6_release` REST
endpoint.

Rails endpoints are what the CW Manage web UI calls internally. They are unversioned,
undocumented, and carry no compatibility guarantee, so anything business-critical sitting on one
is a latent outage waiting for a ConnectWise release.

> **Read-only.** It queries the Rewst GraphQL API only. It never touches ConnectWise and writes
> nothing anywhere.

Built while auditing our own tenant — it found 21 rails calls across 9 workflows on the first run.

---

## Quickstart

1. **Import the workflow**
   - In Rewst, go to **Automations → Workflows → Import** and import
     `workflow-01a01f74-40ca-7df9-a0e5-aeb32b527c25_20260820_141207.bundle.json` from this folder.
2. **Run it.** Nothing to configure — no variables, no credentials, no ids to fill in. Both action
   ids are resolved by `ref` at run time and rails detection is hostname-agnostic, so it works in
   any tenant exactly as imported.
3. **Read `findings_json`** from the output.

---

## Inputs

| Input | Type | Default | Notes |
|---|---|---|---|
| `task_limit` | integer | `1000` | Max tasks fetched per action. If `truncated` comes back `true`, raise it and re-run. |

## Outputs

| Output | Notes |
|---|---|
| `findings_json` | JSON array of findings — the actual product |
| `rails_tasks` | Count of rails calls found |
| `workflows_affected` | Count of distinct workflows involved |
| `writes` | How many findings write rather than read |
| `truncated` | `true` if a sweep hit `task_limit`, so results are incomplete |

There is deliberately **no report or formatting task**. The workflow returns structured findings
and leaves presentation to you — feed `findings_json` into a template, an AppBuilder page, a ticket
note, or a Teams message as suits your shop.

Each finding looks like this:

```json
{
  "workflow": "General - Update Client Team Tab with Reactive",
  "workflow_id": "0196xxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "task": "add_team_tab",
  "task_id": "dfcc47eb0925434c887a97d372511512",
  "org_id": "your-org-uuid",
  "via_action": "cw_manage.generic_request",
  "cw_module": "Contact",
  "cw_action": "UpdateCompanyTeamTabAction",
  "method": "POST",
  "shape": "absolute URL",
  "writes": true,
  "fails_loud": true,
  "loops": true,
  "endpoint": "https://<your-site>/v4_6_release/services/system_io/actionprocessor/Contact/UpdateCompanyTeamTabAction.rails"
}
```

`fails_loud` is `false` when the task has `require_2xx_status` disabled. `loops` is `true` when the
task runs under `with items`, which matters because one failed item fails the whole task with no
per-item retry.

---

## How it works

```
START
  └─ resolve_cw_action          GraphQL: find cw_manage.generic_request by ref
     └─ fetch_cw_tasks          GraphQL: every workflowTask on that action, incl. input
        └─ resolve_http_action  GraphQL: find core.http_request by ref
           └─ fetch_http_tasks  GraphQL: every workflowTask on that action, incl. input
              └─ detect_rails   classify each task url; publishes the audit object
```

Both the CW PSA generic action and the core HTTP Request action are swept, because a rails call can
be made through either.

### How a rails call is recognised

The CW PSA generic action prepends `https://<your-site>/v4_6_release/apis/3.0` to whatever you put
in `url_path`. A rails call has to escape that prefix, which it does one of two ways:

| Shape | Example |
|---|---|
| Absolute URL | `https://<site>/v4_6_release/services/system_io/actionprocessor/<Module>/<Action>.rails` |
| Relative traversal | `/../../services/system_io/actionprocessor/<Module>/<Action>.rails` |

So any url containing **`.rails`**, **`system_io`** or **`actionprocessor`** is conclusive. No
hostname is matched, deliberately — that is what makes it portable.

The CW action class is pulled out of the path (`UpdateCompanyTeamTabAction`,
`GetMultilineDataAction_ProjectAuditTrail`) because that is what tells you what the call actually
does.

**Read vs write comes from the action class, not the HTTP method.** Every rails call is a POST, so
the method carries no signal at all. `GetMultilineDataAction` is a read;
`SaveServiceTicketFinanceBillingPodAction` is a write.

---

## Before you migrate anything off rails

Four things that cost us real time. Worth reading before you start swapping URLs.

1. **Missing pack coverage does not mean the endpoint is missing.** The CW PSA pack does not wrap
   company-team writes, opportunity types/stages/ratings, taxcodes or userDefinedFields — but the
   generic action calls those documented REST paths fine. "No pack action" is not the same as
   "no REST endpoint".

2. **Rails returns a different *envelope*, not just different data.** Rails gives
   `result.result.data.action.multilineRows[]` where each `.row` is a JSON **string** needing
   `json_parse`, plus your own request's `multilineUserParams` echoed back — which some consumers
   rely on to correlate a result back to its input. REST returns a plain array of objects. Anything
   consuming a rails result needs **rewriting**, not repointing. This is the single biggest reason
   a rails migration takes longer than it looks.

3. **Endpoints that look like drop-ins often are not.** Two we got wrong on a desk read:
   - `GET /system/audittrail` accepts `type=Agreement`, but for agreement **Addition** rows it
     returns the audit template with its placeholders **unsubstituted** — literally
     `{product_id}`, `{old_value}`. Those values exist only in the rails `Audit_values` blob, so
     that one is not migratable at all.
   - `POST /company/companies/{id}/teams` is **not** an upsert. A duplicate member/role pair
     returns `400 ObjectExists`.

4. **Exercise the replacement and diff it against the rails output before cutting over.** Ids,
   counts, ordering and field names all drift:
   - `/sales/opportunities/types` returns `description`, not `name`
   - `/marketing/campaigns` filters on `inactive`, not `inactiveFlag`
   - `/system/userDefinedFields` filters on a **numeric** `podId`, not the rails pod strings
   - `/system/locations` returns more rows than the rails location dropdown does

If you are replacing a dropdown feeding a Rewst form, note the rails `dataSelectionList` shape is
`{name, value, isDefault, tag}` where `value` is a **string** and `tag` carries real meaning
(Won/Lost/NA on opportunity statuses). Rebuild that shape rather than publishing REST rows
straight through.

---

## Rewst and Jinja gotchas hit while building this

Recorded because none of them are obvious and each one cost a run:

- **`rewst.generic_graph_request` rejects `raw_query` if you set `operation_type`.** You get
  "The operation you are trying to perform is not allowed. Please contact Rewst staff for
  assistance." — which reads like a permissions wall but is not one. Leave `operation_type`,
  `operation` and `fields` **null** and use `raw_query` plus `variable_values`. With that, it will
  happily query `actions`, `workflows`, `workflowTasks` and `organization`.
- **The result is unwrapped.** For a single-root query, `RESULT.result` is that root field's value
  directly, not `data`. So `workflowTasks { ... }` hands you the array straight.
- **`| d({})` does not catch `None`.** Jinja's `default()` only substitutes for *undefined*. A
  task's `with` is `None` when it does not loop, so `t['with'] | d({})` still yields `None` and
  `.get()` on it throws. Use `or {}`.
- **`with` is a Jinja keyword**, so the loop config is `t['with']`, never `t.with`.
- **`split()` is a string method, not a filter.** `x | split('/')` fails; `x.split('/')` works.
- **A `set` inside a `for` loop does not escape the loop scope.** A prefix-matching loop will
  silently always return its fallback — which is how the first cut of this reported every finding
  as a write. Use a single expression, or `namespace()`.
- **A task with no inbound transitions is an entry point, not a disabled task.** Disconnecting a
  task to stop it firing turns it into a second START and it runs anyway. Delete it instead.

---

## Licence

Do what you like with it. No warranty — it is read-only, but read it yourself before running it in
your own tenant.
