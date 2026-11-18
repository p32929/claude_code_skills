# claude_code_skills

Ready-made instructions for [Claude Code](https://claude.com/claude-code) that I wrote for my own work and use every day. Each one is a single markdown file you drop in, and it turns Claude Code into a careful specialist for one job.

## What's in here

1. **[`design_from_figma.md`](design_from_figma.md):** turns a Figma design into code that matches it pixel for pixel. Most "Figma to code" prompts stop when the result looks close enough. This one does not trust its own eye: it measures.
   1. It reads exact colours, sizes and spacing from the Figma file through the Figma MCP tools, and downloads the design's own icons and images instead of reusing whatever the project already has.
   2. It checks the code and live screenshots together, compares them with the design using ImageMagick (`magick compare`, pixel sampling), and keeps a diff table of every value.
   3. It only stops after the diff is clean five times in a row.

## How to use one

Each file is a custom slash command. Put it in your Claude Code commands folder:

```bash
mkdir -p ~/.claude/commands
curl -o ~/.claude/commands/design_from_figma.md \
  https://raw.githubusercontent.com/p32929/claude_code_skills/master/design_from_figma.md
```

Then run it in Claude Code by its file name, for example `/design_from_figma`, and give it your Figma link.

`design_from_figma` needs the Figma MCP server connected to Claude Code and ImageMagick 7 (`magick`) installed.

These are written for how I work, so read one before you run it and change anything that doesn't fit your project.

## Contributing

Contributions are very welcome, thank you! Before you start on a pull request, please open an issue and tell me what you would like to add or change. That way we can agree on it first, and your time is not wasted on something that might not fit. Thanks for helping make these skills better.

## You might also like

<p><a href="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/0"><picture><source media="(prefers-color-scheme: dark)" srcset="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/0-dark.svg"><img src="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/0-light.svg" width="50%" align="top" alt="More apps by p32929 on OpenStore (1)"></picture></a><a href="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/1"><picture><source media="(prefers-color-scheme: dark)" srcset="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/1-dark.svg"><img src="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/1-light.svg" width="50%" align="top" alt="More apps by p32929 on OpenStore (2)"></picture></a><br><a href="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/2"><picture><source media="(prefers-color-scheme: dark)" srcset="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/2-dark.svg"><img src="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/2-light.svg" width="50%" align="top" alt="More apps by p32929 on OpenStore (3)"></picture></a><a href="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/3"><picture><source media="(prefers-color-scheme: dark)" srcset="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/3-dark.svg"><img src="https://openstore.pfay.workers.dev/more/p32929/claude_code_skills/personal/3-light.svg" width="50%" align="top" alt="More apps by p32929 on OpenStore (4)"></picture></a></p>

<p><a href="https://p32929.github.io/share/?repo=claude_code_skills"><picture><source media="(prefers-color-scheme: dark)" srcset="https://p32929.github.io/readme/footer-share-dark.svg"><img src="https://p32929.github.io/readme/footer-share-light.svg" width="50%" align="top" alt="Share this project with your friends in one click"></picture></a><a href="https://p32929.github.io/support/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://p32929.github.io/readme/footer-support-dark.svg"><img src="https://p32929.github.io/readme/footer-support-light.svg" width="50%" align="top" alt="If you like my work and want to support me, feel free to support"></picture></a><br><a href="https://openstore.pfay.workers.dev/add"><picture><source media="(prefers-color-scheme: dark)" srcset="https://p32929.github.io/readme/footer-join-dark.svg"><img src="https://p32929.github.io/readme/footer-join-light.svg" width="100%" align="top" alt="Put your own apps on OpenStore: free, your listing lives in your own repo"></picture></a></p>
