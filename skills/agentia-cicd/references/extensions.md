<!-- Agentia managed skill file: agentia setup skills -->

# Copado CICD Extensions

Use this reference for experimental git-hosted extension commands under `agentia cicd extension ...`.

## Scope

`cicd extension` packages Copado Functions and Job Templates (plus optional `force-app`) as a git repository with a semver tag. It is not the same as individual `cicd function` / `cicd template` upserts, Salesforce unlocked packages, or MCP tools (none in v1).

Layout:

- `copado/extension_manifest.json` (includes additive semver `version`)
- `copado/functions/`
- `copado/templates/`
- optional `force-app/`

## Author workflow

```sh
agentia cicd extension create --name my-extension --version 1.0.0
agentia cicd extension validate --require-version
agentia cicd extension package --json
# commit the manifest version before publish
agentia cicd extension publish --remote origin
agentia cicd extension publish --local --force   # offline tag only
```

`publish --version` fails if the bump is still uncommitted. Prefer committing the version, then publishing without rewriting it.

## Install and uninstall

```sh
agentia cicd extension install https://gitlab.com/org/my-ext.git --version 1.0.0 --dry-run
agentia cicd extension install https://gitlab.com/org/my-ext.git --version 1.0.0 --yes --sf-alias destOrg
agentia cicd extension install ./my-extension --version 1.0.0 --exclude-sf-deploy --yes --json
agentia cicd extension uninstall ./my-extension --version 1.0.0 --yes --json
```

Rules:

- Install and uninstall need `--yes` (or a prompt) except `--dry-run`.
- `force-app` needs `--sf-alias` or `--exclude-sf-deploy`.
- `.` on a git repo installs its latest `v*` tag, not the working tree; use `--ref <branch>` to test local commits.
- Symlinks and paths outside the extension root are rejected.
- Uninstall deletes listed Functions/Templates only (404 = already gone); it does not undeploy `force-app`.

## Related

- Individual Functions: [functions.md](functions.md)
- Individual Job Templates: [templates.md](templates.md)
- Customer enablement: `docs/enablement/features/agentia-cli-cicd-extensions.md`
