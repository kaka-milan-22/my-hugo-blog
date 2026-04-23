---
title: "Web 2.5 Is the Real State of Blockchain: A Three-Tier Classification of Major Crypto Products"
date: 2026-04-23T10:30:00+08:00
draft: false
tags: ["Web3", "Web2.5", "Blockchain", "Decentralization", "DeFi", "Bitcoin", "Ethereum", "Cryptocurrency"]
categories: ["Blockchain", "Investment"]
author: "Kaka"
description: "It's neither 'Web3 is here' nor 'Web3 is a scam' — it's Web 2.5. Using three dimensions (who can change the protocol / who can freeze your assets / can you self-rescue in a crisis), this post classifies every major crypto product across three tiers."
---

> If Web3 means truly decentralized, then we're currently in **Web 2.5** — and that framing is more honest than both "Web3 is here!" and "Web3 is a scam." It's the most explanatory mid-state description in the industry today.

For the past few years, discussion of blockchain has been dominated by two extremes: one side insists Web3 is taking over the world and will reshape finance and identity; the other believes the entire crypto industry is a Ponzi scheme with no substance. Neither is accurate.

What's actually happening is this: **blockchain has achieved Web3 in some dimensions, but remains Web2 in many more.** We're in the Web 2.5 era. This article does two things:

1. Fully unpack the Web 2.5 framework — what it **genuinely** adds that Web 2 could never do, and where it **hasn't broken through** yet.
2. Using a clear set of criteria (**who can change the protocol / who can freeze your assets / can you self-rescue in a crisis**), classify every major crypto product today into Web 2, Web 2.5, or Web 3.

After reading this, you'll be able to look at any new crypto product and quickly judge which layer of the stack it lives in — so you can price the risk correctly.

---

## 1. What Web 2.5 Really Added That Web 2 Could Never Do

### 1.1 The Ownership Structure of the Asset Layer Changed

The USDT, ETH, NFTs, and LP tokens in your wallet aren't entries in some company's database — they're **entries on a public ledger, signed by a private key that only you control**.

A concrete comparison:

| Scenario | Web 2 (Alipay / Bank) | Web 2.5 (Self-Custody Wallet) |
|----------|----------------------|-------------------------------|
| Platform shuts down | Assets frozen, wait in line for liquidation | Coins still on-chain, switch wallet software and continue |
| Platform locks your account | Appeals take months, or you never get it back | No one can lock it — the private key is yours |
| Cross-border transfer | SWIFT, multi-day settlement, fees | Seconds to settle, fully transparent fees on-chain |

This change alone is a **generational leap** — in Web 2 finance, only physical gold comes close. And **this doesn't get discounted because the frontend is centralized**: as long as you've moved coins to your own wallet, Coinbase could collapse tomorrow and your assets wouldn't move a cent.

### 1.2 Identity and Reputation Are Portable

A single `vitalik.eth` name works across every dApp.

Compare that to Web 2:

- Your Instagram account can't migrate to TikTok
- Your WeChat data can't leave Tencent
- Your LinkedIn network dies the day LinkedIn dies

A wallet address as identity is crude (a 42-character hex string), but **it belongs to you, not to any platform**. More importantly, experiments like DID, Proof of Humanity, and Farcaster are thickening this layer. In the next five years, something real could ship.

### 1.3 Application Composability Is Atomic

This is the fundamental transformation of **the business collaboration model** that most people miss.

In Web 2: Uber wanting to integrate Booking's hotel API needs 6 months of business negotiation, 12 months of API integration, and probably a per-call fee.

In Web 2.5: **Any contract can call any other contract without permission.**

- Yearn can stack on top of Curve as a yield aggregator — one line of code
- Aave can call Uniswap for liquidations — one line of code
- A solo developer can deploy a protocol in an hour that embeds the capabilities of ten existing protocols

This isn't a UI-level difference. This is a difference in **the entire mode of business cooperation** — from "sign contracts before integrating" to "just call it." Uniswap v3 has been called tens of thousands of times and has never charged a single "business cooperation fee."

### 1.4 State Is Globally Transparent

Tether's full reserve table, Uniswap's liquidity distribution, the history of any address — **free, real-time, undeletable**.

Web 2's "transparency reports" are ten orders of magnitude worse:

- Your bank's monthly statement is a curated summary
- Listed companies' quarterly reports come once per quarter, delayed by 45 days
- "Twitter transparency reports" are PDFs, not a real-time API

