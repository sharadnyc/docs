# PersonalOS — All Architecture Decision Records

_20 ADRs covering foundation, product, trust & compliance, expansion, and IP strategy._

---

## Table of Contents

| ADR | Title |
|---|---|
| [0001](#adr-0001) | ADR-0001: Continuous Real-Time Exchange Matching |
| [0002](#adr-0002) | ADR-0002: Base Chain + USDC as Primary Payment Rail |
| [0003](#adr-0003) | ADR-0003: E2E Encrypted Ledger with Local Cultivation |
| [0004](#adr-0004) | ADR-0004: Arweave via Irys for Ledger Storage |
| [0005](#adr-0005) | ADR-0005: Coinbase Smart Wallet for Soul Wallets |
| [0006](#adr-0006) | ADR-0006: Soul Cold Start and Onboarding |
| [0007](#adr-0007) | ADR-0007: Notification Strategy |
| [0008](#adr-0008) | ADR-0008: Data Enrichment Strategy |
| [0009](#adr-0009) | ADR-0009: Exchange Supply Bootstrapping |
| [0010](#adr-0010) | ADR-0010: Brand Fraud Prevention |
| [0011](#adr-0011) | ADR-0011: Insight Correction Model |
| [0012](#adr-0012) | ADR-0012: Konnection Resilience and Explainability |
| [0013](#adr-0013) | ADR-0013: KYC and Withdrawal Design |
| [0014](#adr-0014) | ADR-0014: Soul Retention Model |
| [0015](#adr-0015) | ADR-0015: Conversational Advisor Architecture |
| [0016](#adr-0016) | ADR-0016: Brand Reporting and Data Minimisation |
| [0017](#adr-0017) | ADR-0017: Health Data and Preventive Care |
| [0018](#adr-0018) | ADR-0018: Go-to-Market and Soul Acquisition |
| [0019](#adr-0019) | ADR-0019: Competitive Moat |
| [0020](#adr-0020) | ADR-0020: IP and Patent Strategy |

---

# ADR-0001: Continuous Real-Time Exchange Matching

## Status
Accepted

## Context
The Exchange must match Brand Listings to Souls based on their Insights. The question was when to run matching: after each Cultivation (batch), when a Soul opens the app, or continuously.

## Decision
The Exchange runs continuous real-time matching. Two events trigger a match pass:
1. A new Listing is posted → match against all eligible Souls
2. A Cultivation completes for a Soul → match that Soul against all active Listings

## Alternatives Considered
- **Post-Cultivation batch** — simpler infrastructure, but introduces latency between fresh Insights and Offer delivery. A Soul with a new high-value Insight could miss a time-sensitive Listing.
- **On app-open** — ties matching to engagement, not data freshness. A Soul who doesn't open the app for a week misses all matches during that window.

## Consequences
- Requires an event-driven matching service that reacts to both Listing and Cultivation events.
- Higher infrastructure complexity than batch matching.
- Ensures Offers always reflect the freshest Insights and Listings are filled as fast as possible.
- Infrastructure cost is self-funding: more timely matches drive more Claims, which drives more take-rate revenue.

## Operational Resilience
Event-driven pipelines can fail. Backfill strategy:
- **Durable event queue** (e.g. SQS or Kafka) — events survive consumer failures; no match is silently dropped.
- **Idempotent matching consumers** — replaying the same event produces the same result; no duplicate Offers.
- **Dead letter queue** — failed events are held for inspection and retry rather than lost.
- **Smart contract as payment source of truth** — for Claim settlement, the Base chain is the authoritative record. If the backend fails mid-Claim, payment state can be reconstructed by replaying on-chain events via Alchemy.

---

# ADR-0002: Base Chain + USDC as Primary Payment Rail

## Status
Accepted

## Context
Claims trigger Yield payments from Brands to Souls. We needed a payment rail for these micro-payments. Options included fiat (Stripe, UPI) and crypto.

## Decision
USDC stablecoin on Base (Coinbase L2) is the primary Payment Rail. When a Claim is made, USDC moves atomically from the Brand's Budget escrow to PersonalOS (fee) and the Soul's Wallet via smart contract. Fiat off-ramps (Stripe for US, UPI for India) are available to Souls via Coinbase's built-in on/off-ramp.

## Alternatives Considered
- **Stripe-first** — simpler to start, familiar, but requires separate solutions for India (UPI), creates credit risk, and doesn't align with the data sovereignty narrative.
- **Bitcoin** — too volatile for micro-payments, high transaction fees, slow settlement. Wrong tool for $2–20 Claim payments.
- **Ethereum mainnet** — $1–10 gas fees per transaction make micro-payments economically unviable.
- **Solana** — cheapest and fastest, but different dev stack (Rust/Anchor) and weaker fiat on/off-ramp story.

## Consequences
- Atomic Claim settlement via smart contract eliminates payment processing lag and credit risk.
- Coinbase's built-in on/off-ramp solves the fiat conversion problem without building it.
- Base is EVM-compatible — standard Solidity tooling applies.
- Brands must fund Listings in USDC (pre-funded Budget), which may create friction for non-crypto-native Brands.

---

# ADR-0003: E2E Encrypted Ledger with Local Cultivation

## Status
Accepted

## Context
The Ledger holds a Soul's raw Transaktions. The question was who can read them and where Cultivation (Insight computation) runs.

## Decision
Transaktions are end-to-end encrypted with the Soul's passkey before leaving the device. PersonalOS never holds the decryption key. Cultivation runs entirely on the Soul's iOS device against locally decrypted data. Only the resulting Insight scores are sent to PersonalOS servers.

## Alternatives Considered
- **Platform-readable Ledger, server-side Cultivation** — simplest to build, but contradicts the platform's core promise. If PersonalOS can read all Transaktions, it is functionally indistinguishable from the data brokers it claims to replace.
- **Encrypted at rest, platform-decryptable** — protects against external breaches but not against platform misuse or legal compulsion. Does not support the sovereignty narrative.

## Consequences
- PersonalOS cannot compute Insights centrally — Cultivation logic must ship in the iOS app.
- Insight computation quality is bounded by what can run efficiently on a mobile device.
- Server-side ML models for Insight generation are not possible without a privacy-preserving compute layer (e.g. federated learning) — a future consideration.
- This is a genuine, auditable privacy guarantee: PersonalOS legally and technically cannot access raw Transaktions.

## Trust Assurance
Trust is hard to earn and harder to sustain. The architecture enforces the privacy promise rather than just stating it — but the promise also needs to be verifiable by Souls:
- **Open-source the Cultivation logic** — Souls and auditors can verify that on-device computation does what it claims and that no raw data is exfiltrated.
- **Independent security audit** — commission a third-party audit of the encryption scheme and Cultivation code before launch. Publish the results.
- **Invisible UX** — encryption must be transparent to the Soul. The experience is: connect bank → data is safe → earn Yield. No key management or cryptographic concepts surfaced. Passkey/biometrics (Face ID / Touch ID) are the only interaction point.

---

# ADR-0004: Arweave via Irys for Ledger Storage

## Status
Accepted

## Context
The Ledger must be stored somewhere durable. Options ranged from PersonalOS servers (centralised) to various decentralised storage networks. Since Transaktions are E2E encrypted (ADR-0003), PersonalOS cannot read the ciphertext regardless of where it is stored — but the choice affects durability, sovereignty, and cost.

## Decision
Encrypted Transaktion batches are written permanently to Arweave via the Irys bundling service after each Harvest. PersonalOS holds no copy. The Arweave content hash is the canonical reference to a Soul's Ledger state.

## Alternatives Considered
- **PersonalOS servers (S3-equivalent)** — cheapest (~$5–10k/year at 100k Souls), but PersonalOS controls the storage. A subpoena, breach, or company failure could destroy Soul data. Inconsistent with sovereignty narrative.
- **Arweave + Lit Protocol** — adds threshold key recovery (Soul can recover Ledger even if passkey is lost). Cost is ~$50k/year at 100k Souls vs. ~$100/year for Arweave alone. Deferred: iCloud Keychain backup covers the key recovery problem for the iOS-first launch.
- **Ceramic Network** — strong DID-based identity model, but higher operational complexity and less proven at scale.
- **IPFS** — not permanent by default; requires pinning services that reintroduce centralisation.

## Consequences
- Arweave storage costs are negligible (~$100/year at 100k Souls via Irys).
- Data is permanent and cannot be deleted — consistent with Ledger semantics ("nothing deleted, only superseded").
- If a Soul loses their passkey and iCloud Keychain backup, their Ledger is permanently unreadable. Acceptable for iOS-first launch; revisit with Lit Protocol if key loss becomes a material support issue.
- Irys batches multiple writes into single Arweave transactions, keeping per-Soul write costs near zero.

## GDPR Right to Erasure
Arweave's permanence creates tension with GDPR Article 17 (right to erasure). This is resolved via **crypto-shredding**:

1. On a Soul's deletion request, PersonalOS deletes all Arweave content hash references from its own servers.
2. The Soul's passkey is revoked, destroying the only key that can decrypt the Ledger.
3. The ciphertext on Arweave becomes permanently unreadable — effectively erased even though the bytes persist.

This approach is increasingly accepted by regulators as satisfying the spirit of right to erasure. However, it is not uniformly blessed across all EU jurisdictions. **EU legal review is required before any EU market expansion.** US-first iOS launch proceeds under this model.

---

# ADR-0005: Coinbase Smart Wallet for Soul Wallets

## Status
Accepted

## Context
Each Soul needs a Wallet to receive USDC Yield. The question was whether PersonalOS holds custody of Soul funds or Souls hold their own keys.

## Decision
Each Soul's Wallet is provisioned via Coinbase Smart Wallet on Soul signup. Keys are custodied via passkey/biometrics (iCloud Keychain on iOS) — no seed phrase required. Yield is deposited directly on-chain to the Soul's address. PersonalOS cannot access Wallet funds. Souls can export their wallet at any time for full non-custodial sovereignty.

## Alternatives Considered
- **PersonalOS custodial Wallet** — simplest UX, but PersonalOS controls Soul funds. A freeze, breach, or regulatory action could block Soul access to their Yield. Directly contradicts the platform's data and financial sovereignty promise.
- **Fully non-custodial (seed phrase)** — Soul holds keys entirely, maximum sovereignty. But seed phrase onboarding has high drop-off and irreversible loss risk. Not viable as a default for a consumer app.
- **Other embedded wallet providers (Privy, Dynamic)** — viable alternatives, but Coinbase Smart Wallet is native to Base chain, has the strongest fiat on/off-ramp story via Coinbase, and aligns with the rest of the Base/USDC stack.

## Consequences
- No seed phrases in the onboarding flow — Soul signs in with passkey/biometrics.
- iCloud Keychain backs up the passkey, making key recovery accessible to iOS users without extra friction.
- PersonalOS is technically and legally unable to freeze or access Soul Wallet funds.
- Coinbase's on/off-ramp enables fiat Withdrawal without PersonalOS building payment infrastructure.

---

# ADR-0006: Soul Cold Start and Onboarding

## Status
Accepted

## Context
On first Harvest, a new Soul may have limited recent transaction history. If Cultivation produces weak or no Insights on Day 1, the Soul sees no value, sets no Permits, earns no Yield, and churns before the platform has demonstrated its core promise.

## Decision
Plaid is configured to backfill 24 months of transaction history on the first Harvest. This maximises Insight density from Day 1. Weak Insights (confidence below threshold) are shown with coaching prompts — "Connect 2 more accounts to strengthen these signals and unlock higher Yield floors" — turning the gap into a progress mechanic. Depth becomes a visible score the Soul is motivated to improve, not a wall they hit.

## Alternatives Considered
- **Show nothing until Insights are strong** — honest but anticlimactic; high Day 1 churn before Soul sees value.
- **Pull recent transactions only** — faster onboarding, weaker first-day Insights, misses historical patterns.
- **Wait 30 days before showing Insights** — maximises quality but loses the Soul before they see any value.

## Consequences
- Plaid backfill may take 2–4 minutes on first Harvest; UX must set expectations with a progress indicator.
- Coaching prompts create a natural Konnection expansion loop — each new account connected improves Depth and unlocks stronger Signal Types.
- Depth score becomes the primary onboarding progress indicator and the Soul's incentive to maintain Konnections.
- 24-month backfill enables seasonal pattern detection that recent-only data cannot produce.

---

# ADR-0007: Notification Strategy

## Status
Accepted

## Context
The Exchange runs continuous real-time matching. Without throttling, Souls could receive multiple Offer notifications per day, causing fatigue and uninstalls. Too few notifications means missed Yield and Soul disengagement. Every PersonalOS notification is a trust asset — once Souls learn to ignore them, they stop Claiming.

## Decision
Hybrid notification model combining two controls:
1. **Quality threshold** — an Offer notification is only sent if the bid meaningfully exceeds the Soul's Yield floor (not just meets it). This filters low-value interruptions automatically.
2. **Soul-controlled cadence** — default of 2 high-value Offers per day, ranked by bid-above-floor. Within the daily cap, the best Offers surface first. Cadence preference is surfaced progressively: Day 1 runs on defaults, settings are exposed after the Soul's first Claim.

## Alternatives Considered
- **No throttle** — maximises Yield opportunity but causes notification fatigue and uninstalls.
- **Daily digest** — low fatigue but kills the live marketplace feeling and reduces Claim rate.
- **Quality threshold only** — no Soul control; paternalistic.
- **On app-open only** — ties Offer delivery to engagement rather than freshness; penalises Souls who don't open the app frequently.

## Consequences
- Every PersonalOS push notification becomes a trust asset — Souls learn it is always worth opening.
- The matching engine must rank Offers by bid-above-floor before fanning out notifications.
- Some Offers will expire before a Soul's daily cap opens — an acceptable tradeoff for notification trust.
- Cadence settings must be discoverable in settings but not forced on Day 1 before the Soul has context.

---

# ADR-0008: Data Enrichment Strategy

## Status
Accepted

## Context
Card transaction data via Plaid reveals purchasing power and merchant-of-choice but not item-level purchasing taste — what the Soul actually bought. A transaction reading "AMAZON.COM $127.43" tells us spending power; it does not distinguish a LEGO set from noise-cancelling headphones. Richer item-level data improves Insight precision and enables Signal Types that financial data alone cannot serve.

## Decision
Hybrid three-track enrichment model, designed to avoid single-source dependency:

1. **Direct loyalty API integration** — where retailer APIs exist (Kroger Developer API, Starbucks), integrate directly for item-level purchase history.
2. **Aggregator-as-Provider** — a data aggregator such as Fetch Rewards provides broad item-level coverage across thousands of retailers via a single integration. Fetch is designated as the preferred **dual-role partner**: Provider for item-level data AND Brand for offer inventory — a single commercial relationship delivering two product integrations.
3. **Soul-initiated export** — for retailers without APIs, Souls can export their own loyalty data (via CCPA/GDPR data rights) and upload to PersonalOS. High friction; used as fallback only.

Amazon purchase history is explicitly deferred. No public API exists. Email receipt parsing is technically feasible but requires inbox access — a significantly higher trust ask than bank connections, inconsistent with the platform's data minimisation principles.

## Alternatives Considered
- **Email receipt parsing** — feasible without Amazon cooperation; scoped to order confirmation emails. Deferred to Phase 2. Too invasive for MVP given the trust bar required.
- **Amazon OAuth partnership** — Amazon has its own advertising business built on this data. Not available at this stage; Phase 3 aspiration.
- **Card data only** — sufficient for MVP Signal Types but limits Insight precision over time. Acknowledged gap.

## Consequences
- No single external dependency owns item-level data supply; one provider changing their model does not break the enrichment layer.
- Fetch Rewards dual role (Provider + Brand) is the highest-priority data enrichment business development conversation.
- Loyalty API integrations require separate maintenance per retailer as APIs evolve.
- Item-level data produces richer Signal Types (e.g. `consumer.organic_food_preference`, `consumer.electronics_enthusiast`) that financial data cannot serve.

---

# ADR-0009: Exchange Supply Bootstrapping

## Status
Accepted

## Context
Cold-starting direct Brand relationships takes 6–12 months. Without adequate Offer inventory on Day 1, Souls earn little Yield and churn before the Exchange demonstrates its value. The platform needs immediate supply to run the founding cohort experiment.

## Decision
**Intermediary-as-Brand model.** A coupon and offer intermediary (priority outreach order: Ibotta, Kard, Inmar, Quotient) onboards as a single Brand on the Exchange with a pre-funded USDC Budget. They bring their existing merchant offer inventory. PersonalOS provides the targeting precision the intermediary cannot achieve through their own platforms. The intermediary bills their merchant partners independently; PersonalOS earns its take rate on every Claim.

One contract, one integration, access to thousands of merchant offers from Day 1.

## Brand Value Proposition
This model also crystallises the core pitch to direct Brands:

> *"PersonalOS is the only platform where the consumer has already declared they are in the market, named their price, and agreed to hear from you — before you spend a dollar."*

Current ad platforms (Google, Meta) target based on browsing behaviour — a lagging signal. A Soul who receives a travel ad after booking has already spent their money. PersonalOS's Cultivation detects intent *before* the purchase, grounded in actual transaction behaviour, not search history. Souls who set Permits are active, willing participants — not passive targets. Structural conversion rates are 10–40x higher than Google/Meta's 1–2% industry average because 100% of the audience has opted in and declared intent.

## Alternatives Considered
- **Full white-label of intermediary inventory** — fast supply, dilutes precision narrative, Yield per Claim drops to $0.50–$2 vs. $3–$20 for direct Brand Listings.
- **Direct Brand relationships only** — correct long-term model, too slow for founding cohort launch.
- **Hybrid low/premium tier** — intermediary fills low-Yield, Brands fill premium. Considered but deferred; simpler to start with a clean intermediary-as-Brand model, then migrate to tiered as direct Brands join.

## Consequences
- Early Offers will skew toward CPG, grocery, and retail rather than premium automotive or health categories.
- The intermediary relationship proves the Exchange model and generates conversion data that becomes the pitch to direct Brands.
- Once direct Brands join with higher bids, the intermediary naturally occupies the lower-Yield tier without a structural change.
- Intermediary must pre-fund USDC Budget escrow like any other Brand (ADR-0010).

---

# ADR-0010: Brand Fraud Prevention

## Status
Accepted

## Context
Without fraud controls, bad actors can post fraudulent Offers — fake gift cards, phishing links, misleading claims — to harvest Soul interaction data at low cost. A Soul who receives one fraudulent Offer loses trust permanently. The Exchange's value depends entirely on Souls trusting that every Offer is legitimate.

## Decision
Two-layer fraud prevention:

1. **Funded USDC escrow as primary deterrent** — a Brand must pre-fund a USDC Budget in the Listing escrow contract before the Listing activates. No Budget, no active Listing. This makes low-effort scam operations economically unviable — a fraudster must commit real capital to play.

2. **Hybrid content review** — the first Listing from any Brand undergoes manual review before activation (target: <24 hours). Subsequent Listings from trusted Brands pass through automated content screening only. Bad actors are caught at entry; established Brands move at automated speed.

Minimum Budget escrow threshold to be defined during Brand onboarding design — set high enough to deter disposable fraud accounts, low enough not to block legitimate small Brands.

## Alternatives Considered
- **Manual review only** — reliable fraud detection but creates a bottleneck that does not scale past a few hundred Brands.
- **Automated only** — scales infinitely but misses novel fraud patterns; a sophisticated bad actor gets through at least once before the model catches up.
- **No review** — catastrophic trust risk; one fraudulent Offer reaching a Soul causes disproportionate damage.

## Consequences
- Listing activation latency for new Brands during manual review window — must be clearly communicated in Brand onboarding.
- Automated screening must cover at minimum: phishing links, fake redemption codes, misleading claims, impersonation of known brands.
- Escrow requirement is also a Brand quality signal — Brands with funded Budgets have genuine commercial intent.
- Fraud detection patterns from manual review feed the automated screening model over time.

---

# ADR-0011: Insight Correction Model

## Status
Accepted

## Context
Cultivation makes mistakes. A Soul incorrectly identified as an automotive buyer (e.g., they were researching for someone else) will receive irrelevant Offers and lose trust. Souls must have a correction mechanism. However, correction creates gaming risk — Souls could dispute accurate Insights to manipulate matching or avoid certain Brand categories.

## Decision
**Two-layer correction model:**

1. **Permit toggle** — immediate suppression. Soul turns off the Permit for an Insight category; Offers stop immediately. No model retraining required. Lightweight and instant.

2. **Explicit "This isn't me" Insight-level dispute** — Soul taps an Insight and disputes it. Matching is suppressed immediately. The dispute is flagged in the Cultivation model and feeds back into Insight recalibration over time.

**Gaming risk is structurally contained:** Cultivation is grounded in actual transaction behaviour. Fabricating qualifying transactions requires spending real money — the cost of gaming almost always exceeds the Yield earned. Disputing accurate Insights *reduces* Yield (fewer Offers), so gaming runs against the Soul's own financial interest.

**Safeguards:**
- 7-day cooldown before a disputed Insight can be re-enabled — prevents rapid toggling to probe the matching algorithm.
- Dispute-rate monitoring — Souls disputing >40% of their Insights are flagged for review; may indicate manipulation or a Cultivation quality problem.
- Disputed Insights do not boost alternative Insight scores — the model treats disputes as null signal.

## Alternatives Considered
- **No correction mechanism** — Cultivation self-corrects over 30–90 days as behaviour diverges. Soul receives wrong Offers throughout. High trust damage.
- **Permit toggle only** — immediate suppression without feedback loop; Cultivation never improves from wrong calls.
- **Automatic correction only** — removes Soul agency; feels like the platform is deciding what the Soul is.

## Consequences
- Dispute signals create a Cultivation quality feedback loop — repeated disputes on a Signal Type indicate a model problem to investigate.
- Brands tracking downstream conversion will see lower rates from disputed-Insight cohorts — the market self-corrects bid pricing without PersonalOS intervening.
- Cooldown and dispute-rate monitoring must be implemented in the matching engine and Soul profile service.

---

# ADR-0012: Konnection Resilience and Explainability

## Status
Accepted

## Context
Plaid connections expire or require re-authentication after bank security updates, often silently. Stale Ledger data degrades Insight confidence, reduces Offer qualification, and causes Yield to drop — without the Soul understanding why. Silent degradation is one of the most common churn causes in Plaid-based apps.

## Decision
**Graceful degradation with rationalized explainability.** When a Konnection breaks or data goes stale, PersonalOS surfaces the full causal chain to the Soul in plain language:

> *"Chase hasn't synced in 12 days → your automotive Insight confidence has dropped from 0.78 to 0.61 → you've missed 3 Offers above your Yield floor this week. Reconnect in one tap to resume."*

The explainability is not optional — it is the trust mechanism. Souls who understand *why* their Yield is declining are motivated to fix it. Souls who see Yield drop with no explanation assume the product is broken.

**Response protocol:**
1. Detect broken Konnection or data staleness.
2. Show Depth score declining with clear causal attribution in-app.
3. Send one push notification — not repeated harassment.
4. Surface one-tap Plaid re-auth via update mode (no full re-import of history).
5. Auto-pause Permits after 30 days of stale data to protect Brand ROI — communicated to Soul as: *"We've paused your Offers to keep them relevant. Reconnect to resume."*

## Alternatives Considered
- **Silent failure** — Soul notices only when Offers dry up; by then churn is likely.
- **In-app notification only** — easy to miss; not sufficient for a Yield-impacting event.
- **Aggressive repeated prompts** — damages trust; feels like harassment for a technical issue outside the Soul's control.

## Consequences
- Depth score must be visually connected to Konnection health in the UI — declining Depth has a clear, named cause.
- Auto-pause logic requires stale-data detection in the matching engine with a configurable threshold (default: 30 days).
- The causal chain display requires tracking Offer miss events attributed to data staleness — these are valuable product analytics beyond their UX role.
- Explainability applies to all degradation events, not just Plaid re-auth — including Cultivation failures, Arweave write errors, and Insight score drops.

---

# ADR-0013: KYC and Withdrawal Design

## Status
Accepted

## Context
Coinbase's fiat off-ramp requires identity verification above certain thresholds (~$1,000 cumulative). If KYC is a surprise at withdrawal time — after a Soul has earned Yield and expects instant access — trust breaks disproportionately. PersonalOS must also assess whether it is itself classified as a Money Services Business (MSB) or payment facilitator under applicable law.

## Decision
**KYC delegated entirely to Coinbase.** PersonalOS does not build its own identity verification infrastructure for MVP. Rationale: Coinbase is the payment rail (ADR-0002, ADR-0005); they are the regulated entity with existing KYC infrastructure; adding a parallel PersonalOS KYC layer adds friction without regulatory benefit at this stage.

**Progressive KYC prompt:** when a Soul's wallet balance reaches $500 — before the Coinbase threshold is hit — PersonalOS surfaces an in-app banner:

> *"To withdraw to your bank, you'll need to verify your identity once. This is a legal requirement handled by Coinbase — not something PersonalOS controls. It takes 2 minutes and unlocks unlimited withdrawals."*

Transparency about *why* and *who* defuses the trust risk. The Soul completes KYC in a calm, informed moment — not a frustrated withdrawal attempt.

**Non-negotiable pre-launch requirement:** independent legal opinion on whether PersonalOS is an MSB or payment facilitator under US FinCEN rules (and relevant state money transmission laws). "Coinbase handles KYC" is not a regulatory defence if PersonalOS is deemed the money transmitter.

**Secondary off-ramp provider** (Circle, Transak, or Stripe) identified and under contract before launch as a business continuity backstop.

## Risks Accepted and Mitigated

| Risk | Mitigation |
|---|---|
| Coinbase changes KYC thresholds without notice | Monitor Coinbase policy; surface threshold changes to Souls 30 days in advance |
| UX fragmentation — Soul handed to Coinbase's flow at trust-critical moment | Explicit framing: "Identity verification is handled by Coinbase — here's why and what they collect" |
| Coinbase account freeze contagion | Published Soul support escalation policy; wallet portability (ADR-0005) ensures fund access |
| Geographic coverage gaps (India, EU, Asia) | US-first MVP; secondary off-ramp providers address Phase 2 markets |
| Coinbase business continuity | Wallet portability (ADR-0005) solves fund access; secondary off-ramp solves fiat conversion |

## Alternatives Considered
- **Pre-emptive KYC at signup** — high Day 0 drop-off before Soul has seen any value. Not viable as a default.
- **Surprise KYC at withdrawal** — trust-breaking; disproportionate damage to a relationship the Soul considered established.
- **PersonalOS builds own KYC** — regulatory overhead not justified at MVP; adds compliance complexity before product-market-fit.

## Consequences
- MSB legal opinion is a **launch blocker** — must be resolved before first Soul earns Yield.
- Secondary off-ramp provider must be under contract before launch.
- $500 progressive KYC prompt threshold may need tuning based on average Soul wallet growth rate.
- Coinbase policy monitoring is an ongoing operational task, not a one-time setup.

---

# ADR-0014: Soul Retention Model

## Status
Accepted

## Context
PersonalOS works best running in the background — Harvests run automatically, Offers arrive via push, Yield accumulates. If Souls don't need to open the app, they'll forget it exists, miss high-value Offers, disconnect Konnections, and churn. Passive income without engagement creates passive disengagement.

## Decision
**Four-component retention model** giving Souls two distinct reasons to open the app — a financial hook and an awareness hook:

1. **Insight on Yields with causal explanation** — not just "$34.62 earned this week" but "$34.62 earned because your automotive Insight is strong and 3 Brands competed for your attention." Closes the loop between data, Insight, and money.

2. **Spend pattern alerts** — proactive awareness surfaced from Insight scores (never raw Transaktions): *"Your subscription spend jumped 22% this month. 3 dormant subscriptions totalling $98/mo detected — here's how to cancel."* Useful even when no Offer is attached.

3. **Seasonal Insight surges** — when Exchange bid trends for a category spike seasonally, surface it: *"Your travel Insight is your strongest this year heading into summer — premium Offers active now."* Connects market context to personal relevance.

4. **Conversational advisor** (see ADR-0015) — bidirectional engagement: Soul-initiated questions and Platform-initiated nudges that go beyond Yield to genuine financial and life intelligence.

## Alternatives Considered
- **Weekly Yield digest only** — single financial hook; no awareness dimension; feels like a bank statement.
- **Push Offers only** — purely transactional; builds no relationship between PersonalOS and the Soul's sense of self.
- **Depth milestone gamification** — progress mechanics without substance; motivates account connections but not sustained engagement.

## Consequences
- Yield attribution logic required: which Insight drove which Claim, traceable to the Soul-facing explanation.
- Spend pattern alerts must operate on Insight scores only — never on raw Transaktions — to maintain the E2E encryption promise (ADR-0003).
- Seasonal surge detection requires Exchange-level bid trend aggregation as a data product separate from individual Soul Insights.
- Retention components are sequenced: Yield explanation and spend alerts launch with MVP; seasonal surges and advisor follow in subsequent releases.

---

# ADR-0015: Conversational Advisor Architecture

## Status
Accepted

## Context
The conversational advisor is the highest-retention and highest-trust feature in the product. It positions PersonalOS as a financial intelligence companion rather than a passive Yield machine. However, the advisor must be genuinely useful without violating the E2E encryption model — it can never access raw Transaktions.

## Decision
**Hybrid architecture:**
- **Rules engine** for high-frequency, low-complexity nudges: spend alerts, Yield summaries, Konnection health warnings, seasonal Insight surge notifications.
- **LLM** for deeper advisory moments: quarterly financial reviews, life event signal interpretations, market context connections, and Soul-initiated open-ended questions.

**LLM context is strictly bounded to:**
- Soul's Insight scores (already on PersonalOS servers)
- Soul's Yield history (Claims, amounts, categories)
- Exchange bid trend data (aggregate market signals)
- External market data (interest rates, CPI, seasonal patterns)

**Never in the LLM prompt:** raw Transaktions, merchant names, individual dollar amounts, or any data that required decryption to compute. The privacy promise extends to the advisor: all UI framing uses *"based on your Insights (not your transactions)"*.

**Interaction model:** bidirectional — Soul-initiated conversation (*"Why did my travel Insight drop?"*) and Platform-initiated nudges (*"Your gym membership is active but unused — here are alternatives with Yield attached"*).

**Rollout:**
- **Introductory period** — available to all Souls. Validates Insight quality and advisor relevance at scale; generates feedback data before quality is proven.
- **Graduates to Platinum-exclusive** once quality thresholds are met and the introductory data validates the model.

## Alternatives Considered
- **Rules-based only** — scalable and consistent but shallow; feels like a push notification system, not an advisor.
- **LLM with raw Transaktion access** — richer context but violates the E2E encryption model (ADR-0003) and the platform's core promise.
- **Reactive only (Soul-initiated)** — lower engagement; misses the proactive value of Platform-initiated moments.

## Consequences
- LLM prompt engineering must enforce Insight-only context as a hard constraint, not a guideline.
- External data feeds required: interest rate APIs, CPI data, Exchange bid trend aggregation pipeline.
- Introductory period data is a quality validation exercise — advisor responses must be reviewed for Insight accuracy before Platinum gating.
- The advisor creates a new trust surface: incorrect or tone-deaf advice damages trust more than no advice. Quality bar is high before full rollout.
- Platinum gating of the advisor is one of three Platinum value drivers alongside premium Listings and higher Yield floors.

---

# ADR-0016: Brand Reporting and Data Minimisation

## Status
Accepted

## Context
Brands need sufficient measurement data to justify ROI and optimise Listings. But individual Soul data in Brand dashboards creates re-identification risk — even pseudonymous Soul IDs, tracked consistently across Listings, allow Brands to build Soul profiles over time, undermining the privacy model PersonalOS is built on.

## Decision
**Aggregate cohort reporting only.** Brand dashboards show:
- Total Claims on the Listing
- Total Budget spent
- Claim rate (Claims / Souls reached)
- Qualifying Insight cohort range (e.g. *"automotive.new_vehicle_purchase ≥ 0.80, CA, age 30–40"*)

No individual Soul data. No Soul IDs. No cross-Listing tracking of individual Souls. A Brand learns what kind of Soul responded — not which Soul.

**Opt-in server-side conversion pixel** for downstream measurement: Brand drops a server-side pixel on their conversion event (purchase made, form submitted). PersonalOS matches it anonymously against claimed Souls and returns only an **aggregate conversion rate** (*"23% of your PersonalOS Claims converted"*). Individual Soul identity is never returned. Minimum cohort size of 50 Claims required before conversion reporting activates — prevents de-anonymisation via small-cohort inference.

## Alternatives Considered
- **Soul-level reporting** — richest Brand data, catastrophic privacy risk. Contradicts the platform's core promise and ADR-0003.
- **Claim totals only (no cohort data)** — insufficient for Brands to optimise Listings or justify ROI. Higher churn among direct Brand advertisers.
- **No conversion measurement** — Brands cannot prove downstream ROI; they will not renew Budgets without it.

## Consequences
- Aggregate cohort reporting is a **selling point, not a limitation** — it is more actionable than most ad platforms offer while being demonstrably privacy-safe. Use in Brand pitch.
- Conversion pixel requires privacy-preserving match infrastructure: hashed Soul IDs, aggregate threshold before reporting fires.
- Minimum cohort size (50 Claims) must be enforced in reporting API before any cohort data is returned.
- The combination of high Claim rate + high conversion rate becomes PersonalOS's primary evidence base for the conversion advantage over Google/Meta.

---

# ADR-0017: Health Data and Preventive Care

## Status
Accepted

## Context
Health data unlocks a high-value category of Signal Types (wellness, fitness, preventive care, clinical trials) and enables the conversational advisor to act as a preventive health companion. Healthcare and wellness brands are among the highest-bidding categories on any intent-based platform. However, health data is the most sensitive category PersonalOS handles and faces complex, market-specific regulation.

## Decision
**Two-layer health model:**

**Layer 1 — Health Insights surface to Soul first, always.**
Cultivation detects health-relevant patterns and shows them to the Soul as self-awareness — not to Brands by default:
- `health.activity_declining` — movement patterns dropped significantly over 6 weeks
- `health.preventive_care_gap` — no annual check-up detected in 18 months
- `health.sleep_irregularity` — irregular sleep patterns vs. established baseline
- `health.nutrition_drift` — grocery and delivery spend shifting toward lower-nutrition categories
- `health.subscription_fitness_unused` — active gym membership with declining visit frequency

These Insights are shown in the advisor as awareness and actionable suggestions. They are **not Permitted to Brands by default** — Soul must explicitly opt in via a separate, higher-consent Permit tier.

**Layer 2 — Health Brands on the Exchange, gated by explicit Soul consent.**
A dedicated Health Permit tier with clearer consent language than financial Insights:
*"Health-related Insights are derived from your fitness and wellness data. Brands in preventive care, wellness, nutrition, fitness, and healthcare may reach you."*

Clinical trial matching is included — the highest-bid category on any health exchange ($20–$80 per Claim). Aggregate cohort model (ADR-0016) fully preserves Soul anonymity — pharma companies learn only that a qualifying cohort exists, never which Souls.

**Rollout by market:**

| Market | Approach | Status |
|---|---|---|
| **US — Apple HealthKit** | Activity, sleep, heart rate via HealthKit SDK. Avoids HIPAA — Apple is the covered entity. No clinical records. | **MVP** |
| **US — Epic FHIR (MyChart)** | Clinical data: prescriptions, diagnoses, specialist visits. Requires HIPAA Business Associate Agreement with Epic. | **Post-MVP** |
| **Europe** | Health data is GDPR Article 9 "special category" — requires explicit consent, stricter processing rules, Data Protection Impact Assessment. | **Deferred pending legal review** |
| **India / Asia** | No unified health data standard equivalent to Epic's FHIR API. Market-by-market assessment required. | **Deferred** |

## Alternatives Considered
- **MyChart/Epic FHIR at MVP** — richest clinical data, HIPAA compliance overhead not justified before product-market-fit.
- **Defer health entirely to Phase 2** — leaves the highest-bid Brand category untapped at launch.
- **Health data in same Permit tier as financial Insights** — insufficient consent granularity for sensitive data; regulatory and trust risk.

## Consequences
- Health Insights require a **separate, higher-consent Permit tier** with distinct UI language — cannot be bundled into the standard financial Permit flow.
- Clinical trial matching is the highest-priority new Signal Type for the health vertical — design before Epic FHIR integration begins.
- HIPAA Business Associate Agreement with Epic is a **prerequisite** for any clinical data integration.
- EU expansion requires a GDPR Article 9 Data Protection Impact Assessment before any health data processing in Europe.
- The advisor's preventive health capability is one of its most differentiating features — connects financial + health patterns to actionable steps (e.g. dormant gym membership → wellness Offer → preventive care appointment).

---

# ADR-0018: Go-to-Market and Soul Acquisition

## Status
Accepted

## Context
The Exchange requires critical mass on both sides before it delivers meaningful value — enough Souls with rich Insights to attract Brands, and enough Offer inventory to retain Souls. Standard App Store launch produces slow, unpredictable growth with thin early experiences that do not validate the model.

## Decision
**Three-phase acquisition strategy:**

### Phase 1 — Advisory Board (pre-launch)
Five advisors recruited across complementary profiles, each committing to **100 personal introductions by name** — not passive endorsement, not a logo on a website.

| Profile | Purpose | Network they unlock |
|---|---|---|
| Privacy / data rights advocate | Validates the sovereignty narrative | Privacy-conscious tech workers, journalists, academics |
| Fintech / open banking operator | Validates the Plaid + USDC + Base stack | Fintech founders and engineers already connecting financial accounts to apps |
| Consumer health / preventive care champion | Validates the health data narrative | Healthcare workers, wellness community, active Apple Health users |
| Brand / CMO side | Gives Brands confidence | Marketing leaders at mid-size Brands who are the right early Exchange customers |
| Web3 / decentralised identity | Validates the Arweave + Base + Coinbase Smart Wallet stack | Web3 early adopters who already hold crypto wallets |

**Advisory value exchange:** Platinum for life + 0.1–0.15% equity with 2-year vest + quarterly product access sessions. The Platinum-for-life is the most differentiated offer — advisors become actual Souls earning real Yield, making their advocacy authentic and informed.

### Phase 2 — Founding Cohort (1,000 Souls)
High-Depth, high-trust Souls sourced through advisor personal introductions. These are people predisposed to care about data sovereignty — they will connect multiple Konnections, achieve high Depth quickly, and generate the Insight density needed to validate the Exchange.

Founding Soul benefits: Platinum trial, elevated Yield floors, direct input into product roadmap, "Founding Soul" recognition.

### Phase 3 — Intermediary Co-Launch
The supply intermediary (ADR-0009) promotes PersonalOS to their existing user base: *"Earn more from your data with PersonalOS."* Delivers volume with an audience already comfortable connecting accounts and earning rewards. Transitions the platform from curated cohort to scale.

## Alternatives Considered
- **Open App Store beta** — slow, unpredictable, thin early experience; no feedback loop quality.
- **Paid user acquisition** — expensive before product-market-fit; wrong audience (needs intrinsic motivation, not incentive-driven sign-ups).
- **Waitlist only** — builds anticipation but delays the feedback loop that validates Cultivation quality.

## Consequences
- **Advisory board formation is the first go-to-market task** — precedes App Store submission, TestFlight, and Brand outreach.
- Founding cohort Depth data is the primary validation of Cultivation quality at real-world scale.
- Intermediary co-launch (Phase 3) is contingent on Phase 1/2 proving conversion metrics worth pitching.
- Each advisor's 100 personal introductions must be tracked — not assumed. Onboarding rate from advisor introductions is a leading indicator of the founding cohort strategy's success.

---

# ADR-0019: Competitive Moat

## Status
Accepted

## Context
Apple, Google, and well-funded startups have the distribution, capital, and engineering capacity to build a similar platform. A moat based solely on first-mover advantage is insufficient. The defensibility strategy must be structural and compounding.

## Decision
**Three-layer moat, in order of defensibility:**

### Layer 1 — The Compounding Ledger (primary moat)
Every Soul's Arweave Ledger grows richer over time. Historical Insight depth, seasonal patterns, life event signals, and multi-year spending trajectories compound into a personal data asset that cannot be replicated by a new entrant. A competitor starting fresh has no historical Insight depth on any Soul. Switching costs grow every month a Soul stays on PersonalOS. This is the most defensible layer because it is Soul-owned and irreversible — the data lives on Arweave forever, not on PersonalOS servers, but the Insights derived from it are tied to the PersonalOS Cultivation model.

### Layer 2 — Verifiable Trust Architecture (credibility moat)
Apple and Google *are* the data brokers PersonalOS replaces. A Soul who genuinely cares about data sovereignty will not hand Apple the same data they are trying to take back from Apple. This is not a marketing claim — it is an architectural fact enforced by E2E encryption, on-device Cultivation, and Arweave storage (ADR-0003, ADR-0004). Open-sourcing the Cultivation logic and publishing independent security audits creates a credibility gap that incumbents cannot close quickly without fundamentally changing their business models, which depend on data access.

### Layer 3 — Exchange Network Effect (flywheel moat)
Higher Soul count → higher Brand bids → higher Yield per Soul → more Souls attracted → more Brands join. Classic two-sided marketplace flywheel. Once spinning above critical mass, self-reinforcing. Requires speed — the flywheel must be turning before a well-funded competitor enters.

## Brand Conversion Advantage
The moat is reinforced by a structural conversion advantage that compounds as Exchange data accumulates:

| | Google / Meta | PersonalOS |
|---|---|---|
| Signal source | Browsing history (lagging) | Actual transactions + explicit intent declaration (leading) |
| Consumer attitude | Passive, often post-purchase | Active, willing — Permit-declared in-market intent |
| Wasted spend | ~98% reaches uninterested people | Near-zero — 100% of audience has opted in |
| Typical conversion | 1–2% | Structurally 10–40x higher |

This data, once proven in production through founding cohort and intermediary co-launch, becomes a self-reinforcing Brand acquisition argument.

## Honest Vulnerability
Apple launching "Apple Data Wallet" with native iOS integration would be existential. The defence is speed to critical mass and regulatory recognition before that happens. Regulatory tailwinds (GDPR, India DPDP Act, CCPA expansion) favour PersonalOS's pre-compliant architecture — incumbents face costly retrofitting while PersonalOS is already compliant by design.

## Alternatives Considered
- **IP / patents** — insufficient in software; too slow and expensive to enforce.
- **Exclusive data partnerships** — fragile; any exclusive can be replicated or bought out.
- **Geographic moat** — too narrow; does not create a global-scale defensible position.

## Consequences
- **Speed to 1,000 high-Depth founding Souls is the primary strategic priority** — the Ledger moat only compounds if Souls are on the platform.
- Open-sourcing Cultivation code and commissioning a security audit are moat investments, not just trust signals — treat them as such in the roadmap.
- Exchange bid trend data (aggregate, never individual) becomes a proprietary market intelligence asset over time — Brands on PersonalOS get market signal they cannot get elsewhere.
- The three-layer moat framing is the answer to *"can't Apple do this?"* in every investor and advisory board conversation.

---

# ADR-0020: IP and Patent Strategy

## Status
Accepted

## Context

PersonalOS has three core technical innovations that could be replicated by well-funded competitors (Caden, Brave, or an OpenAI/bank partnership):

1. **The Permit + Yield Floor mechanic** — a Soul-declared minimum price for a specific Signal Type, below which no Offer is delivered. This inverts the advertising model: Souls set the floor, Brands bid above it. No existing patent or prior art covers this precise combination.

2. **On-device Cultivation with Exchange matching** — Cultivation runs exclusively on the Soul's device (ADR-0003). Insight scores (not raw Transaktions) are submitted to the Exchange. This combination — local computation with remote matching — is architecturally distinct from server-side data monetization platforms.

3. **Crypto-shredding for GDPR compliance on permanent storage** — deleting the encryption key hash references renders Arweave-stored ciphertext permanently unreadable, satisfying GDPR Right to Erasure without requiring data deletion from the permanent chain (ADR-0004). This is a novel mechanism for reconciling immutable decentralised storage with privacy regulation.

The question is whether to pursue patents (expensive, slow, enforcement-intensive) or alternative IP strategies.

## Decision

**Three-track IP strategy:**

### Track 1 — Provisional Patent on Permit + Yield Floor (immediate)

File a provisional patent application on the Permit + Yield Floor mechanic within 30 days.

The claim: *A system and method for consumer-declared minimum compensation floors for signal-type-specific data access permissions in a real-time marketplace, wherein a consumer specifies a minimum yield amount for each category of derived insight, and marketplace matching is gated by brand bid exceeding the consumer-declared floor for that specific insight category.*

Cost: ~$3,000 (provisional, self-filed with patent attorney review). Locks a 12-month priority date. Full prosecution decision (est. $50K) deferred to Series A, when patent cost is absorbed by the funding round.

Why this claim specifically:
- Most commercially distinctive — directly tied to the revenue model
- No close prior art identified in consumer data marketplace patents
- Defensible scope: narrow enough to be grantable, broad enough to cover obvious variations
- Signals IP sophistication to investors

### Track 2 — Publish as Prior Art (immediate, parallel)

Open-source the Cultivation logic and publish a detailed technical whitepaper covering:
- On-device Cultivation + Exchange matching architecture
- Crypto-shredding GDPR mechanism
- Arweave Ledger structure with passkey-derived encryption

Published on Arweave itself (immutable, timestamped) + GitHub + arXiv preprint. This creates dated prior art that blocks any competitor from patenting these innovations, even if PersonalOS never files on them. Publishing the Cultivation logic also fulfils the ADR-0019 moat investment — verifiable trust requires open-source code.

### Track 3 — Trade Secret Protection (ongoing)

The Exchange matching algorithm, Insight scoring weights, and Depth calculation methodology are trade secrets — not patented (which requires public disclosure) and not open-sourced. Protected through:
- Employee NDAs and IP assignment agreements
- Limited access within the engineering team
- No publication of specific model weights or scoring thresholds

## Why Not Full Patent Prosecution Now

Full prosecution (18–24 months, ~$50K per patent, 3 patents = ~$150K) diverts capital and founder attention that is better spent on product and market validation at this stage. The provisional filing locks the priority date for 12 months — enough time to reach Series A, at which point $150K in patent costs is immaterial relative to round size.

Software patents are also notoriously difficult to enforce against well-funded defendants. The primary value of the Permit + Yield Floor patent at early stage is:
1. Blocking a direct copycat from claiming they invented it
2. Investor credibility — demonstrates IP awareness and strategic intent
3. Option value for licensing discussions at scale

## Alternatives Considered

- **Full prosecution immediately** — ~$150K and 18–24 months of attorney engagement. Not justified pre-Series A.
- **No IP strategy** — leaves PersonalOS vulnerable to a well-funded competitor filing on the same innovations and creating licensing friction.
- **Open-source everything** — maximises prior art protection and community trust but forfeits all licensing optionality on the Permit + Yield Floor mechanic.
- **Defensive publication only** — cheaper than a provisional but loses the priority date and any enforcement optionality.

## Consequences

- Provisional patent filing is a **30-day action item** — assign to patent attorney immediately.
- Cultivation open-source repository and technical whitepaper are **60-day deliverables** — coordinate with the security audit (ADR-0019) so the audit covers the published code.
- Exchange matching algorithm, Insight scoring weights, and Depth calculation remain **trade secrets** — engineering team NDAs and IP assignment agreements must be signed before any employee accesses these systems.
- Full patent prosecution decision is a **Series A agenda item** — investor counsel will have opinions on portfolio value.
- "Patent pending on our Permit + Yield Floor mechanic" is a legitimate investor narrative point once the provisional is filed.

---

