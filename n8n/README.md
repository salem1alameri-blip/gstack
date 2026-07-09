# CI → n8n integration

GitHub Actions runs the free test gate on every push and pull request. When it
finishes it POSTs the result to an [n8n](https://n8n.io) instance, which owns
everything downstream: Slack / email alerts, opening an issue on failure,
triggering a deploy on success, and so on.

CI stays authoritative on its own. If n8n is down, the webhook call is retried
a few times and then skipped with a warning; a green build never turns red
because of an n8n outage.

```
push / PR ──▶ GitHub Actions (bun test)
                     │  (always, pass or fail)
                     ▼
              POST  N8N_WEBHOOK_URL  ──▶  n8n Webhook
                                              │
                                     ┌────────┴────────┐
                                  passed?            failed?
                                     │                 │
                              Slack ✅ / deploy   Slack ❌ / open issue
```

## Files

- `.github/workflows/ci-n8n.yml` — the CI job + the `notify-n8n` step.
- `n8n/gstack-ci-notify.workflow.json` — importable n8n workflow.

## Setup (one time)

1. **Import the workflow into n8n.** In n8n: *Workflows → Import from File →*
   pick `n8n/gstack-ci-notify.workflow.json`.
2. **Copy the webhook URL.** Open the **CI Webhook** node and copy its
   **Production URL** (looks like `https://<your-n8n-host>/webhook/gstack-ci`).
3. **Add the GitHub secret.** Repo → *Settings → Secrets and variables →
   Actions → New repository secret*:
   - Name: `N8N_WEBHOOK_URL`
   - Value: the production URL from step 2.
4. **Wire your notifiers.** In the imported workflow, open *Notify success* and
   *Notify failure*, attach your Slack (or email/Discord) credential and pick a
   channel. Swap these nodes for whatever downstream action you want.
5. **Activate the workflow** in n8n (toggle, top-right). The production webhook
   URL only responds when the workflow is active.

## Payload

The action sends this JSON body (available in n8n under `{{ $json.body }}`):

| Field        | Example                                              |
|--------------|------------------------------------------------------|
| `event`      | `push` / `pull_request`                              |
| `status`     | `success` / `failure` / `cancelled`                  |
| `repository` | `salem1alameri-blip/gstack`                          |
| `branch`     | `main`                                               |
| `sha`        | full commit SHA                                      |
| `short_sha`  | first 7 chars                                        |
| `actor`      | GitHub username that triggered the run               |
| `run_url`    | direct link to the Actions run                       |
| `pr_number`  | PR number (empty on plain pushes)                    |

## Extending it

The `CI passed?` branch is the natural place to grow:

- **Deploy on green** for `main` only — add an *IF* node checking
  `{{ $json.body.branch === "main" }}` before your deploy step.
- **Open a GitHub issue on failure** — add a *GitHub* node on the failure
  branch using the `run_url` in the body.
- **Fan out to multiple channels** — add more nodes off either branch.

## Notes

- Requires the `N8N_WEBHOOK_URL` secret. Without it the `notify-n8n` step logs
  a skip and passes, so forks and contributors without the secret aren't
  blocked.
- The CI runner uses `ubuntu-latest`. To match the rest of this repo's
  workflows you can switch it to `ubicloud-standard-8` in `ci-n8n.yml`.