The market efficiency this on-chain data enables (arbitrage, risk management, on-chain analytics) is reshaping financial infrastructure. Even the Federal Reserve is researching Ethereum data for macroeconomic analysis — not opinion, but their published papers.

### 1.5 The Exit Right Is Hard

No matter how bad a protocol is, **you can always withdraw and leave** (as long as you aren't blacklisted, and the vast majority of on-chain products don't have blacklists).

In Web 2, if Facebook locks your account, you're powerless — your friends, photos, and memories are all in its hands. **This exit right itself is a fundamental transformation of user power.**

---

> These five points combined represent the **+0.5 progression** from Web 2 to Web 2.5. The key is: **this progression is irreversible.**
> Once self-custody becomes possible, composability is proven, and on-chain transparency becomes habitual, traditional finance must move in these directions too. RWA tokenization, CBDC design, J.P. Morgan's Onyx chain — they're all copying Web 2.5's homework.

---

## 2. Where Web 2.5 Still Falls Short of Web 3

But we also need to be equally honest that Web 2.5 is far from Web 3. The following five points are the **remaining 0.5**.

### 2.1 90% of Infrastructure Still Runs on AWS

Ethereum's top three node operators — Coinbase, Lido, and Binance — run most of their physical servers on AWS, GCP, and Hetzner.

> **Decentralized protocols running on centralized clouds** — a single AWS us-east-1 outage produces noticeable block delays on Ethereum.
>
> Decentralized protocol + centralized physical infrastructure = **fragility**.

### 2.2 The RPC Layer Is a Single Point

Infura + Alchemy account for 90% of Ethereum RPC traffic. Both are U.S. entities, and both **can and will** comply with regulators in blocking requests. MetaMask uses Infura by default, meaning every one of your transactions passes through a ConsenSys subsidiary by default.

Decentralized RPCs like Pocket Network and DRPC exist, but their UX and reliability are a tier below — and regular users don't actively switch.

### 2.3 Frontends and Domains Are Web 2

`uniswap.org`, `aave.com` — these are still Web 2 domains hosted on Web 2 clouds. One OFAC order can take them offline instantly.

IPFS + ENS frontends (e.g., `app.uniswap.eth.limo`) **do** exist, but aren't the default path — and regular users have no idea they exist. When Tornado Cash was sanctioned, its official frontend went down immediately, but the contract kept running — because the contract is Web 3, the frontend is Web 2.

### 2.4 Identity and Payment On-Ramps Remain Captured by Web 2

99% of new users enter crypto via Gmail + Coinbase KYC. **Captured at the gate** — you've already completed a full Web 2 identity review before you hold a single token.

Native "wallet-as-identity" users number in the millions, a rounding error compared to billions of global internet users.

### 2.5 Governance Is Actually Controlled by Capital

So-called DAO voting is bound by the top 10 token holders (usually VCs + insiders). The Curve/Convex war is fundamentally an **arms race to "buy governance"**.

Governance tokens = securities. Regulators just haven't said so yet. The SEC's investigation of Uniswap Labs is already moving down this thread.

---

## 3. The Mental Model: Five Layers of the Crypto Stack

You can map today's crypto world into these five layers:

| Layer | Representative Components | Decentralization Level |
|-------|---------------------------|------------------------|
| Consensus | Bitcoin / Ethereum protocol itself | **True Web 3** |
| Contract | A few immutable protocols (Uniswap V2, Liquity) | **True Web 3**, niche |
| Application | Most DeFi protocols (Aave, Curve, Lido) | **Web 2.5**, multisig + DAO + proxy upgrades |
| Frontend | `uniswap.org`, `aave.com` | **Web 2**, domain + cloud hosting |
| On-Ramp | Coinbase KYC, Gmail login | **Web 1.5**, KYC + banking |

Every DeFi interaction you make actually **traverses five layers with different degrees of decentralization** — the cryptographic guarantees at the bottom prevent confiscation of your assets, while the Web 2 components on top make it easy to use but also easy to intercept.

That's why Web 2.5 is such a **precise description**: not "half decentralized," but "different layers with different degrees of decentralization."

---

## 4. The Complete Product Classification (The Core of This Article)

Below, using the three dimensions I proposed — **who can change the protocol / who can freeze your assets / can you self-rescue** — I'll classify every major crypto product.

### 4.1 True Web 3 (Passes All Three Tiers)

These products satisfy: **immutable or hard-timelocked protocol, no admin (or minimal admin rights), optional decentralized frontend**.

