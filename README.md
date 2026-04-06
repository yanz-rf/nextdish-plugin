# NextDish Plugin for Claude Code

A Claude Code plugin that lets you browse the NextDish menu, get personalized dish recommendations from your order history, and place orders — without leaving your terminal.

## Features

- Browse the menu for any available date
- Filter by vegetarian, price, cuisine, or dish name
- Get AI recommendations based on your past orders
- Place orders end-to-end through your Chrome session

## Requirements

- macOS with Google Chrome
- A NextDish account (nextdish.com)
- Chrome tab open and logged in to nextdish.com
- JavaScript from Apple Events enabled in Chrome (View → Developer → Allow JavaScript from Apple Events)

## Installation

### From a GitHub repo (once published)

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "nextdish": {
      "source": {
        "source": "github",
        "repo": "YOUR_GITHUB_USERNAME/nextdish-plugin"
      }
    }
  }
}
```

Then install via Claude Code:
```
/plugin install nextdish@nextdish
```

### Manual install (no plugin system)

Copy `skills/nextdish/SKILL.md` to `~/.claude/skills/nextdish/SKILL.md`.

## Usage

Just talk to Claude naturally:

```
/nextdish
order from nextdish for tomorrow
what should I order today?
recommend a dish based on my history
order something vegetarian under $15 for Sunday
just pick one for me
```

### Bulk ordering

Order lunch and dinner across multiple days in one go:

```
order both lunch and dinner for Mon–Fri next week
order lunch for the rest of the week, $15–$25 range
order dinner every day this week, just pick for me
```

Claude will fetch all the menus upfront, propose a full plan (or pick automatically if you say "just pick"), then execute all orders sequentially without confirmation prompts between each one.

## How it works

The skill drives your Chrome browser via AppleScript and JavaScript injection — no API keys or credentials needed beyond your existing browser session. It reads the menu, scrapes your order history from `/user-orders`, and navigates the checkout flow.
