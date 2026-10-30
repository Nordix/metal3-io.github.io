# Metal3 Website

Source of <https://metal3.io>, a Jekyll site built with the `github-pages`
gem and served by GitHub Pages. Content rules:
[GUIDELINES.md](GUIDELINES.md). Contribution rules: the
[Metal3 community guide](https://github.com/metal3-io/community/blob/main/CONTRIBUTING.md).

This file holds only what you cannot cheaply infer from the code. Where it
disagrees with the code or the Makefile, trust those and report the
mismatch.

## AI-assisted contributions

Metal3 follows the CNCF and
[Linux Foundation generative AI policy](https://www.linuxfoundation.org/legal/generative-ai).

- AI is a tool. The human who opens a PR is its author and is responsible
  for every line of it, including licensing and third-party content.
- Review is a conversation between humans. A PR where an agent or bot
  answers reviewers will be closed. Agents may draft changes; the
  contributor reads them, understands them, and replies personally.

- Keep changes surgical. The smallest complete change that solves the
  stated problem wins. Every added file, function, flag, dependency or
  test helper must trace to that problem; anything that does not is
  bloat and gets removed. Prefer reusing existing code over new code.
  Report unrelated findings instead of fixing them in the same PR.

## Repository map

- PRs target the `source` branch. `master` holds the built site. It is
  generated and pushed by `.github/workflows/jekyll.yml` on every push to
  `source` and nightly. Never edit `master` or open PRs against it.
- Blog posts: `_posts/YYYY-MM-DD-title.md`. Copy the front matter from a
  recent post. FAQ entries: `_faqs/`. Pages: the root `*.html` files.
  Templates: `_layouts/`, `_includes/`, `_sass/`.
- Post images go in `assets/<post file name without .md>/` and are
  referenced as `/assets/<that dir>/<file>`.

## Testing

- `make lint` runs every `hack/` check in a container (`CONTAINER_RUNTIME`,
  default `podman`). CI jobs:
  [Prow](https://github.com/metal3-io/project-infra/blob/main/prow/config/jobs/metal3-io/metal3-io.github.io.yaml)
  and `.github/workflows/`. The PR link check (lychee) runs only in CI.
- New technical terms go in `.cspell-config.json`. Links that are valid
  but fail checking go in `.lycheeignore`.
- Before finishing, run `make lint`. All checks must pass.
- `make serve` previews the site on `HOST_PORT` (default 4000), but it
  needs a TTY and runs until stopped. Leave previewing to the user.
- `_site/` is local build output and is gitignored.

## Invariants

These apply to writing and to reviewing changes.

- **Published URLs.** Post URLs come from the file name
  (`permalink` in `_config.yml`). Never rename or re-date a published
  post. If you must, add `redirect_from` to its front matter
  (`jekyll-redirect-from` is enabled).
- **Images.** Keep them small. The pre-commit `check-added-large-files`
  hook rejects files over 500 KB.
- **Security.** Do not add third-party scripts, trackers or embeds
  without maintainer agreement. No secrets in content or workflows.
  Keep workflow `permissions` minimal. `jekyll.yml` is the only workflow
  that needs `contents: write`.
- **Pinning.** Pin new external references: GitHub Actions by commit SHA,
  container images by digest, external scripts by an SRI `integrity`
  hash.

## External references

- [Metal3 Book](https://book.metal3.io/)
- [metal3-docs](https://github.com/metal3-io/metal3-docs)