| Product | Why It Fits |
|---------|-------------|
| **Bitcoin** | 10,000+ nodes globally, no "administrator," no company, founder disappeared. Protocol upgrades via miner + node soft consensus, no centralized power can censor a single transaction. **The only one that passes every dimension.** |
| **Ethereum (protocol layer)** | L1 itself is unstoppable, censorship resistance battle-tested through Tornado Cash sanctions. But Ethereum's **ecosystem** (RPC, Lido staking share, client diversity) has centralized elements. **Protocol is Web 3, ecosystem is Web 2.5.** |
| **Monero / Zcash (protocol)** | Native privacy. Monero is more hardcore (no optional transparency); Zcash has company influence but open-source protocol with no admin. True Web 3 of the privacy space. |
| **Uniswap V2 contracts** | Contracts are fully immutable, 0 admin, deployed 7 years ago without a single line changed. But as a **product** its frontend is `uniswap.org` (Web 2) and UNI is controlled by Uniswap Labs. **V2 contracts are Web 3; the product is Web 2.5.** Call V2 directly via Etherscan Write Contract and you get a pure Web 3 experience. |
| **Liquity (LUSD)** | Deposit ETH, mint LUSD stablecoin. Zero admin, zero governance token, IPFS frontend, anonymous core team. One of the rare full-stack-passing DeFi protocols — a "live specimen of true DeFi." |
| **RAI / Reflexer** | Similar philosophy to Liquity: zero governance, floating-peg algorithmic stablecoin. Niche but pure. |
| **Tornado Cash (protocol)** | Immutable contracts, no admin, survived OFAC sanctions. **Battle-tested proof that "protocol-level censorship resistance" is real.** |
| **Compound V2 contract layer** | Contracts themselves immutable, but governance can adjust parameters. **Contracts are Web 3; the protocol as a whole is Web 2.5.** |

> This list totals roughly 60–70% of crypto market cap (mainly via BTC and ETH), **but less than 10% of daily usage**. Most people's daily usage happens at the Web 2.5 application layer.

### 4.2 Web 2.5 (The Main Battlefield of DeFi)

These products: **self-custodied assets + centralized governance/operations + dependency on centralized infrastructure**.

#### DEXs and Lending

| Product | Why It Fits |
|---------|-------------|
| **Uniswap (product)** | V3/V4 has admin, governance token actually controlled by VCs + Labs, centralized frontend, Labs started adding whitelist filtering after the SEC investigation. But underlying contracts are immutable and LP positions can't be seized. |
| **Aave** | Lending king, proxy-upgradable, multisig admin can pause markets, governance can freeze assets (it has). Your collateral is self-custodied, but protocol parameters are fully controlled by the DAO (which in practice is a handful of whales). |
| **Compound V3 / MakerDAO (DAI)** | DAI is a typical hybrid — a large portion of collateral is USDC (Circle can freeze it), the PSM mechanism effectively lets MakerDAO act as an on-chain central bank, DAO voting is controlled by a small circle. **DAI is essentially a "semi-decentralized on-chain dollar."** |
| **Curve / Convex / Balancer / Yearn / GMX / Synthetix** | All of these share the standard Web 2.5 configuration: "upgradable core contracts + multisig admin + centralized governance tokens + self-custodied user assets." |
| **dYdX** | Migrated from Ethereum to its own Cosmos app-chain, with an off-chain order book (centralized matching) and on-chain settlement. Sits on the Web 2 side within Web 2.5. |

#### Staking and Liquidity

| Product | Why It Fits |
|---------|-------------|
| **Lido (stETH)** | Holds 30% of Ethereum staking — the single largest centralization risk for Ethereum. Multisig-managed node operator whitelist, upgradable contracts, LDO held by VCs. But stETH trades freely. **A textbook case of Web 2.5.** |

#### L2 Family

| Product | Why It Fits |
|---------|-------------|
| **Arbitrum / Optimism / Base / zkSync / StarkNet / Scroll / Polygon zkEVM** | **All current L2s have a centralized sequencer (single point of ordering) + admin upgrade keys** (able to replace the entire L2 codebase at any moment). L2Beat's decentralization scores rate most of them as Stage 0 or Stage 1 — none has reached Stage 2 (true decentralization). |

#### Identity, Wallets, Social

