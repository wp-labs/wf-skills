# wf-skills

Product-level skills for Warp Fusion.

## Quick Start

```bash
curl -sSf https://get.warpparse.ai/inst-x.sh | bash -s -- wf-skills
```

Installs into `~/.claude/skills` or `~/.codex/skills` depending on platform detection.

## Available Skills

| Skill | Description |
|-------|-------------|
| `wf-data-governance` | WPL/OML adapter governance and field normalization for Warp Fusion data contracts |
| `wf-wfs-authoring` | Author and review `.wfs` window schema files |
| `wf-wfl-authoring` | Author, review, and lint `.wfl` detection rules |
| `wf-config-authoring` | Create and troubleshoot `wfusion.toml` runtime configuration |
| `wf-test-pipeline-debugging` | Debug wf-rules test pipelines from `wfgen` through `wfusion` alert sinks |
| `wf-integration-authoring` | 把 WarpFusion 接入自有系统的五步集成法（数据来源→输入窗→告警输出窗→输出路由→规则），工程接线与端到端验证 |

## Skill Structure

Each skill follows this structure:

```text
skills/<skill-name>/
├── SKILL.md
└── skill.json
```

## Local Install

```bash
git clone https://github.com/wp-labs/wf-skills.git
cd wf-skills
bash install-skill.sh wf-wfl-authoring
```

## Environment Variables

| Variable | Description | Default |
|----------|-------------|---------|
| `WF_SKILLS_REF` | Branch or tag to install | `main` |
| `WF_SKILLS_SOURCE` | Source repository | `wp-labs/wf-skills` |
| `WF_SKILLS_PLATFORM` | Target platform: `codex`, `claude-code`, or `auto` | `auto` |

## Supported Platforms

- Claude Code: installs to `~/.claude/skills/`
- Codex: installs to `~/.codex/skills/`

Auto-detection installs to all existing platform skill directories by default.

## Versioning

Repository release version is recorded in `version.txt`.

Recommended release flow:

```bash
echo 0.1.1 > version.txt
git add version.txt
git commit -m "chore: release v0.1.1"
git tag v0.1.1
git push origin main
git push origin v0.1.1
```
