# rerout-skill

An [Agent Skill](https://docs.claude.com/en/docs/agents-and-tools/agent-skills)
that teaches AI coding agents how to integrate [Rerout](https://rerout.co) —
branded short links, QR codes, and signed webhooks — into any app.

When an agent has this skill installed and a user asks to "add Rerout short
links," "set up Rerout webhooks," or "render a QR code with Rerout," the agent
reads `SKILL.md` and wires it up correctly the first time: right auth, right
endpoints, the right SDK for the language, and correct webhook signature
verification.

## What's inside

```
SKILL.md                              # entry point — when to use + core workflows
references/
  rest-api.md                         # REST contract (endpoints, shapes, errors)
  sdk-quickstart.md                   # install + usage for all 10 SDK languages
  webhooks.md                         # event types, delivery, signature verification
examples/
  create-link-and-webhook.md          # curl + TypeScript
  receive-webhook-express.md          # verify deliveries (Node + Python)
```

The skill uses progressive disclosure: `SKILL.md` stays short and points to the
`references/` files for depth, so agents load only what a task needs.

## Install

**Claude Code** — clone into your skills directory:

```bash
git clone https://github.com/ModestNerds-Co/rerout-skill.git \
  ~/.claude/skills/rerout
```

**Other agent runtimes** — point your skill/loader at this repo, or vendor the
`SKILL.md` + `references/` into your project. The content is plain Markdown and
provider-neutral.

## Scope (v1)

- Create / list / update / delete short links
- QR code URLs and rendering
- Link and project analytics
- Webhook endpoint **management** (create / list / delete)
- Webhook **receiving** + signature verification

The official SDKs live in
[`ModestNerds-Co/rerout-sdks`](https://github.com/ModestNerds-Co/rerout-sdks)
(TypeScript, Python, Ruby, PHP, Go, Java, Kotlin, .NET, Dart, Rust). Full API
docs: <https://rerout.co/docs>.

## Keeping it current

This skill mirrors the live Rerout API and SDK surface. When the API changes,
update `references/` so agents aren't misled — a stale skill is worse than no
skill. PRs welcome.

## License

MIT — see [LICENSE](./LICENSE).
