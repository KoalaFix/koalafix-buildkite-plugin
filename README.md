# KoalaFix feeder plugin for Buildkite

Write a fact to KoalaFix memory from a Buildkite step on your own agents: the release a tag build makes, the version npm
or PyPI serves as latest, the digest an image tag points to, or an HTTP probe of a service only your network can reach. Agents then read a fact that says who checked it and when. The plugin
runs the feeders from the `@koalafix/feeders` npm package, the same ones the KoalaFix GitHub Action runs. The
Action's `github-release` feeder is `buildkite-release` here: Buildkite has no release event, so the plugin writes
the tag build as the release.

## Before the first run

1. A space with the key registered in it, for example `web.widgets.version`, with a freshness window longer than the
   gap between runs.
2. A feeder token for that space whose allowed keys cover the key. Never a person's token.
3. A Linux or macOS agent: the hook is a bash script. It needs Node 22 or later with `npx`, and for `http-probe` a
   route to the address. If your agents allow only listed plugins, allow `github.com/koalafix/koalafix-buildkite-plugin`.
4. The token, in one of two places:
   - A Buildkite secret named `KOALAFIX_FEEDER_TOKEN` (Agents, your cluster, Secrets, New Secret), or any name you give
     in `token-secret`. A secret belongs to one cluster and needs agent 3.106.0 or later, so the pipeline must run on
     that cluster's agents. The plugin reads it with `buildkite-agent secret get` in its own hook, so it exists only
     inside that hook. Buildkite redacts a secret's value from the build log.
   - An environment variable your agent's environment hook sets, named in `token-env`. Use this when the agents are
     not in a cluster, or the token comes from a vault or the instance's own secret store. End the name in `_TOKEN`,
     for example `KOALAFIX_FEEDER_TOKEN`: the agent redacts variables with that ending from the log by default.

The configuration takes the name of the secret or the variable, never the token. The token is never in the pipeline
file, a command line or the log: the hook passes it to the feeder's process by environment and prints nothing that
holds it. Do not turn on shell tracing for the step (`set -x`) or add the token to a `command`.

## Configuration

| Option | Default | Meaning |
|---|---|---|
| `feeder` | required | `buildkite-release`, `npm-version`, `pypi-version`, `http-probe` or `image-tag` |
| `space` | required | the space the key belongs to |
| `key` | required | the key to write, lower-case `subject.fact` |
| `token-secret` | `KOALAFIX_FEEDER_TOKEN` | the name of the Buildkite secret that holds the feeder token |
| `token-env` | | the name of an environment variable that holds the token; used instead of the secret when set |
| `server-url` | `https://mcp.koalafix.com` | the KoalaFix server's address |
| `package` | | npm-version and pypi-version: the npm package or the PyPI project |
| `url` | | http-probe: the address to request with GET |
| `expect-status` | `200` | http-probe: the status that counts as up |
| `expect-body` | | http-probe: a regular expression the body must match |
| `interval` | `60` | http-probe: seconds between runs; set it to the schedule's gap |
| `image` | | image-tag: the tag to read, `registry/name:tag` |
| `registry-username` | | image-tag, private images: the registry user |
| `registry-password-secret` | `REGISTRY_PASSWORD` | image-tag, private images: the name of the Buildkite secret that holds the registry password or token |

The hook runs after the step's command (`post-command`) and does nothing if that command failed. Buildkite fails a
step that has no command, so every step carries one; `command: "true"` will do for a step that only runs the plugin.

Buildkite names the plugin's environment variables after the repository, `koalafix-buildkite-plugin`, so a step's
options reach the hook as `BUILDKITE_PLUGIN_KOALAFIX_<OPTION>`.

## Write the release a tag build makes

The ticket case: a build for a tag writes the tag as the release. The pipeline must build tags (turn on tag builds in
the pipeline's GitHub settings). The `if` keeps the step off branch builds, where it would fail because there is no tag.
Building the same tag again writes nothing new.

```yaml
steps:
  - label: ":rocket: release"
    command: ./scripts/release.sh
    if: build.tag != null
    plugins:
      - koalafix/koalafix#v1.0.0:
          feeder: buildkite-release
          space: your-space
          key: release.latest
```

## Write the version npm serves as latest

Run it after your publish step. It reads the public registry; no npm credential is needed. Running it again for the
same version is a check: the answer is `unchanged` and the fact's window restarts.

```yaml
steps:
  - label: ":npm: publish"
    command: npm publish
    plugins:
      - koalafix/koalafix#v1.0.0:
          feeder: npm-version
          space: your-space
          key: web.widgets.version
          package: "@your-org/widgets"
```

## Probe a service only your network can reach

Create a pipeline schedule (Pipeline settings, Schedules, for example `*/15 * * * *`) and run the step on an agent
inside your network, chosen by a queue:

```yaml
steps:
  - label: ":stethoscope: billing health"
    command: "true"
    agents:
      queue: internal
    plugins:
      - koalafix/koalafix#v1.0.0:
          feeder: http-probe
          space: your-space
          key: billing.health
          url: http://billing.internal/healthz
          expect-status: 200
          interval: 900
    if: build.source == "schedule"
```

It writes `{ status, ok, matched }`. A service that does not answer is written as `ok: false`, never skipped. The step
fails only when KoalaFix did not take the fact; whether the service is up is in the fact itself. To read the token
from the agent's environment instead of a secret, add `token-env: OUR_FEEDER_TOKEN`.

## Exit codes

| Code | Means |
|---|---|
| 0 | the fact landed, or KoalaFix already held it; also the step's command had failed and nothing was sent |
| 1 | KoalaFix refused the request or could not be reached, or the feeder could not read its source |
| 2 | an option or the token is missing or wrong; nothing was sent |
| 3 | KoalaFix took the request but refused the fact; the log says which key and why |

## Testing the plugin

`tests/buildkite.test.ts` in this package parses `plugin.yml` against the Action's inputs and runs the hook under
bash with a fake `npx` and a fake `buildkite-agent`, checking the flags, the environment and that the token is never
printed. It needs bash, so it is skipped where there is none.

The plugin's tests live in the Kalibrate repository, not in the public plugin repository, so Buildkite's plugin tester
(which runs `bats tests/`) has nothing to run there. Buildkite's linter works on a checkout of the published plugin
repository, and is not part of this repository's CI because it needs Docker:

```sh
docker run --rm -v "$PWD:/plugin:ro" buildkite/plugin-linter --id koalafix/koalafix
```

## Publishing a version (maintainers)

The source of this plugin is `packages/feeders/buildkite/` in the Kalibrate repository. Buildkite resolves
`koalafix/koalafix#v1.0.0` to the public repository `koalafix/koalafix-buildkite-plugin`, tag `v1.0.0`, with
`plugin.yml` and `hooks/` at its root.

1. Publish the `@koalafix/feeders` version to npm.
2. Set `FEEDERS` in `hooks/post-command` to that version. A test keeps it equal to this package's version.
3. Copy this folder to the plugin repository and commit. A copy made on Windows stages the hook as mode 100644, and
   Buildkite runs a hook only when it is executable, so check `git ls-files -s hooks/post-command` shows `100755`
   and fix it with `git update-index --chmod=+x hooks/post-command`.
4. Push a tag, for example `v1.0.0`.
