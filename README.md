# PlanForge

**A Claude Code skill that turns an app idea into a detailed technical plan, then has five adversarial reviewers try to break it.**

Most AI-written plans sound confident and hide gaps. PlanForge writes a 16-section plan, sends it to a council of five critics who each attack it from a different angle, asks you the questions that need a human decision, then rewrites the plan. It repeats until nothing serious is left, or you say stop.

It runs inside Claude Code on your existing subscription. There is no server, no API key, and nothing to deploy. A skill is just a folder of text files.

## See it work first

[`examples/porch-library/porch-library.md`](examples/porch-library/porch-library.md) is a real run on a made-up app idea (a neighborhood book-swap map). Read the final plan, then open [`porch-library.json`](examples/porch-library/porch-library.json) to see both rounds, the critics' findings, and the answers given.

What the two rounds caught in that run:

| Round 1 draft looked fine, but the council found | Fixed in round 2 |
|---|---|
| Login needed database tables that the schema did not have | Added adapter tables and a role column |
| Any member could edit any library (no ownership checks) | Added one authorization helper on every write route, plus tests |
| Rate limits reset on every deploy and saw one shared IP behind the proxy | Moved to a database-backed limiter and trusted the proxy header |
| Staging was promised but had no port, database, or budget | Defined a full staging setup |
| The free email tier would silently stop all logins | Budgeted the paid plan and added a daily send cap |

Round 2 then found second-order problems (for example, the content security policy would have blocked the bot-check widget on the login page). In this example the run was ended after round 2 with `/finalize`, so those findings are listed at the bottom of the plan instead of fixed. A normal run keeps going until the round limit you chose (default 3, at most 6). That example was made before the interview step existed.

## Install

```bash
git clone https://github.com/JackB2003/planforge-skill.git
mkdir -p ~/.claude/skills
cp -r planforge-skill/skills/planforge ~/.claude/skills/
```

Restart Claude Code. The skill is then available as `/planforge`.

## Use

```
/planforge new "My App"          start a plan (begins with a short interview)
/planforge import ./my-spec.md   critique and gap-fill a plan you already have
/planforge resume my-app         pick up an unfinished plan
/planforge show my-app           print the sections
/planforge finalize my-app       stop now and write the final plan
/planforge list                  list saved plans
```

Plans are saved as JSON (and a readable Markdown file when finalized) in `~/planforge-plans/`.

## How a round works

1. **Interview.** For a new plan, Claude first asks seven short questions, one at a time, with suggested answers: what the app is, the goal, the users, the must-haves, constraints, what it must not do, and how many review rounds you want (default 3, up to 6). Your answers are shown to every critic, so they judge the plan against your real goal.
2. **Draft.** Claude writes all 16 sections: architecture, database, server, environment, features, navigation, authentication, integrations, security, QA testing, error logging, code health, deployment, budget, timeline, and the AI agent team that will build it.
3. **Critique.** Five reviewers read the draft in fresh context, in parallel:
   - Security Auditor (attacks it)
   - Systems Architect (checks it holds together from first principles)
   - Implementer (plays the build agent who has to follow it)
   - Ops/Reliability Engineer (checks it against how the stack really behaves in production)
   - Completeness Auditor (hunts for what is missing)
4. **Judge.** Findings are merged. Clear fixes are applied automatically. Real judgment calls become questions for you.
5. **Answer.** You answer one question at a time, with suggested options. You can skip, discuss with the critic who raised it, or finalize early.
6. **Repeat** until no high-severity gaps remain, or the round limit you chose.

## Honest limits

- It plans. It does not build anything.
- The critics are Claude subagents, so they share Claude's blind spots. Treat the output as a strong first review, not a guarantee.
- The Ops/Reliability critic can use web search when your Claude Code setup allows it. In the example run it reported that it had not searched in round 1 and did search in round 2. The plan's claims about third-party policies should be checked against the live pages before you commit to them.
- A full run uses a meaningful amount of your subscription (each round starts five subagents). Expect a few minutes per round.

## Related skills

Spec-Roast pressure-tests a finished spec before development. Roast pressure-tests an idea before you plan it. Both use the same five-persona council pattern.

## License

MIT
