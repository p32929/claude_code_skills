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

## Share

<a href="https://p32929.github.io/share/?repo=claude_code_skills"><picture><source media="(prefers-color-scheme: dark)" srcset="https://p32929.github.io/readme/share-dark.svg"><img src="https://p32929.github.io/readme/share-light.svg" alt="Share this with your friends in just a click" width="420"></picture></a>

## Support

<a href="https://p32929.github.io/support/"><picture><source media="(prefers-color-scheme: dark)" srcset="https://p32929.github.io/readme/support-dark.svg"><img src="https://p32929.github.io/readme/support-light.svg" alt="If you like my works and want to support me, feel free to support" width="420"></picture></a>