| Product | Why It Fits |
|---------|-------------|
| **ENS** | Individual ENS names self-custodied — no one can seize them — but the `.eth` root domain's controller is a Safe multisig (ENS DAO), which could in theory change pricing or even revoke. Much more decentralized than DNS, but not fully. |
| **MetaMask / Rabby / Trust Wallet** | Self-custody wallets with private keys in your control, but default RPC is centralized (Infura/Alchemy), software updates pushed by companies, and wallet companies can add address blacklists in the frontend (MetaMask has already blocked certain addresses). **"Self-custody but not censorship-resistant."** |
| **Farcaster** | Claims decentralized social, protocol is open-source, but in practice 99% of traffic goes through Merkle Manufactory's centralized hubs. **Nominally Web 3, actually Web 2.5.** |

#### Stablecoins by Tier

| Stablecoin | Tier | Reason |
|------------|------|--------|
| **LUSD** | True Web 3 | Pure ETH collateral, zero governance |
| **DAI** | Web 2.5 (hybrid) | Collateral includes large amounts of USDC, indirectly influenced by Circle |
| **GHO** | Web 2.5 | Issued by Aave, relatively centralized governance |

### 4.3 Web 2 (Avoid Zone)

These products "just happen to run on crypto rails" — fundamentally centralized.

#### Exchanges

| Product | Why It Fits |
|---------|-------------|
| **Binance / Coinbase / OKX / Kraken / Bybit / Bitget** | Essentially **digital banks** — your coins are in their hands, they can freeze, move, or go bankrupt. Same nature as Alipay — just the underlying assets are crypto. |

#### Stablecoins (Centralized Issuer)

| Product | Why It Fits |
|---------|-------------|
| **USDT / USDC** | Even though they live on-chain and can be self-custodied, **issuers can unilaterally blacklist your address and destroy your balance**. The "can freeze your assets" check completely fails. But note — holding them on-chain is still more free than in a bank (no ID lookup), so **practical utility exceeds the decentralization score**. |

#### Chains Pretending to Be Public Chains

| Product | Why It Fits |
|---------|-------------|
| **Tron** | Super Representatives (SRs) system of only 27 nodes, Justin Sun effectively controls the ecosystem. **A blockchain technically, a private company practically.** |
| **Ripple / XRP** | Company retains vast token reserves, validator nodes permissioned by Ripple Labs. Classic Web 2 in disguise. |
| **BSC (BNB Chain)** | 21 validators all hand-picked by Binance. **A Binance-backed private chain.** |
| **Solana (disputed)** | Protocol open-source but validator economic threshold is extremely high, foundation has huge influence, network has crashed multiple times. **I categorize it as Web 2.5, but leaning toward Web 2.** |

#### NFT Marketplaces

| Product | Why It Fits |
|---------|-------------|
| **OpenSea / Blur** | NFT marketplace matching and royalty enforcement both happen on centralized off-chain servers, and they've censored multiple NFT collections. **The NFT asset is Web 2.5, but the platform is Web 2.** |

---

### 4.4 The Real Distribution of Traffic and Market Cap

Stacked together, the industry's actual reality:

- **The most hardcore Web 3** = Bitcoin + Ethereum protocol + Monero + Liquity + Uniswap V2 contracts + Tornado Cash. This list totals 60–70% of market cap (mainly via BTC and ETH), **but less than 10% of daily usage**.
- **Mainstream DeFi is basically Web 2.5.** If you're using Aave, Uniswap, Lido, Curve, or Arbitrum, you're in the Web 2.5 ecosystem. Your assets are much more free than in a bank (self-custodied), but the protocols themselves aren't magic.
- **Web 2 CEXs and stablecoins** actually carry 80%+ of crypto financial traffic. Tether + Coinbase + Binance combined is the crypto world's "JPMorgan + Chase + Visa."

---

## 5. Historical Analogies and Timeline

### 5.1 Web 1.0 → 2.0 Took 10–15 Years

The 1995–2010 history is worth revisiting:

- For the first few years, many called "the internet a scam" — e-commerce didn't work, search was unusable, social networks didn't exist, connections were slow
- Webvan, Pets.com and similar early e-commerce did go bankrupt
- But the infrastructure (HTTP, DNS, TCP/IP) was **irreversible once built**
- Over the next 15 years, "Web 2 applications" grew on top: Google, Facebook, YouTube, Netflix, Uber

### 5.2 Web 2.5 → Web 3 May Also Take 15–20 Years

We roughly entered Web 2.5 in 2017–2018 (ICO bubble + early DeFi Summer). It's now 2026, so 8 years of progress.

By the Web 1 → 2 analogy, there's another 7–12 years to some mature state.

**But the path won't be linear:**

- There will be bear markets where Web 2 takes over
- Regulatory reversals
- Technology roadmap switches
- New centralized forces emerging (like Lido today)

