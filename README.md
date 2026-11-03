# dsh-superagent

> **Fork Note:** Forked from [deepseek-ai/dsh-superagent-prototype](https://github.com/deepseek-ai/dsh-superagent-prototype). This fork removes the mandatory single-GitHub-repository and GitHub-agent-account requirement, allowing `dsh-superagent` to run in **Feishu-only / local workspace mode** as a personal daily session orchestrator while preserving full backward compatibility with the original GitHub + Feishu workflow.
>
> **Research prototype for reference only, not an official DeepSeek product.** This code grew out of our team's day-to-day development work, with much of the engineering done by coding agents. Treat it as research code, not production-ready software. It may still contain bugs, and its coding style and engineering practices may be inconsistent. Review and adapt it before using it in your own environment.
>
> We have adapted it for publication, so it is not identical to our internal version or the implementation described in *Exploring Super-agent Design: Towards a More Autonomous Agent System*. Some designs discussed in the article are directions for future development and are not implemented in this prototype.

dsh-superagent is a proactive coding agent packaged as a bundle for [DeepSeek Harness](https://www.deepseek.com/en/harness/). It listens to Feishu and/or GitHub. When someone allowed asks the agent for work, it opens a work item, works on it in its own agent session and directory, and answers in the original conversation (for example in a Feishu topic thread or with a pull request). A web panel shows what needs a person.

It is also a way of working with an agent: **take as little human attention as possible, and give as much value as possible.** An agent that can act for a whole team can also waste a whole team's attention, so it acts only when asked or when a rule the team wrote allows it, and speaks only when someone must act or can use a result. [docs/000-philosophy.md](docs/000-philosophy.md) explains the principle and what it asks of the operator.

## Setup

Setup has many steps, so an agent does it with you. Ask your coding agent:

> Read SETUP.md in this repository and set up dsh-superagent with me.

It asks one short question at a time, shows the full plan once, and then installs and configures everything. [SETUP.md](SETUP.md) is what it follows.

## Start, Stop & Status（启动、关闭与状态管理）

The service runs in a background `tmux` session (`dsh-superagent`) under the dedicated `superagent` DeepSeek Harness profile (default panel URL: `http://127.0.0.1:3082`). Run these commands from the repository root:

```sh
# 启动后台服务 / Start the background service
scripts/dsh-superagent start

# 查看运行状态与健康检查 / Check service status and health
scripts/dsh-superagent status

# 重启服务 / Restart the service
scripts/dsh-superagent restart

# 停止服务（优雅关闭） / Gracefully stop the service
scripts/dsh-superagent stop
```

Logs are written to `.artifacts/dsh-superagent-superagent-*.log`. See [006 Deployment](docs/006-deployment.md) for full details.

## Documentation

Read the numbered documents in order; each builds only on the ones before it.

- [000 Philosophy](docs/000-philosophy.md)
- [001 Domain model](docs/001-domain.md)
- [002 Authorization](docs/002-authorization.md)
- [003 Execution](docs/003-execution.md)
- [004 Prompts](docs/004-prompts.md)
- [005 Panel](docs/005-panel.md)
- [006 Deployment](docs/006-deployment.md): manual setup, configuration, upgrades and troubleshooting
- [007 Architecture](docs/007-architecture.md)
- [008 Cookbook](docs/008-cookbook.md): adding a Source

## Development

`pnpm check` runs the typecheck, the tests and the build.

## License

[MIT](LICENSE)
