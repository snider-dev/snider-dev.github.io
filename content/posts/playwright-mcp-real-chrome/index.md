---
title: "Claude Code + Playwright MCP: Using Your Real Chrome Profile"
description: "Claude Code's WebFetch gets 403'd on most sites. Here's how to use the Playwright MCP Bridge extension to give Claude Code a real browser session, your logged-in Chrome, for web research and automation."
date: 2026-06-09
draft: false
tags:
  - claude-code
  - MCP
  - playwright
  - browser-automation
faq:
  - q: "Why does Claude Code get 403 errors when fetching URLs?"
    a: "Claude Code's built-in WebFetch tool sends requests without a real browser session, so sites that check for browser fingerprints, cookies, or login state block it with a 403. The fix is to use Playwright MCP with your real Chrome browser instead."
  - q: "Why doesn't --profile work with Playwright MCP in Claude Code?"
    a: "The --profile flag launches Playwright's own Chromium binary pointed at your profile directory. That binary doesn't share cookies or login state with your installed Chrome, so you're always unauthenticated regardless of the path you pass."
  - q: "How do I use my real Chrome profile with Claude Code's Playwright MCP?"
    a: "Install the Playwright MCP Bridge Chrome extension, make sure Chrome is already running, then open the browser with --browser=chrome --extension --headed. Playwright connects to your already-running Chrome over CDP. You'll see an approval dialog on each Chrome restart."
  - q: "Does the Playwright MCP extension need re-approval after Chrome restarts?"
    a: "Yes. Each time Chrome restarts you'll see a dialog asking you to approve the connection. Just click Allow & select on the tab you want to automate and you're good to go."
  - q: "How do I configure the playwright-cli Claude Code skill to use my real Chrome?"
    a: "Add a Profile mode section to the top of your ~/.claude/skills/playwright-cli/SKILL.md telling Claude to always ask whether to use your real Chrome before opening a browser, and to use --browser=chrome --extension --headed instead of --profile."
---

If you use Claude Code for day-to-day development, you've run into this. You ask it to pull up a doc, check a Reddit thread, or look at a GitHub issue, and it comes back with a 403. Or it fetches some stripped-down cached version that's missing the thing you actually needed.

`WebFetch` doesn't use a real browser. No cookies, no session, no JS rendering. A lot of the sites developers actually need hit it with a 403 or a login redirect.

ChatGPT and Gemini have web search built in. Claude Code doesn't, at least not out of the box. But you can get pretty close by connecting Claude Code to your real Chrome via Playwright MCP. Here's how.

## Why this keeps happening

When Claude Code fetches a URL it's sending a plain HTTP request. No browser fingerprint, no cookies, no rendered DOM. Sites see it as a bot and either block it (403), redirect to login, or throw a Cloudflare challenge.

It's most annoying when you're:
- Integrating a new library and the actual docs are behind a JS render
- The answer to your bug is on a Reddit thread that needs login to view properly
- You want Claude Code to use Gemini's deep research or web search instead of fetching raw URLs

The obvious fix looks like `playwright-cli open --profile`. It doesn't work, and it fails without telling you why.

## What's wrong with --profile

```bash
playwright-cli open --browser=chrome --profile="/Users/you/Library/Application Support/Google/Chrome/Default"
```

Looks fine. The path is right. But Playwright doesn't launch your Chrome app, it launches its own bundled Chromium binary and points it at that directory. Chromium and Chrome don't share a cookie store, so you get a fresh unauthenticated session no matter what path you pass.

I tried this three different ways before I understood what was happening. The path was right the whole time. The binary was the problem.

## The actual fix

The Playwright MCP Bridge extension connects Playwright to your already-running Chrome over CDP. Same browser you use every day, same cookies, same sessions. Everything you're logged into just works.

**Step 1: Install the extension**

Search "Playwright Extension" in the Chrome Web Store. The ID is `mmlmfjhmonkocbjadbfplnigmagldckm` if you want to go directly.

**Step 2: Use --extension instead of --profile**

Chrome needs to already be running. Then:

```bash
playwright-cli open --browser=chrome --extension --headed https://example.com
```

That's it. Playwright connects to your running browser and your login state is there.

## Approving the connection

The first time you run this, and every time Chrome restarts, you'll see a dialog:

> "playwright-cli is trying to connect to the Playwright Extension"

Click **Allow & select** on the tab you want to automate. That's the whole setup. The re-approval on restart is intentional — the extension has full access to your browser including everything you're signed into, so it asks each session.

## Using this for web research in Claude Code

Once it's set up you can point Claude Code at ChatGPT, Gemini or Perplexity and have it do research there with your real session, rather than firing WebFetch at raw URLs and getting blocked.

If you're using the `playwright-cli` skill in Claude Code, add this section to the top of your `~/.claude/skills/playwright-cli/SKILL.md` so Claude always knows to ask before opening a browser:

```markdown
## Profile mode (use my real Chrome)

**Before opening a browser, if the user hasn't specified, always ask:**
> "Should I use your Chrome profile (so you're logged in), or open a fresh browser?"

If the user wants their profile:
- Chrome must already be running — if not, tell the user to open it first
- Always use `--browser=chrome --extension` (never `--profile`, it doesn't work)
- Always use `--headed` so the browser is visible
- If the connection dialog appears in Chrome, tell the user to click Allow & select

```bash
playwright-cli open --browser=chrome --extension --headed https://example.com
```

If connection times out, remind the user that Chrome needs to be open and the Playwright Extension must be installed (Chrome Web Store ID: `mmlmfjhmonkocbjadbfplnigmagldckm`).
```
