---
title: 📜 Barren Wuffet - Technical Learnings
layout: single
authors:
  - name: psyf
    link: https://twitter.com/psyf01
  - name: MadNeutrino
    link: https://github.com/madneutrino
tags: Development
last_modified_at: 2023-02-20
header:
  overlay_image:
  caption:
  show_overlay_excerpt: false
  cta_url: https://github.com/jezer0x/blog/blob/main/_posts/2023-02-20-bw-dev-learnings.md
  cta_label: "Edit on Github"
  arweave_tx:
---

It's August of 2022. I've understood as much of the tech stack powering blockchains that my brain can hold without exploding. I've learnt the basics of smart contracts and security from [CryptoZombies](https://cryptozombies.io/) and [Ethernaut](https://ethernaut.openzeppelin.com/). I've dabbled in DeFi as a user and tried to understand how each of the protocols were designed. It's time to build something of my own.

Thankfully, my friends and I were playing around with the idea of a platform where traders could launch actively managed on-chain funds and people could put money in them. It sounded just the right amount of crazy to me that I could vaguely understand which direction to code in, without anticipating all the problems that would come up (thanks to the constraints the EVM imposes). This article will quickly discuss the surprising, unanticipated technical things I learnt while building Barren Wuffet - with the lens of a fresh-faced web3 dev.

