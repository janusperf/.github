# Janus cloud run (GitHub Action)

Composite action that runs a Janus performance test on a cloud device (AWS Device Farm)
and gates the pipeline on the result. It is a thin wrapper around `janus cloud run`: the
schedule, the device, the capture, and the gate verdict are all computed server-side by
the Studio API, so the runner only uploads the app and polls for the verdict.

## Usage

```yaml
- uses: actions/checkout@v4

- name: Janus cloud run
  uses: ./.github/actions/janus-cloud-run # or <org>/janus/.github/actions/janus-cloud-run@<ref>
  with:
    config: janus.cloud.yml
    token: ${{ secrets.JANUS_TOKEN }}
    workspace: ${{ vars.JANUS_WORKSPACE_ID }}
    cli-spec: ${{ vars.JANUS_CLI_SPEC }}
```

The job's exit status is the gate verdict: `0` pass, `1` gate failed, `2` config error,
`3` infrastructure error.

## Inputs

| Input          | Required | Default            | Purpose                                                                            |
| -------------- | -------- | ------------------ | ---------------------------------------------------------------------------------- |
| `config`       | no       | `janus.cloud.yml`  | Path to the cloud run config.                                                      |
| `token`        | yes      |                    | Studio token, sent as `JANUS_SERVICE_TOKEN`. Use a workspace API token in CI.       |
| `workspace`    | no       | (from config)      | Target workspace id. Falls back to `workspace:` in the config.                     |
| `cloud-url`    | no       | public Studio API  | Studio API base URL (`JANUS_STUDIO_API_URL`).                                            |
| `cli-spec`     | yes      |                    | How to install the CLI: npm spec, tarball URL, or path for `npm install -g`.        |
| `node-version` | no       | `20`               | Node.js version on the runner.                                                     |
| `json`         | no       | `false`            | Emit machine-readable JSON instead of human-readable lines.                        |

## Auth

`token` is sent to the Studio API as `JANUS_SERVICE_TOKEN`. In CI, mint a **workspace API
token** (Studio: workspace settings) and store it as `secrets.JANUS_TOKEN`; the API records
the run against that token. Do not use a personal login token in a pipeline.

## janus.cloud.yml

```yaml
version: 1
platform: android # ios | android
app:
  file: ./app-release.apk # built earlier in the job
  bundle_id: com.example.app
test:
  kind: script # script | ai-driven
  bundle: ./e2e/suite.zip # script only
  # goal: "Sign in and reach checkout" # ai-driven only
device:
  model: pixel_7 # optional device targeting
gate:
  min_ux_score: 70
  max_ux_drop: 5
  fail_on_friction: critical
  fail_on_crash: true # fail if any crash or ANR was captured
  thresholds: # per-metric numeric limits (stat: avg|max|min|pN|sustained)
    - metric: app_cpu_percent
      stat: sustained
      above: 90
      for_seconds: 10
```

Paths in `app.file` / `test.bundle` are resolved relative to the config file.

## Distribution prerequisite

`cli-spec` is required because the CLI is not yet on a public registry. Point it at a
published package once available, or at a packed tarball URL (the output of
`npm run package:df:*` / `npm pack`) hosted on your releases. Store the value in a repo
variable (`vars.JANUS_CLI_SPEC`) so the workflow stays version-pinned.
