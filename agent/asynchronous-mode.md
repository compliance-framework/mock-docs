---
title: Asynchronous agent runs
description: Run the CCF Agent once per pipeline event, with the CLI or the GitHub Action.
sidebar_position: 4
---

# Asynchronous agent runs

As an operator or pipeline developer, you can run the Continuous Compliance Framework (CCF) Agent when a deployment or repository event occurs. Use one-shot mode to collect evidence without keeping an agent daemon running.

## How it works

Your pipeline receives an event, starts the CCF Agent, waits for it to finish, and records the job result. The pipeline controls when to run; the agent does not subscribe to events itself.

With `daemon: false` (the default), the agent downloads its configured CCF Plugins and CCF Policies, runs the plugins once in name order, and exits. Plugin `schedule` settings apply only in daemon mode. Here, asynchronous means independent, event-triggered jobs: the agent command still waits for the run to complete.

A plugin execution error stops the run, skips subsequent plugins, and returns an error to the pipeline. A policy finding such as `not-satisfied` is evidence, not automatically a failed pipeline job. If you need a compliance gate, define a separate decision based on the evidence; a successful process exit alone does not prove compliance.

## Before you start

- For the CLI example, install agent `v0.9.3` as described in the [agent overview](./index.md#install-the-agent). The GitHub example uses `agent-action` `v0.0.3`, which bundles agent `0.9.2`.
- Have an API URL reachable from your runner, an agent service account client ID, and its client secret. Ask your CCF administrator to provision the account and key if you do not manage agents.
- Allow the runner to reach the plugin and policy OCI registries, and the system the plugin checks. Give the plugin the credentials and permissions it requires on that system.
- Choose a plugin and policy bundle for your target. The examples use the APT versions plugin, which inspects packages in the environment where it runs. In a container action, that environment is the action container, not a deployed host.
- Keep API and plugin credentials in your pipeline's secret store. Run credentialed jobs only from trusted events and trusted configuration.

## Enable it

Create `ccf-event.yaml` for a CLI job:

```yaml
daemon: false
api:
  url: https://ccf-api.example.com
remote_config:
  mode: off
plugins:
  apt-versions:
    source: ghcr.io/compliance-framework/plugin-apt-versions:v0.7.0
    policies:
      - ghcr.io/compliance-framework/plugin-apt-versions-policies:v0.5.1
    labels:
      environment: example-ci
```

Replace the API URL, plugin, and policies with your deployment's values. `remote_config.mode: off` keeps this CLI example's configuration local to the pipeline. No `schedule` is needed.

Inject `CCF_API_AUTH_CLIENT_ID` and `CCF_API_AUTH_CLIENT_SECRET` from your secret store into the job environment. The agent reads these without an `api.auth` block; do not commit the key to the configuration file.

## Use it

### Run from an event handler or CI job

After the event's deployment or artifact preparation succeeds, run:

```sh
concom agent --config ccf-event.yaml --daemon=false
```

The explicit flag overrides `daemon: true` if you reuse an existing configuration. Use `false` in YAML rather than relying on the shorthand `daemon=0`.

Have the pipeline wait for the command and preserve its exit status. A webhook consumer can enqueue a job that runs this command; a CI system can place it after its deployment step. Give each concurrent job a separate working directory so downloads and state files do not share a writable location.

For repeated jobs that represent the same logical agent instance, set `CCF_INSTANCE_ID` to a fixed UUID stored as a pipeline variable, for example `0b6f8c2e-4d1a-4f5e-9a3b-7c2d1e0f9a8b`. Without it or a persistent state directory, fresh runners generate new instance IDs. Use distinct IDs for distinct logical instances. This setting identifies the instance; it does not deduplicate events or prevent overlapping jobs.

Keep evidence labels stable when repeated events should update the same evidence stream. Adding a unique build ID as a label creates a new stream rather than updating the existing one. See [what the agent adds to evidence](./plugins.md#what-the-agent-adds-to-evidence).

### Run with the GitHub Action

Create `.github/workflows/ccf-evidence.yaml` in your pipeline repository. This example runs after a push to `main` or a manual dispatch:

```yaml
name: Collect CCF evidence
on:
  push:
    branches: [main]
  workflow_dispatch:

permissions:
  contents: read

jobs:
  evidence:
    runs-on: ubuntu-latest
    steps:
      - name: Run CCF Agent once
        uses: compliance-framework/agent-action@f84eca9a4722ced55c9e04eafa078b057c4ed223 # v0.0.3
        env:
          CCF_API_AUTH_CLIENT_ID: ${{ secrets.CCF_API_AUTH_CLIENT_ID }}
          CCF_API_AUTH_CLIENT_SECRET: ${{ secrets.CCF_API_AUTH_CLIENT_SECRET }}
          CCF_INSTANCE_ID: ${{ vars.CCF_INSTANCE_ID }}
        with:
          api_url: ${{ vars.CCF_API_URL }}
          verbosity: '0'
          plugins: |
            apt-versions:
              source: ghcr.io/compliance-framework/plugin-apt-versions:v0.7.0
              policies:
                - ghcr.io/compliance-framework/plugin-apt-versions-policies:v0.5.1
              labels:
                environment: example-ci
```

Set the repository or environment secrets `CCF_API_AUTH_CLIENT_ID` and `CCF_API_AUTH_CLIENT_SECRET`. Set the variable `CCF_API_URL` to your API URL and `CCF_INSTANCE_ID` to your chosen fixed UUID. Use a Linux runner with Docker support for this container action.

The action accepts `api_url` and `plugins` as required inputs, and optional `verbosity` (default `0`). Pass only the plugin map under `plugins`, without an outer `plugins:` key. There is no `daemon` input: the entrypoint generates a configuration with `daemon: false`, then runs the agent in the foreground. It provides no action outputs.

The action prints its generated configuration. For plugin secrets, put a literal reference such as `token: "${env:PLUGIN_TOKEN}"` in the plugin configuration and supply `PLUGIN_TOKEN` through the step's `env`. Do not interpolate a GitHub secret directly into the `plugins` input.

For deployment-triggered collection, place this step after the deployment in the same job, or in a job that depends on the deployment with `needs`. Select a plugin that can reach the deployed target and provide its required settings. Add a checkout step before the action only if your selected plugin needs repository files.

## Troubleshooting

- **The job never finishes:** check for `daemon: true` or a `--daemon` flag in a CLI wrapper. Use `--daemon=false`. A plugin can also take time to finish; inspect its logs and configure a pipeline job timeout.
- **A plugin's schedule is ignored:** one-shot mode runs it immediately once. Configure the pipeline's event trigger instead of a plugin cron schedule.
- **Authentication or connection fails:** check the API URL, runner network access, service account key, and whether the event has access to the required secrets. Do not expose secrets to untrusted pull request jobs.
- **Later plugins do not run:** the agent stops on the first execution error. Fix the failing plugin's configuration, credentials, or registry access, then rerun the job.
- **The job passes with failed controls:** inspect the submitted evidence. Policy status and plugin execution errors are different results; add your own evidence-based gate if required.
- **Every event shows a new instance:** set `CCF_INSTANCE_ID` consistently, or preserve the state directory with `CCF_STATE_DIR` for CLI jobs.

## Related

- [Agent overview](./index.md)
- [Configuration reference](./configuration.md)
- [Plugin contract and evidence labels](./plugins.md)
- [Agent one-shot implementation at v0.9.3](https://github.com/compliance-framework/agent/blob/v0.9.3/cmd/agent.go)
- [GitHub Action inputs at v0.0.3](https://github.com/compliance-framework/agent-action/blob/v0.0.3/action.yml)
- [GitHub Action entrypoint at v0.0.3](https://github.com/compliance-framework/agent-action/blob/v0.0.3/entrypoint.sh)
