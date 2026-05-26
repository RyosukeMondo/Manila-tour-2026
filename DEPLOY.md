# Deployment Notes

## Live URL
https://ryosukemondo.github.io/Manila-tour-2026/

## Deployment

Static site auto-deployed by `.github/workflows/pages.yml` on push to
**`claude/manila-business-tour-OryBY`**. The workflow uploads `index.html`
(plus any sibling assets) to GitHub Pages via `actions/deploy-pages@v4`.

To deploy a change:

```bash
git checkout claude/manila-business-tour-OryBY
# edit index.html ...
git commit -am "update"
git push origin claude/manila-business-tour-OryBY
```

The Action will run in ~30 seconds and the site goes live within ~1 minute
of the workflow finishing.

---

## Why the first deploy took so long (post-mortem)

The first 5 workflow runs all failed with no log output (the job exited
after 2 seconds with `steps: []`), which made diagnosis hard. Two factors
compounded:

### Root cause: environment branch protection rule

When GitHub Pages was first enabled with **Source = GitHub Actions**, GitHub
auto-created a `github-pages` deployment environment **and locked it to
whichever branch deployed first** — in our case `claude/manila-business-tour-OryBY`.

Subsequent pushes to `main` tried to deploy to the same `github-pages`
environment, hit the protection rule, and were **rejected before any step
ran** — hence the empty `steps: []` and the 2-second runtime. This is not
shown as a normal step failure; it looks like the workflow silently dies.

Verified via:

```bash
curl -sH "Accept: application/vnd.github+json" \
  https://api.github.com/repos/RyosukeMondo/Manila-tour-2026/environments/github-pages/deployment-branch-policies
# -> only "claude/manila-business-tour-OryBY" was in the allow-list
```

### Secondary cause: diagnosis was slow

- `github-pages` workflow failures show **no inline error** in the GitHub UI
  step list — you have to look at the *environment protection rules* page.
- Unauthenticated GitHub API has a 60 req/hour rate limit per IP, which
  burned out during polling.
- First-deploy DNS/CDN propagation on a new Pages site adds 1–5 minutes
  even after a green workflow.

---

## Prevention checklist

If this repo is ever cloned or Pages is re-enabled, do these in order to
avoid the same trap:

1. **Decide the deploy branch first.** Push the workflow file from the
   branch you actually want to deploy from. The branch that triggers the
   first successful Pages deploy locks the environment policy.

2. **Verify the environment allow-list immediately after first deploy:**
   - Settings → Environments → `github-pages` → Deployment branches
   - Should match the branch in `on.push.branches` of `pages.yml`.
   - If you later switch deploy branches, **update this list** before pushing.

3. **Keep `on.push.branches` and the env allow-list in sync.** Mismatched
   entries fail silently with empty `steps`.

4. **When a Pages deploy "fails with no logs":** check
   `https://api.github.com/repos/<owner>/<repo>/actions/runs/<run_id>/jobs`
   — if `"steps": []` and runtime < 5s, it's an environment/permission
   rejection, not a workflow bug.

5. **Repo Settings → Actions → General → Workflow permissions** should be
   "Read and write permissions" (required for `pages: write` token scope).

6. **Don't poll the GitHub API unauthenticated.** Use the bundled MCP
   GitHub tools or `gh` CLI with a token, otherwise the 60 req/hour limit
   trips fast.
