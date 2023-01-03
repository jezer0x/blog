---
title: LLM-As-A-Dev - Building a Trading Bot
layout: single
authors:
  - name: psyf
    link: https://twitter.com/psyf01
tags: AI
last_modified_at: 2023-01-03
header:
  overlay_image: /assets/2022-12-30-trading-bot/midjourney_depiction.png
  caption: Midjourney
  show_overlay_excerpt: false
  cta_url: https://github.com/jezer0x/blog/blob/main/_posts/2022-12-30-trading-bot.md
  cta_label: "Edit on Github"
  arweave_tx:
---

## The Task

I was feeling pretty down after we had to close down a project we had been working on for months and wanted to take a small break from smart contracts to collect my thoughts about what we did wrong. Around the same time, I was digging around Github Copilot and OpenAI's GPT3. They released ChatGPT right around then, and - similar to most people - I was blown away by its abilities. The new Large Language Model was inpiring me to build something fun and useful to test its limits. But what?

My colleague Krenzx was following a Telegram group that made calls about the crypto markets and boasted about their success.

![Trading Calls](../assets/2022-12-30-trading-bot/call_example.png){: .align-center}

As someone who had never traded before[^seriously], I saw this as the perfect opportunity.

## The Test

There were 2 metrics to test the LLMs against:

1. How much Google/StackOverflow do I need?
2. How fast can I iterate on the product?

> "Hey, could you write me a **Python** script that takes in **trading calls** from a **Telegram group** and executes them on **Binance**?"

## The Good

- **Assumptions:** It made a few reasonable assumptions about (1) the contents of the trading call, (2) using a Telegram Bot and (3) using Spot on Binance.
- **Fast Iteration:** It churned out some mostly-usable lines of code that encompassed the entire workflow.
- **Learning:** It would also explain what it was doing, so even though I didn't know anything about the Telegram API or Binance API, I picked it up faster than I would otherwise[^otherwise].

## The Bad

- **Bugs:** The draft code had some bugs and I couldn't run it directly. I had to ask ChatGPT to tweak some lines and tell it what errors the repl was throwing in order to fix them.
- **Tweaking:** I also gave it examples of chat messages the group sent and had it tweak the parser. The regex it wrote wasn't perfect, but perhaps giving it more examples would have helped it write more generalizable code instead of overfitting to an example.
- **Domain Knowledge:** While ChatGPT greatly reduced my workload, I still had to be careful to ensure the code was bug-free and actually understand what I was doing. As a result, I ended up asking Krenzx and Madneutrino a lot about the futures concepts in order to understand the API, as I couldn't fully trust the language model.[^itried]
- **Infinite Loop:** There were also times when ChatGPT got caught up in loops or made incorrect assumptions about the APIs, requiring me to read the documentation and understand the nuances myself. For example, the Telegram API had multiple ways of doing things and the Binance API was total crap[^tbf]. It also could not deal with multiple versions of SqlAlchemy, and could not code to SQLite spec.
  ![meme](../assets/2022-12-30-trading-bot/openai-meme.png)
  {: .align-center}

## Development and Deployment

- coded and ran on ReplIt[^whynot]
- but soon moved it to a digitalOcean Droplet[^config]
- Synced the calls from the telegram channel to a local SQLite DB and stored logs for what the bot was doing on flat files (to debug - and there was a lot of debugging).
- Started off on the spot testnet but hit annoying limits. So we just YOLO-ed 1K of ACTUAL USDT. To our surprise, we did not lose money. Time spent so far was about a couple days of work, and I think we went at least 2x fast. **As the code gets more complex returns you get from using LLM drops off.**

## Ape In

Encouraged by this initial success, we decided to try futures. This is when it all went downhill[^sweet]. Both Copilot and ChatGPT were kind of useless and I ended up throwing away a lot of the Spot code, rewriting for the futures API and refactoring for maintainability[^maintain]. To be fair, Copilot did act sometimes as a delightful auto-complete. This part of the project took 4-5 days of work, and mostly unaided by LLMs. Here's a prime example of what the current version of ChatGPT/Codex would never be able to figure out: https://dev.binance.vision/t/how-to-implement-otoco-tp-sl-orders-using-api/1622/18

## Future of the Project

Looking to the future, there were 2 potential directions for this side project:

1. Build a BotAsAService. Anyone who wanted to copy trade a telegram channel could give it to us and we'd run it for them for a flat daily fee. One interesting twist on this would be to integrate the LLM into the onboarding flow so it could talk to the user, write a parser, do some testing and deploy it to production all by itself (think @levelsIO style business for software consultancy, but where the typical onboarding personnel/tier3Dev would be replaced by an AI).
2. Ramp up the risk and leverage by sending in a larger amount of money (say 10k USDT), and see where it takes us with a 10x leverage, take profits and stoplosses. While I was tempted, we ultimately didn't have the motivation to risk such a large amount of money without proper backtesting (and I was too lazy to backtest properly). Similarly, we didn't have the motivation to develop our own trading algorithm, as it felt like a PvP game that wasn't adding much value to the world _(spoiler alert: that's the followup post!)_

The money in either the directions did not motivate us, and we found alternative BotsAsAService solutions that seemed to have a reasonable UX. Just make sure you're not using 3commas :clown_face:

But if you're interested in tinkering with the bot yourself, or going ape shit with leverage, you can find the source code [here](https://github.com/madneutrino/trading-bot/tree/main).

Just be careful and don't use more money than you're willing to lose! :monkey:

[^seriously]: seriously, I'll struggle to explain the difference between margin and futures
[^otherwise]: i.e. by reading docs, seeing and trying to replicate easy examples
[^itried]: Yes, I tried asking chatGPT about the finance concepts but it wouldn't go back and forth with examples, analogies, and step through specific scenarios like my friends would.
[^tbf]: To be fair, even though ChatGPT made incorrect assumptions about the parameters and responses, I'd made the same assumptions before reading the docs
[^whynot]: Because why not explore a shit tonne of things at the same time and whine when it doesn't all work butter smooth - it's 2022 people! I'm an entitled af Dev
[^maintain]: e.g. onitoring, debugging, making future changes, db schema etc. ChatGPT also structures python projects in a very throwaway-script manner which is no bueno for production
[^sweet]: apart from the fact that the futures testnet is SWEET in comparison to the rest of the clusterfuck Binance has
[^config]: Where I had to configure a minimal ubuntu droplet with poetry+pyenv - BY HAND
