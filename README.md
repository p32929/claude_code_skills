# claude_code_skills

Just some skills I wrote for my own [Claude Code](https://claude.com/claude-code) CLI. Sharing them in case they're useful to someone else.

## What's in here

| Skill | What it does |
|---|---|
| [`design_from_figma.md`](design_from_figma.md) | A pixel-perfect design-to-code agent — audits the code *and* compares live screenshots against the Figma in a loop, downloads the design's own assets, and only stops once the diff is genuinely zero |

## How to use one

Claude Code skills are just markdown files. Drop the one you want into your skills directory:

```bash
mkdir -p ~/.claude/skills
curl -o ~/.claude/skills/design_from_figma.md \
  https://raw.githubusercontent.com/p32929/claude_code_skills/master/design_from_figma.md
```

Then invoke it in Claude Code by name — for the one above, `/design_from_figma`.

These are written for how *I* work, so read one before you run it and adjust anything that doesn't fit your project.

## Contributing

Contributions are warmly welcomed and greatly appreciated! Whether it's a bug fix, new feature, or improvement, your input helps make this project better for everyone.

Before submitting a pull request, please:

1. Create an issue describing the feature or bug fix you'd like to work on
2. Wait for discussion and approval to ensure alignment with project goals
3. Fork the repository and create your feature branch
4. Submit your pull request with a clear description of changes

This approach helps avoid duplicate efforts and ensures smooth collaboration. Thank you for considering contributing!

## Share

Sharing this repository with your friends is just one click away from here

[![facebook](https://user-images.githubusercontent.com/6418354/179013321-ac1d1452-0689-493f-9066-940cf2302b6e.png)](https://www.facebook.com/sharer/sharer.php?u=https://github.com/p32929/claude_code_skills/)
[![twitter](https://user-images.githubusercontent.com/6418354/179013351-7d8d6d1c-4ce2-46ab-bef8-4c4765a1b888.png)](https://twitter.com/intent/tweet?url=https://github.com/p32929/claude_code_skills/)
[![tumblr](https://user-images.githubusercontent.com/6418354/179013343-3111f55a-3b90-40c7-8487-9777348672b0.png)](https://www.tumblr.com/share?v=3&u=https://github.com/p32929/claude_code_skills/)
[![pocket](https://user-images.githubusercontent.com/6418354/179013334-b095c45f-becf-49f4-9ee1-5a731a9b1f85.png)](https://getpocket.com/save?url=https://github.com/p32929/claude_code_skills/)
[![pinterest](https://user-images.githubusercontent.com/6418354/179013331-44cd9206-11b1-4b65-becb-5863b61c828f.png)](https://pinterest.com/pin/create/button/?url=https://github.com/p32929/claude_code_skills/)
[![reddit](https://user-images.githubusercontent.com/6418354/179013338-7416ae3f-73ba-4522-86e1-1374d7082d22.png)](https://www.reddit.com/submit?url=https://github.com/p32929/claude_code_skills/)
[![linkedin](https://user-images.githubusercontent.com/6418354/179013327-ca7b7102-1da8-4b1c-858f-1a6e5f21bd70.png)](https://www.linkedin.com/shareArticle?mini=true&url=https://github.com/p32929/claude_code_skills/)
[![whatsapp](https://user-images.githubusercontent.com/6418354/179013353-f477fa0b-3e6f-4138-a357-c9991b23ff88.png)](https://api.whatsapp.com/send?text=https://github.com/p32929/claude_code_skills/)

---

## Support

If this saved you time, you can buy me a coffee — it keeps these projects maintained and free.

[![Buy Me A Coffee](https://img.shields.io/badge/Buy%20me%20a%20coffee-%E2%98%95-FFDD00?style=for-the-badge&logo=buymeacoffee&logoColor=black)](https://www.buymeacoffee.com/p32929)

<!-- kit-block -->

---

## Using Claude Code CLI for real work?

I put together the **[Claude Code Starter Kit](https://p32929.github.io/claude-code-starter-kit/)** — tested `.claude/` subagents, slash commands and guard hooks (blocks `rm -rf` and `.env` reads) that install in 60 seconds. Free Lite version on GitHub, or the full kit plus a done-for-you Team ($999) / Enterprise ($1,499) rollout across your repos.
