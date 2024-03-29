---
title: 🦾 Using External APIs to Augment LLMs
layout: single
authors:
  - name: MadNeutrino
    link: https://github.com/madneutrino
tags: AI
last_modified_at: 2023-02-17
header:
  overlay_image: assets/2023-02-17-using-external-apis-to-augment-llms/abstract-image-of-an-artificial-intelligence-with-many-network-connections-382592674.png
  caption:
  show_overlay_excerpt: false
  cta_url: https://github.com/jezer0x/blog/blob/main/_posts/2023-02-17-using-external-apis-to-augment-llms.md
  cta_label: "Edit on Github"
  arweave_tx:
---

## Why do we need to augment LLMs?

We know that LLMs aren't always accurate. They do some kind of broad pattern matching, and get the general context good enough to be convincing. But, they bad at math or computational responses.

![](https://media.giphy.com/media/n8IDk4HoJJu18IqQIX/giphy.gif){: .align-center}

**Could we make the LLMs work with external data sources or APIs that are good at math, computation, fact checking such that they provide more accurate responses?**

I want to explore ToolFormer and reAct, 2 ways augmenting LLMs to work with external APIs to improve their accuracy.

## Toolformer

### How does it work?

Meta research recently shared ToolFormer, a tool to teach LLMs to call external APIs to augment its response. Here is a [nice summary](https://twitter.com/mathemagic1an/status/1624870248221663232) of what it does.

In ToolFormer, you pre-train the LLM with examples, so it can, by itself, **figure out what APIs are useful for what data**. In other words, by teaching it to convert a statement like `Two + Three = Five` to `Two + Three = Calculator(2+3)`, you teach it to respond to `Two + Three = ` with `Calculator(2+3)`. And then we code up the `Calculator` API to get the answer, `Five` and the LLM continues from there.

![](https://i.imgflip.com/1p4trj.jpg){: .align-center}

How does it know that the right thing to call here is `Calculator(2+3)` and not `Calculater(2-3)`? You rely on the LLM's existing ability to make decent guesses and fine tune it.

This is done per-API. In the case of `Calculator`, the LLM makes repeated guesses on WHEN and WHAT to call the API with and checks if it gave the correct result. In this case, `Calculater(2-3)` results in `Minus One`, but `Calculater(2+3)` results in `Five`, so the latter is the correct guess. It might similarly try to use Calculator on the `Three` by guessing `Calculator(2)` (and a bunch of other guess) but none of them will give the correct answer so it will discard that guess. The paper says the calculator was trained with a million documents, but it found only a few thousand documents where it was actually useful to call the `Calculator`.

### Why is this interesting?

Typically when you train LLMs, you teach it to guess the next word / token in a sequence. That means we would teach the LLM to say `Five` when prompted with `Two + Three = `. In ToolFormer, we teach the LLM to say `Calculator(2+3)` instead of `Five`. Note that, this doesnt mean the LLM knows to say `Calculator(Number(Two) + Number(Three))`. Each pass is done with 1 specific tool (`Calculator` in this case), and it learns where that specific tool can give the correct answer by trying to call the tool with various inputs.

One of the main advances here seems to be that you can use a much smaller LLM like GPT2 or GPTJ outperform plain GPT3 when they are able to call external APis to augment the LLM predictions. Which makes sense: As long as the base LLM is smart enough to use the tools provided to it, it will be better than a larger model which is limited by what it knows and its ability to predict the answer by itself.

## reAct

### How does it work?

There is also some overlap with the reAct framework, we are still trying to get the LLM to answer `Two + Three = ` as `Calculator(2+3)` but the approach is a bit different.

You can see how it breaks down the problem in this more fleshed out example provided in the langchain implementation of reAct:

[ReAct — 🦜🔗 LangChain 0.0.84](https://langchain.readthedocs.io/en/latest/modules/agents/implementations/react.html)

The difference in approach here is that, with each Tool (`Calculator` is a Tool in this setup), you need to provide a text description. The LLM uses this text description to figure out if it’s the right tool for the job, and it also guesses what to call it with. The example above shows the default Search Tool, but you can create custom Tools. All you need to do a provide a description and specify the "type" of the tool.

[Defining Custom Tools — 🦜🔗 LangChain 0.0.84](https://langchain.readthedocs.io/en/latest/modules/agents/examples/custom_tools.html)

![](https://media.giphy.com/media/doYuyYIEkAYPJYrwvM/giphy.gif){: .align-center}

### Why is this interesting?

The nice thing to notice from the complex examples above ("Author David Chanoff has collaborated with a U.S. Navy admiral who served as the ambassador to the United Kingdom under which President?") is that reAct is doing nested queries because it is considering the prompt as a whole. It breaks down the prompt, finds the the answer to the first query, using that response to do the next query till it finds the answer.

![](https://media.giphy.com/media/xTeWOVI8shjIDTq9Yk/giphy.gif){: .align-center}

ToolFormer isn't designed to do that, it's explicitly trained to specific words / tokens with API calls - but the point of ToolFormer is that it expects much less of the LLM it's working with. reAct will give you garbage unless the base LLM is very good.

## Conclusion

I dont know of research comparing applying reAct vs toolformer to a sophisticated LLM like GPT3 using the ToolFormer dataset (we dont want to test ToolFormer on nested queries, it's not designed for it). I suspect that toolformer will do a better job, because you would need a much smarter model to figure out what API to call with what inputs based on the description alone.
