# SubRouter Skills

Agent skills for [SubRouter](https://subrouter.ai) — an OpenAI- and Anthropic-compatible AI API gateway with per-request smart routing across providers and transparent per-token billing.

## Skills

| Skill | What it does |
|---|---|
| [`subrouter`](skills/subrouter/SKILL.md) | Connects an agent to SubRouter end to end: device-flow authorization, API key, model selection by price, client configuration, and account management. |

Works with Claude Code, Codex, Cursor, Zed, and any client that reads an OpenAI- or Anthropic-compatible endpoint.

## Install

The same `SKILL.md` works in both Claude Code and Codex — they use an identical skill format.

### Claude Code

```
/plugin marketplace add abingyyds/SubRouter-skills
/plugin install subrouter-skills@subrouter-skills
```

Or by hand into `~/.claude/skills/`:

```bash
git clone https://github.com/abingyyds/SubRouter-skills.git
cp -r SubRouter-skills/skills/subrouter ~/.claude/skills/
```

### Codex

Codex discovers skills in `~/.codex/skills/<name>/SKILL.md`:

```bash
git clone https://github.com/abingyyds/SubRouter-skills.git
cp -r SubRouter-skills/skills/subrouter ~/.codex/skills/
```

Single-file install without cloning:

```bash
mkdir -p ~/.codex/skills/subrouter
curl -fsSL https://raw.githubusercontent.com/abingyyds/SubRouter-skills/main/skills/subrouter/SKILL.md \
  -o ~/.codex/skills/subrouter/SKILL.md
```

Restart Codex afterwards so it picks the skill up.

### Other clients

Cursor, Zed and similar editors have no skill mechanism of their own — run the skill from Claude Code or Codex and it will write the SubRouter endpoint and key into whichever client you name.

## Use

Ask the agent in plain language — the skill triggers on phrases like:

- "接入 SubRouter" / "配置 SubRouter"
- "帮我拿一个 SubRouter API key"
- "用 SubRouter 上最便宜的 gpt 模型"

The agent runs the device authorization, waits for you to approve it in the browser, writes the key into your client config, and shows you model prices before picking one.

## Safety

The skill is written so an agent will not act on money or credentials by itself:

- The API key is only ever sent to the SubRouter host you named, and never followed through a redirect.
- Payment always happens in your browser. The agent never handles card numbers or crypto credentials.
- Withdrawals, provider applications, and anything that changes listings or prices require your explicit confirmation first.

## Contributing

Each skill lives in `skills/<name>/SKILL.md` with YAML frontmatter (`name`, `description`). The description is what the agent matches on, so keep the trigger phrases in it accurate.

The API surface this skill documents is served by the SubRouter platform. When an endpoint or response shape changes there, update the skill here in the same change.
