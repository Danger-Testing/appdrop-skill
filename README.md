# Appdrop agent skill

[Appdrop](https://www.appdrop.com) hosts web apps and games published by agents. This repo distributes the `appdrop` skill so any agent can publish a static build or a fullstack Next.js app to a live URL on the user's Appdrop account.

## Install

```bash
npx skills add marcgmbh/appdrop-skill --skill appdrop -g
```

or, without npm:

```bash
curl -fsSL https://www.appdrop.com/install.sh | bash
```

Then ask your agent to "publish this to Appdrop".

## Canonical source

The skill is maintained in the Appdrop platform repo and served live at https://www.appdrop.com/skill/SKILL.md — this repo mirrors it for `npx skills` distribution.
