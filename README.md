# fpv-release-ci

Release automation for [`bob9/FPVTracksideCore`](https://github.com/bob9/FPVTracksideCore),
kept in a **separate repo on purpose** so none of it can ever land in a pull
request against the source (or its `uewepuep` upstream). The source repo stays
100% CI-free — there is **no file in `FPVTracksideCore`** and **no external
service**.

## How it triggers ("every check-in")

GitHub Actions can't see a push in another repo without either a file in that
repo or an external relay. To keep the source clean, this repo **polls** the
source fork on a schedule instead:

- Every ~10 min the `check` job reads the tip of `master` on the fork.
- If that commit hasn't been released yet (no `auto-<shortsha>` release exists),
  it builds Windows / macOS / Linux and publishes the release.

Because you commit straight to `master`, "new master commit" == "new release".
Latency is the poll interval (GitHub's scheduler can lag a bit). You can also
trigger a build immediately from the **Actions → Run workflow** button.

## Where the release goes

Always into **`bob9/FPVTracksideCore/releases`** (set by `SOURCE_REPO`). This
repo only runs the build; it never holds artifacts. Releases are tagged
`auto-<shortsha>` and marked **pre-release**.

## One-time setup (~5 min)

1. **Create the repo and push this skeleton:**
   ```bash
   cd /Users/glenn.pringle/git/fpv-release-ci
   git init && git add . && git commit -m "Add polling multi-platform release workflow"
   gh repo create bob9/fpv-release-ci --private --source=. --remote=origin --push
   ```

2. **Add a Personal Access Token as a secret** so the workflow can read the fork
   and create releases in it:
   - Token with **`repo`** scope (classic), or fine-grained with **Contents:
     Read & Write** on `bob9/FPVTracksideCore`.
   - `gh secret set FORK_PAT --repo bob9/fpv-release-ci`  (paste when prompted)

That's it — polling starts on the schedule. No further action per release.

## On-demand build of any ref

```bash
gh workflow run release.yml --repo bob9/fpv-release-ci \
  -f ref=some-branch-or-sha \
  -f release_tag=v2.75.2     # optional; blank = auto-<shortsha>
```

## Tuning / caveats

- **Poll interval:** change the `cron` in `release.yml` (`*/10 * * * *`). Faster
  polling = lower latency but more (cheap) `check` runs.
- **True instant on push** would need a webhook on `FPVTracksideCore` → a small
  relay (Cloudflare Worker / Lambda) → this repo's `repository_dispatch`. The
  webhook is a repo *setting* (not a tracked file, so it won't leak into a PR),
  but the relay is infra you'd host. Polling avoids all of that.
- **Watching more than `master`:** the poll only checks `WATCH_BRANCH`. Literal
  "every push on every branch" isn't practical via polling.
- **macOS** builds `osx-arm64` only and is **unsigned** (Gatekeeper warnings);
  add an `osx-x64` matrix row for Intel, and signing/notarization if you
  distribute widely. Output is a published folder, not a `.app` bundle.
- **Windows** (`net9.0-windows`) is why a `windows-latest` runner is required —
  it can't be cross-built from macOS.
- First run: confirm each `dotnet publish` succeeds; per-project flags
  (single-file, trimming, content files) may need adjustment.
