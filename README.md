# Publish to kan.fyi

Publish a directory of static files to your [kan.fyi](https://kan.fyi) site
from CI. The publish is judged by its **outcome** — the exit code says
whether the site is actually live, not merely whether an upload was accepted.

## GitHub Actions

```yaml
- uses: ifarobi/kan-publish-action@v1
  with:
    token: ${{ secrets.KANFYI_TOKEN }}
    site: abc12345          # your site's eight-character ID
    dir: ./dist
```

Setup, once:

1. Create the site in your [dashboard](https://app.kan.fyi) — CI publishes
   to an existing site, it never creates one.
2. Mint a token at **Settings → Access tokens** and save it as the
   `KANFYI_TOKEN` repository secret. The token is shown exactly once, and it
   can publish to **every** site on your account — treat it like a password.
   Revoke it from the same page if it ever leaks.

The action's `url` output carries the live URL for later steps, and `outcome`
carries what became of the publish (`live`, `held`, `pending`, `failed`).

The step fails on anything but `live`. To handle a held or pending publish
without failing the job — a new account's first publish is always held — let
the step continue and branch on the outcome:

```yaml
- uses: ifarobi/kan-publish-action@v1
  id: publish
  continue-on-error: true
  with:
    token: ${{ secrets.KANFYI_TOKEN }}
    site: abc12345
    dir: ./dist
- if: steps.publish.outputs.outcome == 'held'
  run: echo "Under review — nothing further is needed."
- if: steps.publish.outputs.outcome == 'failed'
  run: exit 1
```

## Any other CI (GitLab, Jenkins, a laptop)

Download the `kan` binary for your platform from
[Releases](https://github.com/ifarobi/kan-publish-action/releases), verify
it against `checksums.txt`, then:

```sh
export KANFYI_TOKEN=...   # from Settings → Access tokens
kan publish ./dist --site abc12345
```

## Static sites only

kan.fyi serves files — nothing runs on the server. There is no Node runtime,
no functions, no database. Publish your generator's **built output**, never
the project source. `kan publish` refuses a directory with `package.json` or
`node_modules` at its root: that's a source tree, not built output.

| Generator | Publish this |
|-----------|--------------|
| Next.js with `output: 'export'` | `./out` |
| Astro, Vite | `./dist` |
| SvelteKit (`adapter-static`) | `./build` |
| Hugo | `./public` |
| Eleventy | `./_site` |
| Plain HTML/CSS/JS | the folder itself |

Not supported: SSR, API routes, middleware, image-optimization endpoints —
anything that needs a server. For Next.js, three config lines matter:

```js
// next.config.js
module.exports = {
  output: 'export',
  trailingSlash: true,           // /about resolves as /about/index.html here
  images: { unoptimized: true }, // there is no image-optimization server
}
```

## Exit codes — the contract

| Code | Meaning |
|------|---------|
| 0 | The publish outcome is **live**. The URL is printed to stdout (and nothing else is). |
| 1 | The publish **failed** or the site is not being served — the reason is on stderr. Also any usage or API error. |
| 2 | The site is **held** for review. Expected for a new account's first publish; nothing further is needed. Do not retry — republishing will not clear a review. |
| 3 | Still **pending** when the wait ran out. Check later with `kan status --site <id>`; do not republish, that spends one of your daily publishes. |

`kan status --site <id>` re-reads the outcome any time, with the same exit
codes and the same stdout shape.

## Good to know

- Directories are validated locally before upload with the same rules the
  server enforces: no `.git`/`.env*`/credentials files, no executables,
  no symlinks, at most 2000 files, 30 MB upload cap (25 MB per site).
  Pre-compressed siblings (`.gz`/`.br`/`.zst`) are dropped and regenerated
  at publish time.
- A release replaces the site's content atomically. Visibility (public,
  link-only, private) is a separate setting the CLI never touches — set it
  in the dashboard.
- Rate limits per account: 100 publishes per rolling 24h, 3 uploads in
  flight, 10 sites.
