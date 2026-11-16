---
description: Release rtlreviewbot-action as vX.Y.Z, in lockstep with the runtime's vX.Y.Z
argument-hint: <X.Y.Z>
---

Release this public action as version `$ARGUMENTS`, pointing it at the
rtlreviewbot runtime of the same version.

`$ARGUMENTS` must be a bare version of the form `X.Y.Z` (no `v`). If it is
missing or malformed, stop and ask for it. Below, `VERSION` is that
value.

This repo and the private runtime `Ride-The-Lightning/rtlreviewbot` are
versioned in lockstep: `rtlreviewbot-action@vVERSION` defaults
`runtime_ref` to the runtime's `vVERSION`. Release the runtime first
(`/release VERSION` in that repo).

Never move or re-create an existing tag.

## 1. Check the starting point

- The runtime tag exists. If this fails, stop: the action would point at
  a ref that cannot be checked out.
  ```bash
  gh api repos/Ride-The-Lightning/rtlreviewbot/git/ref/tags/vVERSION
  ```
- Tag `vVERSION` does not exist in this repo yet.
- The current branch is `main` and matches `origin/main`
  (`git pull --ff-only origin main`). Uncommitted changes are allowed
  only in `action.yml` and `README.md`; show them and carry them into
  the release commit.

## 2. Make the edits

- `action.yml`: set the `runtime_ref` input's `default` to `'vVERSION'`.
- `README.md`: set the same value in the `runtime_ref` row of the inputs
  table, and the tag in the usage example
  (`uses: Ride-The-Lightning/rtlreviewbot-action@vVERSION`). The README
  has drifted from `action.yml` before, so check both.
- New inputs: compare this `action.yml` with the runtime's
  `.github/actions/review/action.yml` at `vVERSION`:
  ```bash
  gh api "repos/Ride-The-Lightning/rtlreviewbot/contents/.github/actions/review/action.yml?ref=vVERSION" --jq .content | base64 -d
  ```
  For each input the runtime has and this action lacks, add the input
  with the same description plus "Requires runtime vVERSION+; earlier
  runtimes ignore it.", pass it through the `env:` block of the "Run
  rtlreviewbot" step under the runtime's env name, and add a row to the
  README inputs table. Inputs go through `env:`, never `${{ }}` inside a
  `run:` body.
- Keep this repo free of review logic: `action.yml` and
  `bootstrap-token.sh` only mint a token, check out the runtime and run
  it.

## 3. Commit, push, tag, release

Pushing `main` is the moment production changes for every consumer that
uses `rtlreviewbot-action@main`: they get the new runtime on their next
review. Say this, show the diff, and get a yes before pushing.

```bash
git commit -am "chore: bump runtime_ref default to vVERSION"   # append ", add <input> input" if any were added
git push origin main
git tag -a vVERSION -m "vVERSION"
git push origin vVERSION
gh release create vVERSION --verify-tag --title vVERSION --notes "Runtime default bumped to vVERSION.<one line per new input>"
```

## 4. Report

Give the commit, the tag and the release link. Then remind the user to:

- pin consumers that track a tag to `rtlreviewbot-action@vVERSION`;
- verify on a real RTL PR by running `/rtl review` and checking the run
  log shows the runtime checked out at `vVERSION`.
