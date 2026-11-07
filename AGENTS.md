# Agent instructions

Use ASD-STE100 Simplified Technical English by default. Keep changes scoped and preserve the lab instructions. Do not run publishing workflows, merge, or deploy without explicit user authorization.

## Private data and publication checks

- Treat commits, PR text, issue text, screenshots, and release files as public-ready, even when this repository is private.
- Never commit credentials, tokens, cookies, authenticated browser state, private keys, real scans, customer data, database exports, or unredacted logs. Use synthetic test data and placeholder values in example configuration.
- Keep local evidence under ignored `output/` or `artifacts/` directories. These folders are not safe places for secrets to be shared: inspect and redact each file before attaching it anywhere.
- Use repository-relative paths in committed documentation and PR descriptions. Do not include local usernames, home-directory paths, private hostnames, or links to private evidence in public text.
- Before staging, inspect `git status --short` and select explicit paths. Before committing, inspect `git diff --cached --name-status`, review the staged diff privately, and run `git diff --cached --check`.
- Run `gitleaks git --staged --redact --no-banner` before committing and `gitleaks git --log-opts="origin/main..HEAD" --redact --no-banner` before pushing (use the actual PR base if different). If Gitleaks is unavailable, install it from the official project or report the missing check; do not claim a clean scan. Keep scan output local and redacted.
- Review scanner matches before proceeding. A public identifier or synthetic fixture can be a false positive; document that reason without copying a sensitive value. Do not add broad exclusions or disable checks to get a passing result.
- `.gitignore` does not remove tracked files or earlier commits, and it does not control what a build or deployment uploads. Inspect new public assets, generated bundles, screenshots, archives, and their metadata before publication. Keep deliberate public assets and safe configuration examples available.
- If a credential may have been exposed, stop its publication and tell the owner privately with the repository, path, and revision, not the value. Recommend revocation or rotation. Do not test the credential, rewrite history, change visibility, or delete remote content without explicit authorization.