Calling "Web3 vaporware" today is like calling "e-commerce a bubble" in 2002 — **you were right about the timing, but wrong about the direction.**

### 5.3 There's Also a Pessimistic Scenario: Web 2.5 May Be the Endpoint

To be honest, **Web 2.5 may never reach Web 3.** Three forces all push the industry toward Web 2.5, with no force pushing it toward Web 3:

1. **Regulatory pressure** makes every protocol add admin (for self-preservation)
2. **Capital structure** makes every protocol issue governance tokens (for monetization)
3. **UX requirements** make every product need customer service and rescue flows (for growth)

30 years from now, the real legacy of the "Web3 movement" may not be decentralization — it may be **giving traditional finance digital rails**: BlackRock's BUIDL, Circle's USDC, Coinbase's Base. These will swallow the market, while truly decentralized protocols like Liquity and Uniswap V2 become permanent "cypherpunk preserves."

### 5.4 My Prediction: A Hybrid Path

**Mainstream moves to "Web 2.5 + regulatory compliance," while a small but critical set of extremely decentralized protocols exists permanently as a floor.**

Most people won't use Liquity directly, but **the fact that Liquity exists constrains the evil-doing room of every centralized player.**

> It's like nuclear weapons — they don't exist to be used. They exist **to make not-using possible.**

---

## 6. Practical Recommendations for Everyday Users

### 6.1 Acknowledging "Web 2.5 Has Meaning" Is the Most Honest Entry Posture

This recognition lets you:

- **Not get sucked in by utopian narratives** (FOMO into scams)
- **Not get dragged astray by cynicism** (miss the real progress)
- **Clearly identify which layer each product lives in** (price risk correctly)
- **Position yourself at the appropriate layer to participate** (investing, building, using)

### 6.2 A Three-Tier Usage Strategy

| Scenario | Recommended Tier | Reason |
|----------|------------------|--------|
| Long-term BTC/ETH holding | True Web 3 (self-custody) | Maximize censorship resistance |
| Daily DeFi interaction | Web 2.5 (Aave, Uniswap) | Best UX and efficiency |
| Fiat on/off-ramps | Web 2 (CEX) | Compliance channel |
| **Critical moments (sanctions, account freezing risk)** | **Switch to true Web 3** | Use Liquity, IPFS frontends, Tornado Cash |

**Use Web 2.5 daily, know Web 3 exists in the background, switch to Web 3 at critical moments to protect yourself** — this is the actual operating pattern of crypto veterans over the past 5 years, and it will still hold for the next 5.

### 6.3 Hands-On Suggestion: Try a True Web 3 Experience Once

Paper theory isn't enough. Pick one thing to try today:

- **Option A**: Open [Liquity](https://www.liquity.org/), deposit some ETH, borrow some LUSD. Going through the flow makes you realize what "zero governance, zero admin" actually feels like.
- **Option B**: Use an IPFS frontend to access Uniswap V2 (`app.uniswap.eth.limo`, or directly call the contract via Etherscan's Write Contract) to do a swap. Compared to `uniswap.org` you'll see how critical the frontend layer is.
- **Option C**: Run a Bitcoin full node (Umbrel or Start9 bring the barrier way down). You'll feel, for the first time, what "verifying consensus yourself" is.

After doing it once, you'll have **concrete understanding** of "true Web 3 experience" — and when you return to Web 2.5 products, you'll have a real mental yardstick.

---

## 7. Conclusion

The greatest value of the Web 2.5 framework is that it decomposes "progress" into **measurable dimensions**:

- Which features are already truly decentralized
- Which are still Web 2
- Where the waterline rises each year

This is 100× more useful than debating "crypto is a revolution / crypto is a scam" — the former is an **actionable observation**, the latter is a **tribal emotion**.

Keep watching Etherscan, reading contracts, comparing governance structures across protocols, and you'll develop **"decentralization connoisseurship"** — a scarce skill at the crypto/finance intersection over the next decade, more valuable than any specific technical skill.

---

## References

- **L2Beat** (`l2beat.com`) — L2 decentralization scoring; check the Stage 0 / 1 / 2 classification
- **Etherscan** — Interact with contracts directly, bypassing official frontends
- **DeFiLlama** — Transparent TVL and governance structure data across protocols
- Vitalik Buterin's *"The Most Important Scarce Resource is Legitimacy"* — understand the power structure of the governance layer
- *"Why Decentralization Matters"* — Chris Dixon's classic long-form, the original articulation of Web 3 values
