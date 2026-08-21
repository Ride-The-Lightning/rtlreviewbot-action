# rtlreviewbot-action

Public entry point for [**rtlreviewbot**](https://github.com/Ride-The-Lightning/rtlreviewbot), an AI PR-review GitHub App.

The review logic lives in a **private** repo. A public consumer repo can't `uses:` a private repo's action — GitHub's org Actions-access setting only extends a private action to private/internal org repos, never to public ones. This thin **public** action is resolvable from any consumer (public or private); at run time it fetches the private runtime and executes it.

It contains **no review logic** — only `action.yml` (the composite entry point) and `bootstrap-token.sh` (a least-privilege token minter). See [rtlreviewbot#17](https://github.com/Ride-The-Lightning/rtlreviewbot/issues/17) for the design.

## How it works

1. **`bootstrap-token.sh`** mints a short-lived GitHub App installation token scoped to **`contents:read` on `rtlreviewbot` only**. It discovers the installation from the repo (`GET /repos/Ride-The-Lightning/rtlreviewbot/installation`) — the org installation id is not hardcoded.
2. **`actions/checkout`** pulls the private `rtlreviewbot` runtime at `runtime_ref` using that token.
3. The CLI is installed and the runtime's existing `scripts/run-review.sh` is exec'd — identical to running the in-repo private action.

The token the consumer's `installation_id` produces (for posting reviews on the consumer PR) is minted separately inside the runtime, exactly as before. The bootstrap token is a distinct, narrower token used only to read the runtime.

For the full flow — the two-token model, the run sequence, and why the runtime needs no changes — see [`docs/architecture.md`](docs/architecture.md).

## Usage

In a consumer repo, point the shim's `uses:` at this repo:

```yaml
- uses: Ride-The-Lightning/rtlreviewbot-action@v0.10.0
  with:
    event_name:      ${{ github.event_name }}
    event_action:    ${{ github.event.action }}
    repo:            ${{ github.repository }}
    pr_number:       ${{ github.event.issue.number || github.event.pull_request.number }}
    actor:           ${{ github.event.sender.login }}
    comment_body:    ${{ github.event.comment.body }}
    comment_id:      ${{ github.event.comment.id }}
    installation_id: REPLACE_WITH_INSTALLATION_ID
    app_id:                  ${{ secrets.GATEWAY_APP_ID }}
    private_key:             ${{ secrets.GATEWAY_PRIVATE_KEY }}
    anthropic_api_key:       ${{ secrets.ANTHROPIC_API_KEY }}
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
```

The full shim (triggers, permissions, job) is in the runtime repo at [`templates/rtlreviewbot.yml`](https://github.com/Ride-The-Lightning/rtlreviewbot/blob/main/templates/rtlreviewbot.yml). The only difference vs. the legacy in-repo action is the `uses:` line above.

### Inputs

Same surface as the private action, plus `runtime_ref`:

| Input | Required | Default | Notes |
|-------|----------|---------|-------|
| `exclude_paths` | no | `''` | Comma-separated globs for committed build artifacts (e.g. `frontend/**`) removed from the reviewed diff before the size ceiling is measured. Runtime v0.13.0+. |
| `runtime_ref` | no | `v0.13.0` | Ref of the private runtime to run. Defaults to the release this action was tagged in lockstep with. Override only to test an unreleased runtime. |

All other inputs (`event_name`, `repo`, `pr_number`, `actor`, `comment_*`, `installation_id`, `app_id`, `private_key`, `anthropic_api_key`, `claude_code_oauth_token`, `review_mode`, `exclude_paths`) match the private action one-for-one.

## Versioning

Two version axes, pinned in lockstep: `rtlreviewbot-action@vX.Y.Z` defaults `runtime_ref` to the runtime's `vX.Y.Z`. Pin a consumer to a tag and both the entry point and the runtime are pinned together.

## Prerequisites

- The rtlreviewbot App must have **`contents:read`** on the `rtlreviewbot` repo (i.e. the repo is in the App installation's selection). *(Confirmed: the Ride-The-Lightning installation is set to all-repositories with `contents:read`.)*
- The App must be installed on the consumer repo so its `installation_id` exists.

## Security

The action executes private code fetched at run time using App credentials the consumer already supplies and already runs today via the legacy private action — no new code-execution exposure. The bootstrap token is the App's own, minted from those same credentials, scoped to `contents:read` on a single repo, and masked in logs. The only public surface here is `action.yml` + `bootstrap-token.sh`; both are kept logic-free (mint + checkout only).

## License

[MIT](LICENSE) © 2026 Shahana Farooqui