{: .notice--info}
The scope of this article is limited to the technical aspects only. If you want to learn more about the intended purpose of the project, or the nitty gritty details of the smart contract, please check out the docs at [https://barren-wuffet.gitbook.io/barren-wuffet/](https://barren-wuffet.gitbook.io/barren-wuffet/). If you want to know why it ultimately failed, please read the [business post-mortem](https://jezer0x.com/bw-post-mortem/).

{: .notice--warning}
Everything I write here is predicated on my own experience circa August 2022. I'm sure there are better ways to do some things now, I'm sure I've made mistakes, and I'm sure some things have changed. If you have any feedback, please reach out to me on [Twitter](https://twitter.com/psyf01) or comment below so I can update this article!

I've split this article into 4 sections - Backend, Infra and Tooling, Community and Frontend. While Web3 systems don't transfer directly to the Web2 model of frontend/backend/infra, it's an okay way to think about the different aspects of the project.

## Backend - Smart Contracts

### The EVM is ... frustrating sometimes

By far my biggest frustration was [EIP-170](https://eips.ethereum.org/EIPS/eip-170) shipped with the [Spurious Dragon Hard Fork](https://eips.ethereum.org/EIPS/eip-607) to counteract DoS vulnerabilities found during the Shanghai Attacks.
I've hit this limit at least 3 times as my system evolved to be more complex. Everytime I dreaded the thought of having to refactor to fit within the 24KB limit. This article by [soliditydeveloper](https://soliditydeveloper.com/max-contract-size) outlines the different things you can do to be within the limit.

TLDR:

- separate your contracts well (I was doing this),
- use libraries (I wasn't _really_ doing this, but this created a nice side effect of making the whole thing more readable),
- use proxies if the contract is deploying child contracts (see below)
- write only what you _must_ in contracts (see below)
- shorten error messages (ended up marking reverts with `"OW1"` and having comments at the top of the file to explain what they mean)
- change optimizer value to 1 (... well don't have to think TOO much about gas optimization if I'm on an L2 right?!)

### Proxies Galore

There are mainly 2 reasons I had to use proxies in the system:

1. The main contract was deploying child contracts, and I couldn't pack the child contract's bytecode into the main contract's without hitting limits.
2. We wanted to be able to upgrade contracts without having to redeploy the whole system. (at least until we were out of Beta)

Proxies are _confusing_ since there are different kinds that evolved over the years, with different use cases and tradeoffs and future consequences. OpenZeppelin has a [well written guide](https://docs.openzeppelin.com/contracts/4.x/api/proxy) but I'll be lying if I say I remember much. The annoying thing is that you have to be aware of the proxy pattern you're using (we were using Upgradable Beacon iirc), think through the workflows and then _change your contracts_ to fit the pattern. It's taking you away from the business logic you're trying to solve. It may be fun to learn, but definitely not fun to test. I believe the "Diamond Pattern" is in vogue now, but I didn't get the chance to use that (would overcomplicate things for no reason).

<diagram to show how crazy it got>

### What Should The Contract Do?

If gas was free (i.e. the EVM was infinitely scalable), this woudln't be something to consider. Just throw in convenience functions, store everything, make the frontend dumb as hell. But gas isn't free, so you have to think about exactly what smart contracts are for.
Smart Contracts are for replacing trusted third parties. They are not "just a database". They are for enforcing rules in a system, for which they need to store a minimum set of data. This minimum set of data will also be used by the frontend to display information to the user.
However, you get into the problem of third party dependency again here.

Let's take an example to illustrate the problem better. In Barren Wuffet, fund managers would be able to create IFTTT rules for the future, which would be automatically executed in the future. Now, EVM doesn't have an automatic TX execution hook like that (Cosmos SDK does), so someone has to send a TX to trigger the execution. The smart contract has to make sure whatever is being triggered matches the IFTTT rule the fund manager specified (otherwise the fund manager would have to run a bot to do it, mooting the whole point of an IFTTT system). To do this, the smart contract has to store some info as "proof". There are 2 ways of doing this:

1. Smart Contract stores the entire IFTTT rule with all its triggers, actions and their parameters and gives back an identifier for this rule. Later, someone can send a TX saying `execute(id)`. Smart Contract would lookup in a mapping `id => IFTTTRule`, check that on-chain trigger conditions are met and execute the actions. Frontend would only have to be aware of the id. This incurs extra gas cost at rule creation time.
2. Smart Contract stores the hash of the IFTTT rule (essentially a unique identifier). When some bot thinks it's time to execute the rule, it sends a TX saying `execute(IFTTTRUle)` and the smart contract checks that the hash of the IFTTT rule matches the hash it has stored. Frontend would have to be aware of the entire IFTTT rule. This incurs extra gas cost at execution time. This is more akin to the "UTXO model" of Bitcoin/Cardano.

In both these cases, you need to think about "data availability". Do you expect frontends to keep track of the entire history of the contract and lookup/reconstruct the data? Do you use something like `The Graph` that the frontend queries? What happens if `The Graph` goes down? (see more below). Does the gas cost economics make sense for the use case?

No clear answers!

### "Admin" Functions - Where's The Balance?

If you come from a web2 backend world, you're used to having a "superuser" who can do anything - mostly for fixing bugs. This is not the case in Ethereum. You can get to an abusive level of power with `onlyOwner` modifiers, but then would users use the product? You could decentralize the onlyOwner invocations with a DAO model, but would that have enough velocity to catch and fix bugs in the future?

No easy answers here either. What we ended up doing is limiting the `onlyOwner` functions to the bare minimum - pausing during an attack, upgrading the contract logic (and make it in a largely opt-in way), and then designing a kind of "emergency" escape hatch for the community to use in case they feel this power is being abused (not implemented when we shut it down). We wanted to retain the ability to move fast (i.e. have a small multisig of devs who could trigger these), with plans to decentralize some things in the future (DAO), deprecate the upgradability (why would you need this if it's been stable for a while on mainnet?) and then finally remove the `onlyOwner` functions altogether.

Open a dialogue with your community and see what they are comfortable with.

### Test, Test and Test Some More

I _hate_ writing tests. There was a running joke on the team - if you want psyf to be productive, ask him to write tests and he'll get everything else in the org done first just to avoid writing tests.
Jokes aside, props to madneutrino for setting up the first set of tests and getting me to write more (even though a redesign made us remove EVERY TEST I HAD PAINSTAKINGLY WRITTEN...UGHHHH). Tests meant I was able to refactor the codebase and make changes (add/remove) with confidence that I did not break something.

### Decimals...WHY?

The amount of time me and madneutrino wasted on figuring out how to handle decimals makes me want to cry. Different popular ERC20s use different decimals, chainlink returns different decimals for each price feed, you have to account for both decimals when doing math like combining ETH/USD and UNI/USD to ETH/UNI. Different smart contracts expect different decimals when you send them limits for slippage. Do you lose resolution if you go through this workflow? Do your order of operations potentially make something go to zero in error? Are there tokens you can't handle? Can you handle events like the UST crash?

It's a confusing mess. I wish we had just used 18 decimals everywhere.

### Can we just all agree to use WETH please?

The amount of code I had to write for different pathways of ERC20 and ETH for each integration (both input and output combinations) just makes me want to die.
I discovered Uniswap v3's way of writing contracts for ERC20 and using WETH wrapper for ETH way too late :sad-face:

## Infra and Tooling

### Chainlink, where's the Feed Regsitry on Arbitrum Bro?

Don't get me wrong. Chainlink is awesome and has powered a lot of the DeFi infrastructure. Their integration guides are nice and easy.
BUT, the contract has to know which contract to call to get ETH/USD, and which to call to get UNI/USD and so on...So you have to spend all that gas saving this into your contract and having a way to upgrade these.
Chainlink has rolled out a [Feed Registry](https://docs.chain.link/data-feeds/feed-registry) so the contract can query at runtime, but I have zero idea why this is only available on Etherum Mainnet. So now I have to write this extra contract to act as the chainlink registry when I deploy on other L2s (we were going to do this on Abitrum, and then mainnet/optimisim/otherL2s later).

WHY??!

### Gelato Is Delicious

I initially nerdsniped myself into the project by thinking about "good MEV" and how to align incentive mechanisms so that an IFTTT ecosystem could have strong guarantees. Ultimately, I caved in and just assumed we were going to use Gelato to start off.
Their docs were awesome and the devs in the discord were helpful. I was able to get a working prototype up and running in a few hours!

2 things I didn't like about using them:

1. when I was integrating, they had rolled out some new upgrades to their contracts and the docs hadn't been updated to reflect that. So I had to go and read the smart contracts and figure things out myself. They've now updated them, but I bring this up because I've seen this happen in multiple smart contract systems I was integrating with over this time (looking at you Uniswap).
2. Their bot is closed source and the network hasn't decentralized yet. So I had to write extra code to pretend to be the bot in my integration tests (using hardhat's mockAddress feature or whatever it's called). So even though I did assume we're going to use Gelato, I had to design the code such that it would be easy to swap out the Gelato integration for something else (like a bot we ran ourselves).

### Forking Is Awesome

I was dreading setting up hardhat for forking Arbitrum Mainnet, but it was actually pretty easy\*. This is perhaps my favourite feature of developing smart contracts. You just grab an API key from Alchemy and then you can fork mainnet and test your code against it. You don't have to write mocks for everything you're integrating with, just use hardhat to send TX from a different address. This is a HUGE time saver - to the point where I almost never had an isolated local node running. Quite proud of how frictionless the test suite was to run, to be honest.

### Testnets are Not As Clean As You Think

Perhaps this problem was exacerbated by my timing, but deploying to testnet was a pain because:

1. Some platforms had different contracts on testnet than they did on mainnet. So documentation was behind, sometimes the changes were not backwards-compatible, and I had to code in-faith that they'd have deployed to mainnet by the time we were ready to launch.
2. The Ropsten, Rinkeby and Kovan nets were being deprecated, and everyone hadn't moved over properly to Goerli yet.
3. Getting Goerli ETH was such a pain. Faucets were clogged or drained, and I had to waste multiple hours of my time and several days of CPU cycles to get enough testnet ETH. Not cool.

### Decentralized Price Feeds For Frontend

For all the hulla-balloo about decentralizing frontends that I've been hearing forever, I was really surprised when I couldn't find a decentralized price feed to show to the traders (think TradingView Charts on the frontend, marked by their positions and drawings).
ALL the options were centralized, and were too expensive for a beta launch, and had terrible granularity (once per 5 mins? really?). I'm talking spot prices, not even OHLC or something fancier. We have chainlink prices every block, why can't we just make a subgraph on the graph? We have DEXs with subgraphs, why can't they have prices? (sometimes one of a few of these things did exist, but NOT FOR L2s. UGH!). Why can I not easily delpoy the code of a subgraph to a different chain? Why can't I build on top of a subgraph's output without reindexing the entire chain?

Shoutout to my homies at Coingecko and Defillama for giving us free access to their APIs so we could at least keep building.

### Why Are Audits So Expensive?

Of course, being our first novel smart contract system, we wanted to make sure it was audited by someone more experienced. We used our experience, lessons learnt from Ethernaut, integrating knowledge of hacks we saw, and tools like Slyther and Mythril to get "audit ready". Yet, audits costs would be so prohibitively expensive (6 figures!) that we'd definitely not be able to do that without VC money or launching a token (neither of which we wanted to do at this point in time).

I'm not sure why 6 figure audits are a thing. Was that because of the bull market? It also looks like that auditor incentives are not aligned with the protocol's - auditors don't pay anything (except perhaps with reputation currency) in the case of a hack. Our strategy was to do a controlled launch (limited users, limited money that they could put in), until we had enough money to pay for an audit. We also planned on rolling out a small bug bounty right from the alpha release to incentivize people to find bugs. If I was doing it now, I'd probably be using GPT3 a bit for security analysis. A team (fortephy) has already done some work on using AI to do audits much cheaper. I'd probably talk to people from the Code4rena community to see if they could help us out as well.

## Frontend

### Writing TXs != POST

Writing TX to a blockchain have a lag, and it's not like your usual HTTP POST call. So your UI needs to capture all the error states properly.
[WAGMI](https://wagmi.sh/) helps you handle all this well in React Hooks.

### Take The Time To Search For Nice Libraries

We've already mentioned WAGMI above. This + [RainbowKit](https://www.rainbowkit.com/docs/installation) encapsulate a lot of the complexity of interacting with the blockchain - chain selection, transaction broadcasting, reading from the chain - in a nice way.

[TradingView](https://www.tradingview.com/HTML5-stock-forex-bitcoin-charting-library/) was also super useful for everything we wanted to do out of the box.

If we hadn't used these, we would have had to write a lot more code to handle all the edge cases.

### Where Does Frontend Get Info?

Firstly, it's important to use a decentralized service, but The Graph hadn't decentralized yet (not on L2s at least). Infura hasn't decentralized yet. Both have plans of doing so.
Secondly, you also have to decide what to use The Graph for, what to use the wallet's provider for, and what to use Alchemy for when the dapp is not connected to the wallet. (Every time I mentioned The Graph some dev that we were using The Graph, they sounded concerned and asked us about the backup plans.)

It's also interesting that token lists are maintained as JSON files in frontend repos. Then you have to maintain one for each testnet. Some don't have the tokens you want.
Similarly, you have to curate a list of addresses that are related to your protocol, times each chain you're deploying on.

<here's what our file looks like>

## Community

### Community Makes or Breaks This Space of Builders

One of the biggest surprises in my short dev career is how nice devs can be.
Uniswap was one of my first protocol integrations and I was having quite some trouble different aspects (especially with LP-ing). I was able to get help from CryptoRachel on their discord.
I was similarly having trouble with the GMX integration, and itburnz was able to help me out. Itburnz has since become a friend of the project and joined our discord. I've mentioend Gelato, Defillama and coingecko before.
I've also had the pleasure of taking help from various individuals who joined our twitter and startups that agreed to let us test their product for free (shout out to fortephy and \_ )
On the flip side, I've stopped working on some integrations and products because nobody answered my questions.

What I've found is as long as you are nice and respectful, describe the problem in the right level of detail and tell them what you've tried, people are generally willing to help you with pointers and advice.

### Having Usable, Upto Date Documentation is Important

This should be really self-explanatory for any open-source project, and even more so for a protocol. I won't belabour this, other than the fact that it inspired me to create [https://barren-wuffet.gitbook.io/barren-wuffet/](https://barren-wuffet.gitbook.io/barren-wuffet/) and think about how to structure it for various audiences.
