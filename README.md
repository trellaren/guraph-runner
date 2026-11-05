# Guraph runner

The runner does a Guraph organization's agent work on your own machine or
CI. It claims jobs with a runner key, clones the repository, runs the agent
on your organization's own model key, stops for a person at every
checkpoint, pushes, and reports what the work cost. Your code and your keys
stay where you run it. [guraph.com](https://www.guraph.com)

You need a runner key: in Guraph, open **Settings → Runners** and mint one.
The key is shown once. The tab also shows the commands below with your
address and key filled in.

## With Docker

```sh
docker run -d --name guraph-runner --restart unless-stopped \
  -e GURAPH_API_URL=https://<your-project>.supabase.co/functions/v1 \
  -e GURAPH_RUNNER_KEY=guraph_rk_... \
  -v guraph-work:/work \
  ghcr.io/trellaren/guraph-runner:latest
```

Images are built for `linux/amd64` and `linux/arm64`.

- **Follow it:** `docker logs -f guraph-runner`.
- **Stop it:** `docker stop guraph-runner` stops it after it reports the
  run in flight.

### Your repository's toolchain

The image has git, Node 24 and corepack (pnpm and yarn). An agent that
builds or tests a repository needs that repository's toolchain too. Add it
on top:

```dockerfile
FROM ghcr.io/trellaren/guraph-runner:latest
USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends python3 python3-venv \
 && rm -rf /var/lib/apt/lists/*
USER 10001:10001
```

Build it, then run your image in place of this one.

## Download

Each download holds the runner, Node and the Claude CLI. You need git.

| Platform              | File                                                                                                                                       |
| --------------------- | ------------------------------------------------------------------------------------------------------------------------------------------ |
| Linux x64             | [guraph-runner-linux-x64.tar.gz](https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-linux-x64.tar.gz)       |
| Linux arm64           | [guraph-runner-linux-arm64.tar.gz](https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-linux-arm64.tar.gz)   |
| macOS (Apple silicon) | [guraph-runner-darwin-arm64.tar.gz](https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-darwin-arm64.tar.gz) |
| macOS (Intel)         | [guraph-runner-darwin-x64.tar.gz](https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-darwin-x64.tar.gz)     |
| Windows x64           | [guraph-runner-win32-x64.zip](https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-win32-x64.zip)             |

Check what you downloaded against
[SHA256SUMS](https://github.com/trellaren/guraph-runner/releases/latest/download/SHA256SUMS):

```sh
sha256sum -c SHA256SUMS --ignore-missing          # Linux
shasum -a 256 -c SHA256SUMS --ignore-missing      # macOS
```

### macOS or Linux

```sh
curl -LO https://github.com/trellaren/guraph-runner/releases/latest/download/guraph-runner-linux-x64.tar.gz
tar xzf guraph-runner-linux-x64.tar.gz
GURAPH_API_URL=https://<your-project>.supabase.co/functions/v1 \
GURAPH_RUNNER_KEY=guraph_rk_... \
./guraph-runner/guraph-runner
```

macOS quarantines a file downloaded in a browser and may refuse to run the
Node or Claude binaries inside it. Download with `curl` as above, or clear
the flag: `xattr -dr com.apple.quarantine guraph-runner`.

### Windows (PowerShell)

```powershell
Expand-Archive guraph-runner-win32-x64.zip -DestinationPath .
$env:GURAPH_API_URL = 'https://<your-project>.supabase.co/functions/v1'
$env:GURAPH_RUNNER_KEY = 'guraph_rk_...'
.\guraph-runner\guraph-runner.cmd
```

`guraph-runner --version` says which version you have.

## Settings

Every setting is an environment variable.

| Variable                            | Default                                      | What it does                                                   |
| ----------------------------------- | -------------------------------------------- | -------------------------------------------------------------- |
| `GURAPH_API_URL`                    | required                                     | Your deployment's functions address                            |
| `GURAPH_RUNNER_KEY`                 | required                                     | The key; read once at start, then removed from the environment |
| `GURAPH_WORKSPACES`                 | the system temp folder; `/work` in the image | Where each job's repository is cloned, one folder per job      |
| `GURAPH_POLL_SECONDS`               | 5                                            | How often it asks for work, and checks a checkpoint, when idle |
| `GURAPH_LEASE_SECONDS`              | 300                                          | The claim length it asks for; the server may shorten it        |
| `GURAPH_MAX_TURNS`                  | 50                                           | The most turns one agent session takes                         |
| `GURAPH_CHECKPOINT_TIMEOUT_SECONDS` | 86400                                        | How long a run waits for a person before it gives up           |
| `GURAPH_GITHUB_ORIGIN`              | `https://github.com`                         | A GitHub Enterprise host, if that is where your code is        |
| `ANTHROPIC_API_KEY`                 | none                                         | Used only by an agent that names no model connection           |

## How Claude signs in

An agent whose model connection in Guraph holds a key uses that key, and the
sign-ins below are ignored for it. An agent with no connection signs in
with whatever the runner was started with. That can be any one of these:

- **An Anthropic API key:** `ANTHROPIC_API_KEY`.
- **Your Claude subscription:** run `claude setup-token` once on any machine
  where Claude Code is signed in, and start the runner with the token it
  prints as `CLAUDE_CODE_OAUTH_TOKEN`. A `claude login` on the runner's own
  machine is not seen, because each job runs with an empty home folder.
- **Amazon Bedrock:** `CLAUDE_CODE_USE_BEDROCK=1`, `AWS_REGION`, and either
  `AWS_BEARER_TOKEN_BEDROCK` or the usual `AWS_ACCESS_KEY_ID` and
  `AWS_SECRET_ACCESS_KEY`. For a profile, name its files with
  `AWS_CONFIG_FILE` and `AWS_SHARED_CREDENTIALS_FILE`.
- **Google Cloud:** `CLAUDE_CODE_USE_VERTEX=1`, `CLOUD_ML_REGION`,
  `ANTHROPIC_VERTEX_PROJECT_ID` and `GOOGLE_APPLICATION_CREDENTIALS`.
- **Microsoft Foundry:** `CLAUDE_CODE_USE_FOUNDRY=1`,
  `ANTHROPIC_FOUNDRY_RESOURCE`, and `ANTHROPIC_FOUNDRY_API_KEY` or
  `ANTHROPIC_FOUNDRY_AUTH_TOKEN`.

A cloud's credentials reach the agent only when that cloud is switched on.
Every credential is removed from what the runner reports to Guraph. Usage is
billed to whoever owns the key, subscription or cloud account; Guraph never
pays for or resells it.

## Updating

- **Docker:** `docker pull ghcr.io/trellaren/guraph-runner:latest`, then
  remove and start the container again.
- **Download:** replace the `guraph-runner` folder.

Stopped with Ctrl+C or SIGTERM, the runner reports the run in flight before
it exits.

## What it holds

It never holds a person's Guraph sign-in, Guraph's database keys, or a
repository credential beyond the job it was leased for. It logs one JSON line per
event, by job and run id, never an agent's output. What the agent said and
did is in your Guraph organization, behind its permissions.

## Licence

See [LICENSE](LICENSE). The Claude Code CLI inside each download and the
image is Anthropic's, unmodified, and under
[Anthropic's terms](https://code.claude.com/docs/en/legal-and-compliance).
