# Figma Friend

A Claude Code plugin that lets you interact with Figma directly through the browser.

## Installation

```bash
/plugin marketplace add markacianfrani/claude-code-figma
/plugin install figma-friend
```

## Usage

Once installed, use the `/figma-friend:skills` skill to start interacting with Figma:

```bash
/figma-friend:skills
```

The plugin will:
1. Open Figma in your browser
2. Prompt you to log in and open a design file
3. Let you create shapes, modify properties, and extract information using natural language

## Requirements

- Claude Code
- A Figma account with edit permissions on the file you want to modify

## Troubleshooting

If you get `figma is not defined`:
- Make sure you have edit permissions on the file (try creating a branch)
- Open any plugin in Figma and close it, then try again
