# Agentia plugin for Cursor

Salesforce developers can take AI-assisted changes through their Copado pipeline without leaving Cursor. Describe what you need in plain language, and Cursor runs the matching Agentia CLI commands on your Copado records, so Copado stays the system of record.

Agentia teaches Cursor how to operate Copado from your machine with the Agentia CLI. It adds skills that Cursor loads when your request matches them, plus a rule that points the agent at those skills:

| Skill              | Use it for                                                                                                                                                                                                                          |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `agentia-cicd`     | Copado CICD: user stories, commits, cloud commit and promote, local quality gates, publish and submit, promotions and merge conflicts, job executions, Functions, Job Templates, extensions, environments, credentials, pipelines, Slack notifications, release data templates; Org Intelligence (read Salesforce metadata in a sandbox or developer org); Copado metadata dependency list |
| `agentia-adv`      | Copado Pipelines Advanced: organizations, deployments, components, and deployment jobs                                                                                                                                              |
| `agentia-testing`  | Copado Robotic Testing: projects, robots, jobs, builds, schedules, data tables, variables, browser testing, Robot Framework runs, and QWeb and QForce keyword guidance                                                              |
| `agentia-ai`       | Copado AI agents and the interactive help agent: Plan, Build, Test, Release, Operate, Knowledge, and orchestrate, with workspace, user story, and Salesforce org context                                                            |

The skills tell Cursor which Agentia commands to run, in what order, and which IDs to read first. You still install and authenticate the Agentia CLI yourself.

## What the plugin contains

The plugin ships Markdown skill files and one Cursor rule (`rules/agentia.mdc`). It has no MCP server, hooks, slash commands, or binaries, and it does not collect or send any data itself.

When a skill applies, Cursor runs `agentia` commands on your machine through its normal shell tool, so Cursor asks for your permission as it does for any other command. Those commands call the Copado services your Agentia credentials point to:

- The Copado CICD gateway for your region, for `agentia cicd` commands, including Org Intelligence (`agentia cicd metadata`)
- Copado Pipelines Advanced, for `agentia adv` commands
- Copado Robotic Testing, for `agentia testing` commands
- Copado AI, for `agentia ai` commands and `agentia help-agent` (the help agent signs in with your CICD credentials, not your AI credentials)

The skills tell Cursor to read records before changing them, to use `--json` for results it parses, to confirm the target and your intent before destructive actions such as deletes or cancelling jobs, and never to print API keys, personal access tokens, or other secrets.

If Agentia MCP tools are already available in your Cursor session, the skills prefer those tools for the same operations. This plugin does not install that MCP server.

## Requirements

- A Copado account with access to the products you want to use: CICD, Pipelines Advanced, Robotic Testing, or AI
- Node.js 22.22.2 or later, and npm
- The Agentia CLI, installed and authenticated. The plugin version matches the CLI release its skills were taken from, so use that CLI version or later.

Install the CLI:

```sh
npm install -g @copado/agentia-cli
```

## Install

From the [Cursor Marketplace](https://cursor.com/marketplace), open **Customize** in the sidebar, find **Agentia**, and choose **Install**. Pick a user or project scope.

On a Teams or Enterprise plan, an admin can import this repository from **Dashboard → Plugins & MCPs** and set the plugin to Default Off, Default On, or Required. Teammates then install it from **Customize**. That import does not list the plugin on the public marketplace.

To try a local copy before it is listed, copy this repository to `~/.cursor/plugins/local/agentia`. A symlink that points outside that folder is ignored. Reload the window, then open **Customize** and confirm the skills (`agentia-cicd`, `agentia-adv`, `agentia-testing`, and `agentia-ai`) and the Agentia rule.

The plugin replaces `agentia setup skills create --target cursor`. If you already ran that command in a project, Cursor sees two copies of each skill and rule. Remove the copies under `.cursor/skills/agentia-*` and `.cursor/rules/agentia.mdc` in that project, or don't enable the plugin there.

## Troubleshooting

- **Cursor says `agentia` is not found.** Install the CLI as shown above and make sure the npm global bin folder is on your `PATH`.
- **Commands fail with an authentication error.** To see which credential is missing, run `agentia auth get --cicd --json` (CICD and the help agent), `agentia auth get --crt --json` (Robotic Testing), `agentia auth get --ai --json` (AI agents), or `agentia adv auth status --json` (Pipelines Advanced). Then set it with `agentia auth set`.
- **A skill does not load.** Open **Customize**, confirm **Agentia** is installed, and reload the window. Mention Copado or Agentia in your request so Cursor picks the right skill.
- **The CLI warns that an update is available.** Run `agentia update`, then update the plugin from **Customize**. The CLI may suggest `agentia setup skills update`. You only need that for projects that use CLI-managed skills, not for this plugin.

## License

This plugin is released under the [MIT License](LICENSE).

## Documentation and support

- Copado developer documentation: https://developer.copado.com/docs
- Copado: https://www.copado.com
