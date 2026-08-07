**FreshRoute -- Senior Engineer Interview Guide**

System Design + AI Integration Assessment · V2

_Role: Senior Software / Solutions Engineer · Format: Open-Ended Problem → Architecture Discussion · Duration: 70 - 80 minutes_

# **Section 1: Interview Purpose & Philosophy**

This interview evaluates a candidate's ability to think like a senior systems engineer: asking the right clarifying questions, decomposing an ambiguous problem into a coherent architecture, and reasoning about trade-offs across reliability, scalability, security, and integration patterns.

V2 adds an AI integration layer. Candidates are expected to identify where AI genuinely improves the system and where deterministic solutions are more appropriate. The goal is not to see whether they use AI -- it is to see whether their instinct is calibrated.

The interview is intentionally open-ended. There is no single correct answer. The interviewer should resist filling silences, allow the candidate to drive, and probe where necessary to surface deeper thinking.

| **Dimension**                | **What to Observe**                                                                                                     |
| ---------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| **Requirements Elicitation** | Does the candidate ask smart clarifying questions before diving into a solution?                                        |
| **Systems Thinking**         | Can they decompose a complex domain into well-bounded components?                                                       |
| **Architecture Breadth**     | Do they naturally surface UI, API, async flows, data storage, batch, events, security?                                  |
| **Trade-off Reasoning**      | Do they articulate why they chose one approach over alternatives?                                                       |
| **AI Judgment**              | Do they identify where AI adds genuine value vs. where deterministic logic is correct? Do they know when NOT to use AI? |
| **Communication**            | Can they clearly explain a diagram and technical choices to a mixed audience?                                           |

# **Section 2: Problem Statement (Read to Candidate)**

**Interviewer note:** Read the scenario below verbatim or in your own words. Do not volunteer extra details -- let the candidate ask. Give them the printed problem card if doing an in-person session. In V2, add the final sentence about AI opportunities.

**The Scenario: FreshRoute**

HarvestLink Distribution is a regional fresh produce wholesale company operating across the US Pacific Coast. They source from 300+ farms and regional distributors and sell to restaurants, hotels, cafeterias, catering companies, and grocery chains -- roughly 1,500 food service buyers in total.

Today, every supplier sends a daily availability sheet via email (Excel or CSV) by 5 AM. A small team manually consolidates these into a master spreadsheet, then faxes or emails a PDF price list to buyers by 8 AM. Buyers call or email orders before a 2 PM cutoff. The business has doubled in three years and this process is completely overwhelmed -- sheets arrive late, orders are lost, and buyers are complaining about stale pricing.

They want to build a digital platform called FreshRoute that automates this entire workflow. Suppliers should be able to self-onboard and push their daily availability electronically. Buyers should be able to browse live inventory, place orders, and get notified when items they want come back in stock. Pricing must update automatically every morning based on USDA market data. The platform also needs to expose an API for large grocery chain buyers who want to integrate their own procurement systems.

Where you see opportunities to incorporate AI or machine learning into this platform, please call them out and reason about whether AI is the right tool for that specific component.

You have 45 minutes to design this system. Please think out loud. Start by asking any clarifying questions you need, then walk us through your architecture.

## **Key Problem Dimensions (Do Not Reveal Upfront)**

Track which capabilities the candidate surfaces organically vs. only with prompting.

| **Capability**                          | **Expected Architectural Component**                                        | **AI Opportunity?**                                                             |
| --------------------------------------- | --------------------------------------------------------------------------- | ------------------------------------------------------------------------------- |
| Buyer & Supplier self-registration      | Auth service, identity provider (Cognito / Auth0), email verification, RBAC | No -- auth must be deterministic and auditable                                  |
| Browse & search live inventory          | React SPA, REST API, search index (OpenSearch) with structured filters      | Partial -- NL search via embeddings is a strong AI add-on                       |
| Daily availability file ingestion       | S3 pre-signed upload + async pipeline, validation, dedup                    | Yes -- LLM normalization for non-standard / PDF supplier sheets                 |
| Order placement: spot vs. standing      | Sync REST for spot; async SQS for standing orders                           | No -- order confirmation must be deterministic                                  |
| Daily market repricing (USDA data)      | Scheduled batch job at ~4 AM; external API integration, fallback            | No -- pricing calculations must be deterministic and auditable                  |
| Inventory change event streaming        | Kinesis / SQS FIFO: AvailabilityAdded, PriceUpdated, OrderFilled events     | No -- event routing is deterministic                                            |
| Buyer 'watchlist' in-stock alerts       | Event-driven notification service triggered on matching availability        | Yes -- semantic matching (embeddings) finds 'leafy greens' -> 'romaine lettuce' |
| Grocery chain partner API               | API Gateway with OAuth2, rate limiting, versioning, webhook callbacks       | No -- API contract must be deterministic and versioned                          |
| Security & food safety compliance       | WAF, VPC, IAM, encryption, FSMA audit log                                   | No -- compliance logic must be deterministic and auditable                      |
| Observability                           | Structured logging, distributed tracing, CloudWatch Alarms                  | Partial -- AI anomaly detection for pricing outliers (\$0.00 bug scenario)      |
| **Supplier sheet normalisation (NEW)**  | LLM extraction pipeline for PDF/Excel/non-standard formats                  | Yes -- genuinely unstructured input, AI is the right tool                       |
| **Natural language buyer search (NEW)** | Embedding model + vector search layer alongside keyword search              | Yes -- NL queries cannot be handled by rules                                    |

# **Section 3: Interview Flow & Timing**

| **Phase**   | **Timing & Name**                                | **Interviewer Actions**                                                                                                                                                                                    |
| ----------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Phase 1** | **0 -- 5 min**<br><br>Intro & Setup              | Welcome candidate. Explain format: open-ended, they drive, whiteboard available. Emphasise thinking aloud. Read the scenario. Add: "Remind them they are welcome to call out AI opportunities throughout." |
| **Phase 2** | **5 -- 20 min**<br><br>Clarifying Questions      | Candidate asks questions. Do not volunteer answers unless asked. After ~15 minutes: "Feel free to start sketching your architecture based on the assumptions you've made."                                 |
| **Phase 3** | **20 -- 50 min**<br><br>Architecture Walkthrough | Candidate draws and explains design. Listen for component coverage and AI judgment. Probe where needed.                                                                                                    |
| **Phase 4** | **50 -- 70 min**<br><br>Deep Dives & Probing     | Use probe bank. Focus on gaps. Include at least two AI-specific probes (Section 5.7).                                                                                                                      |
| **Phase 5** | **70 -- 80 min**<br><br>Wrap-Up                  | "If you had another month, what would you tackle first -- and which part would you most want to revisit now that you've seen the whole system?"                                                            |

# **Section 4: Clarifying Questions -- What to Expect**

A strong candidate spends meaningful time in this phase. These are the questions a senior engineer should ask, the answers you can provide, and what the question signals.

## **4.1 Business & Scale Questions**

| **Expected Candidate Question**                                         | **Suggested Answer**                                                                                                                            | **What It Signals**                                                                 |
| ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| _How many suppliers upload availability sheets and how large are they?_ | ~300 suppliers, each uploading 1--300 item CSV files daily. Peak total: ~200k rows by 6 AM.                                                     | Immediately surfaces large file / high-volume ingestion challenge.                  |
| _How time-sensitive is the inventory data for buyers?_                  | Buyers expect inventory to be searchable within 10 minutes of a supplier upload completing.                                                     | Probes SLA, pipeline latency, and near-real-time processing requirements.           |
| _What's the order cutoff and are all orders the same?_                  | 2 PM cutoff for next-day delivery. Two types: spot orders (immediate confirmation) and standing orders (weekly recurring, processed overnight). | Distinguishes sync vs. async patterns, order lifecycle complexity.                  |
| _How does pricing work today and what triggers a change?_               | USDA publishes daily terminal market prices by 3 AM. Batch repricing job runs at 4 AM to update all items before buyers log in.                 | Recognises need for external API integration + scheduled batch + price propagation. |
| _What should happen if a buyer's preferred item is out of stock?_       | Buyers can set a watchlist. When matching inventory arrives from any supplier, they get an email or SMS alert.                                  | Triggers event-driven notification design thinking.                                 |
| _Do large grocery chain buyers have different needs?_                   | Yes -- 3--4 major chains want to pull live inventory and push orders via API, not via the UI.                                                   | Surfaces partner API, authentication, rate limiting, and webhook design.            |
| _Any regulatory or compliance requirements?_                            | FSMA requires full traceability: every lot number, harvest date, origin farm, and handling chain must be logged.                                | Compliance and audit log awareness.                                                 |
| _What are availability and latency requirements?_                       | 99.5% uptime during 4 AM--2 PM peak. After-hours degraded mode acceptable. API p95 < 500ms for search.                                          | SLO definition and failure mode planning.                                           |

## **4.2 AI-Specific Clarifying Questions**

| **Expected Candidate Question**                                                   | **Suggested Answer**                                                                                                 | **What It Signals**                                                                                                         |
| --------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------- |
| _Are all supplier files in a standard CSV format?_                                | No -- about 20% arrive as Excel files with merged cells or pivot tables. A handful of smaller farms still send PDFs. | Opens the door to AI-assisted parsing; candidate who asks this will think about the non-standard input problem.             |
| _How do buyers describe items they want on their watchlist?_                      | Free text -- they type 'organic strawberries' or 'leafy greens, Grade 1'. No structured SKU required.                | Reveals the semantic mismatch between buyer language and supplier naming conventions -- motivates embedding-based matching. |
| _How quickly do buyers expect watchlist alerts after matching inventory arrives?_ | Within 5 minutes of a supplier upload completing.                                                                    | Surfaces the latency constraint for the semantic matching pipeline.                                                         |

**Green flag:** A candidate who asks about supplier format variability AND reasons about the semantic gap between buyer watchlist language and supplier naming conventions demonstrates real AI product thinking -- not just AI enthusiasm.

**Red flag:** A candidate who proposes using an LLM for the USDA repricing calculation or for lot number format validation without prompting -- both of which are deterministic, auditable problems -- shows miscalibrated AI instinct.

# **Section 5: Probe Bank -- Deep-Dive Questions**

Use these probes during Phase 4, or during Phase 3 if the candidate skips an area. Pick 3--4 based on background and observed gaps. Always include at least one from Section 5.7.

## **5.1 File Processing & Peak Ingestion**

**_"All 300 suppliers upload by 6 AM. Walk me through what happens architecturally from the moment a file lands in the system to when a buyer can search for that inventory."_**

**Strong answer:** Pre-signed S3 URL upload -> S3 event triggers Step Functions -> schema validation -> chunk into batches -> SQS -> parallel ECS workers -> DynamoDB + OpenSearch. Mentions dedup check, error handling, and supplier webhook receipt.

**_"A supplier uploads a 50,000-row CSV. What's your chunking strategy and why?"_**

**Strong answer:** Chunk into 1,000--5,000 row batches, push each to SQS, process in parallel ECS tasks. Explains why: avoids Lambda timeout limits, enables retryability at chunk level, limits memory per worker.

**_"A supplier re-uploads the same file twice -- once at 5:02 AM and once at 5:47 AM with minor corrections. How do you handle deduplication?"_**

**Strong answer:** Content hash per row (supplier ID + item + lot + date). Second upload computes hash; rows already seen are skipped; new/changed rows are upserted. Supplier gets a report: X rows ingested, Y skipped (duplicate), Z updated.

**_"The USDA API is down at 4 AM. What happens to buyers who log in at 7 AM?"_**

**Strong answer:** Previous day's prices remain in effect (stale-but-visible, not 500). Batch job retries every 15 minutes until 7 AM; after that, manual override or hold. Buyers see a banner: 'Prices last updated \[timestamp\]'. PriceUpdated events not emitted until job succeeds.

## **5.2 Batch Repricing Job**

**_"Walk me through the 4 AM repricing batch job -- what does it do, in what order, and where could it fail?"_**

**Strong answer:** Fetch USDA terminal market prices -> join with catalogue items by commodity code -> apply margin rules (configurable per category) -> write new prices to DynamoDB -> emit PriceUpdated events to Kinesis -> invalidate OpenSearch price fields. Failure modes: USDA API down (fallback to prior day + alert), DB write failure (idempotent retry with job ID), partial completion (checkpoint: resume from last committed batch).

**_"After the repricing job runs, one SKU shows a price of \$0.00. How does your system detect and handle this before buyers see it?"_**

**Strong answer:** Post-job validation step: check for prices &lt;= \$0.00 or &gt; 3x the 7-day average for that commodity (statistical, not AI). Anomalies are flagged: price write is held, alert goes to ops team, prior price stays live. Strong candidates use a z-score or simple threshold; they do NOT propose an LLM to detect the anomaly.

**_"How would you make the margin rules configurable without a redeployment?"_**

**Strong answer:** Margin rules stored in DynamoDB or SSM Parameter Store, not hard-coded. Admin UI (or API) allows ops team to update rules; batch job reads config at start of each run. Versioned config so rules can be rolled back if a bad margin causes pricing errors.

**_"A buyer placed a standing order last week at \$1.20/lb. The repricing job drops the price to \$0.95/lb overnight. Does the standing order honour the new price or the price at time of order?"_**

**Strong answer:** This is a business rule question, not a technical one -- the candidate should identify it as such and ask. Either answer can be correct: lock price at order time (simpler, predictable for buyer) or apply current price at fulfillment (reflects market, better for buyer when prices drop). Most food service businesses apply current price at fulfillment. Either way, the architecture must store the price at order creation for audit.

## **5.3 Event Streaming & Watchlist Notifications**

**_"A buyer's watchlist says 'organic strawberries'. A supplier uploads 'Driscoll's Organic Strawberries 8oz Grade 1'. How does your notification pipeline decide this is a match?"_**

**Strong answer:** Keyword matching alone fails here. Strong candidates propose embedding-based semantic similarity: both strings are embedded, cosine similarity >= threshold triggers the alert. They also mention a confidence threshold below which the match is held for human review rather than auto-notified.

**_"What happens to in-flight watchlist alerts if the notification service goes down for 20 minutes during the 5--7 AM upload window?"_**

**Strong answer:** Events are durably held in Kinesis or SQS. The notification service resumes reading from its checkpoint when it comes back up. No alerts are lost; some arrive ~20 minutes later than expected. DLQ for events that fail after max retries.

**_"A buyer has 50 watchlist items. 200 AvailabilityAdded events fire in 30 seconds. How do you avoid overwhelming the buyer with 40 notifications in two minutes?"_**

**Strong answer:** Digest window: collect all matching events over a 5--10 minute window, then send a single 'Your watchlist has X new matches' email/push with a link to the filtered view. Per-buyer rate limiter prevents burst. Strong candidates also consider buyer preference settings (instant vs. daily digest).

**_"The event stream carries AvailabilityAdded, PriceUpdated, and OrderFilled events. Which consumers care about which events and how do you route them?"_**

**Strong answer:** Fan-out from Kinesis to multiple consumer groups: watchlist service (AvailabilityAdded only), search index updater (AvailabilityAdded + PriceUpdated), grocery chain webhook relay (OrderFilled + PriceUpdated), FSMA audit log writer (all three). Filter by event type at the consumer, not the producer.

## **5.4 API Design for Grocery Chain Partners**

**_"A grocery chain's procurement system pulls live inventory every 5 minutes. There are 4 chains each running this poll. How do you protect the platform from this traffic pattern?"_**

**Strong answer:** Rate limiting per API key at API Gateway. Aggressive cache on the GET /inventory response (1--2 minute TTL at CloudFront or ElastiCache) so most polls return cached data. Better design: push model -- chains subscribe to webhooks and receive PriceUpdated / AvailabilityAdded events in near real-time, eliminating the poll entirely.

**_"You need to introduce a breaking change to the inventory API -- a field rename that the grocery chain's system depends on. How do you handle versioning?"_**

**Strong answer:** URL versioning (/v1/, /v2/) or header-based. Keep v1 alive for a deprecation window (e.g. 90 days), notify partners with a migration guide. v2 serves the new schema; v1 translates from v2 internally. API Gateway routes by path version. Strong candidates also mention semantic versioning and partner SLA commitments.

**_"A grocery chain's system places an order via API. The order succeeds but their webhook delivery fails -- they never get the confirmation. What does your system do?"_**

**Strong answer:** Webhook retry with exponential backoff (3 attempts, then DLQ). Partner can also poll GET /orders/{id} for status. Order state is the source of truth in DynamoDB -- the order is not cancelled because webhook delivery failed. Ops alert when a webhook consistently fails so account team can contact the chain.

## **5.5 Data Storage & Traceability**

**_"FSMA requires lot-level traceability from farm to delivery. Walk me through how your data model supports a recall scenario -- 'find every restaurant that received lot 2024-HRV-001 in the last 30 days'."_**

**Strong answer:** Lot number stored on every inventory item and copied to every order line. Recall query: scan OrderLines table filtered by lot_number + created_at range -> join to Orders -> join to Buyers -> list of affected restaurants. DynamoDB GSI on lot_number enables fast lookup. Raw supplier upload stored in S3 for audit.

**_"Inventory data changes constantly -- price updates, quantity decrements as orders arrive. How do you keep OpenSearch in sync with DynamoDB without double-writes causing inconsistency?"_**

**Strong answer:** DynamoDB is the source of truth. OpenSearch is updated via DynamoDB Streams -> Lambda -> OpenSearch (event-driven, not dual-write). If OpenSearch falls behind, buyers may see stale search results briefly (acceptable) but orders are placed against DynamoDB (authoritative). Periodic reconciliation job detects and resyncs drift.

**_"How long do you retain audit log data and where does it live?"_**

**Strong answer:** FSMA requires 2 years for most records. Hot storage (DynamoDB) for 90 days for fast recall queries; cold storage (S3 + Glacier) for 2+ years. Immutable log entries: append-only DynamoDB table with no delete permissions, or write directly to S3 with object lock. Strong candidates also mention CloudTrail for API-level audit.

## **5.6 Security**

**_"A rogue supplier uploads a file with prices for items they don't own -- attempting to inject fake inventory for a competitor. How does your system prevent this?"_**

**Strong answer:** Pre-signed S3 upload URL is scoped to the authenticated supplier's prefix. Pipeline validates that every item in the upload is within the supplier's approved catalogue (supplier_id on each item must match the JWT claim). Cross-supplier item writes are rejected at validation step. Suppliers can only see and modify their own listings.

**_"A grocery chain API key is compromised. What happens?"_**

**Strong answer:** Key is rotated immediately in API Gateway (new key issued, old key invalidated). All requests made with the old key are logged -- CloudWatch Logs / S3 -- so the breach scope can be assessed. Orders placed during the breach window are reviewed for fraudulent patterns. Strong candidates also mention API key scoping (read-only vs. order-write), rate limiting anomaly alerts, and mutual TLS for high-value partners.

**_"How do you ensure a buyer cannot see another buyer's order history or pricing tiers?"_**

**Strong answer:** All read APIs filter by buyer_id extracted from the JWT -- never from a request parameter the client controls. DynamoDB access patterns use buyer_id as partition key or GSI filter. OpenSearch queries always include a buyer_id term filter. Automated tests assert that buyer A cannot retrieve buyer B's data by manipulating IDs.

## **5.7 AI & Machine Learning Judgment (New in V2)**

Always include at least two of these probes. They are the core AI-calibration signal for V2.

**_Probe 1: "About 20% of your supplier uploads are non-standard -- Excel files with merged cells, PDF price lists, even the occasional scanned fax. Standard CSV parsing breaks on these. What's your approach?"_**

**Strong answer:** Proposes a branch in the pipeline -- standard CSVs take the deterministic validation path; non-standard formats route to an LLM extraction step that normalises the data into a structured schema. Critically: validates LLM output against the same schema validator used for standard CSVs before writing to inventory (AI extracts, deterministic logic validates). Does not replace the standard pipeline with LLM everywhere.

**_Probe 2: "A buyer types 'need 50 cases of leafy greens ideally organic under \$3/unit' into the search bar. How does your architecture handle this query?"_**

**Strong answer:** Recognises this is a natural language query that cannot be handled by keyword search alone. Proposes embedding the query using a sentence embedding model, then performing a vector similarity search against pre-computed item embeddings alongside a structured filter for price and quantity. Importantly: structured filters (price &lt; \$3, quantity &gt;= 50 cases) remain deterministic -- only the semantic matching uses embeddings.

**_Probe 3: "Supplier A uploads 'Certified Organic Driscoll Strawberries Grade 1 8oz.' A buyer's watchlist entry reads 'organic strawberries'. Is this a match? How does your system decide -- and what happens if it gets it wrong?"_**

**Strong answer:** Semantic embedding similarity is the right tool for fuzzy name matching -- the embedding of both strings should be close. But: the candidate must have a confidence threshold below which the match is held for human review rather than auto-notified. Wrong watchlist alerts erode buyer trust quickly. Strong candidates also note that a feedback loop (buyer confirms or dismisses the alert) improves the matching model over time.

**_Probe 4: "The USDA repricing job -- would you consider using an LLM to help set or recommend prices? Why or why not?"_**

**Strong answer (correct answer is NO):** Pricing must be deterministic, auditable, and reproducible. An LLM cannot explain why it recommended \$2.47/lb for Fuji apples on a given day, and FSMA-adjacent compliance contexts require a full audit trail. The correct answer is algorithmic: fetch USDA market price, apply configured margin rules, emit PriceUpdated events. A price anomaly detector (statistical, not AI) validates the output. This is a deliberate 'when NOT to use AI' probe -- the correct answer is to not use it.

**_Probe 5: "The LLM that normalises PDF supplier sheets occasionally extracts a lot number incorrectly -- '2024-HRV-001' becomes '2024-HRV-01'. This lot number ends up in the FSMA audit log. What are the implications and how do you prevent it?"_**

**Strong answer:** FSMA traceability errors are serious -- a wrong lot number breaks the chain of custody. Prevention: the LLM extraction output must be validated against a lot number regex before being written anywhere (LLM extracts, deterministic regex validates). For audit log entries, the raw source document must also be retained (S3) so any discrepancy can be investigated. This is not a place where you trust the LLM output unconditionally.

# **Section 6: Reference Architecture (Interviewer Guide Only)**

**Do not share this with the candidate.** Use it to calibrate depth and coverage -- not as the 'expected' answer.

## **6.1 Component Overview**

**Presentation Layer**

- React SPA (responsive, progressive web app) served via CloudFront CDN
- Dual portals: Buyer portal (browse, order, watchlist) and Supplier portal (upload, track, manage listings)
- WebSocket or SSE connection for real-time inventory count updates during peak hours
- NEW: Natural language search input in the buyer portal -- embedding model runs server-side, not client-side

**API & Gateway Layer**

- AWS API Gateway: auth validation, rate limiting (buyer tiers), logging, routing
- Sync REST: GET inventory, POST order (<\$5k spot order), user profile, watchlist management
- Async pattern: POST /availability-upload returns 202 + job ID; POST /standing-order queues via SQS
- WebSocket API on API Gateway for real-time inventory updates

**Core Services**

- Auth Service: Cognito or Auth0, JWT tokens, RBAC (buyer / supplier / admin / partner roles)
- Inventory Service: reads/writes DynamoDB, syncs to OpenSearch, handles dedup
- Order Service: spot orders sync (DynamoDB, idempotent), standing orders async (SQS queue, overnight processor)
- Notification Service: consumes Kinesis events, evaluates watchlist matches, sends email/SMS via SES/SNS
- Repricing Service: scheduled batch (EventBridge cron), fetches USDA data, applies margin rules, emits PriceUpdated events

**File Processing Pipeline -- Extended for AI**

**Step 1 -- Supplier uploads via pre-signed S3 URL (bypasses API Gateway for large files)**

Step 2 -- S3 ObjectCreated triggers Step Functions workflow

Step 2a -- Format detection: is this a standard CSV, Excel, or unstructured (PDF/image)?

Step 3a (standard path) -- Schema validation + lot number format check (deterministic, fast)

**Step 3b (non-standard path) --** LLM extraction: normalise unstructured sheet into structured schema. Prompt includes the target schema and valid field formats. Output validated against the same schema validator as Step 3a before proceeding.

Step 4 -- Deduplication check against existing catalogue (same supplier + item + lot)

Step 5 -- Chunk into batches of 2,000 rows, push to SQS for parallel ECS workers

Step 6 -- Workers write to DynamoDB + sync to OpenSearch index + compute and store item embeddings

Step 7 -- Publish AvailabilityAdded events to Kinesis stream

Step 8 -- Supplier receives webhook + in-portal report: rows ingested / rejected / warnings / LLM-extracted rows flagged for review

**AI scope boundary:** LLM is used ONLY for non-standard format extraction (step 3b). Standard CSVs never touch the LLM. All LLM output is validated against a deterministic schema before being written. Lot numbers extracted by LLM are cross-referenced against the lot number regex -- mismatch = flagged for human review, not auto-written.

**Semantic Search & Watchlist Matching -- New**

- At ingestion time: each inventory item description is embedded (sentence embedding model, e.g. text-embedding-3-small) and stored alongside the item in OpenSearch's kNN index
- Buyer NL search: query is embedded server-side; OpenSearch kNN search returns semantically similar items; structured filters (price, quantity, grade) applied as post-filters
- Keyword search remains the default for structured queries ('Fuji Apple 88ct') -- embedding search is invoked when the query cannot be matched by keyword alone
- Watchlist matching: buyer watchlist items are embedded at creation time; when AvailabilityAdded event fires, new item embedding is compared against all watchlist embeddings; matches above a confidence threshold trigger notification
- Confidence threshold: matches below 0.75 cosine similarity are held for human review (not auto-notified); feedback loop updates the threshold over time

**Batch Repricing Job**

- EventBridge cron triggers at 4 AM daily
- Lambda or ECS task fetches USDA Terminal Market Report via HTTP (retry with backoff)
- Join USDA commodity prices to FreshRoute catalogue by commodity code
- Apply per-category margin rules (read from SSM Parameter Store / DynamoDB config table)
- Write new prices to DynamoDB (idempotent upsert, job_id prevents double-write)
- Emit PriceUpdated events to Kinesis; OpenSearch price fields updated via stream consumer
- Post-job anomaly check: statistical z-score against 7-day trailing average; outliers held for ops review

**No LLM in repricing.** Pricing logic is deterministic: USDA price x margin rule = output. A statistical anomaly detector (z-score against trailing 7-day average) flags outliers like the \$0.00 scenario. Statistical, not AI -- the output is explainable and auditable.

**Event Streaming**

- Kinesis Data Streams (or SQS FIFO for simpler deployments)
- Events: AvailabilityAdded {supplier_id, item_id, lot_number, quantity, price}, PriceUpdated {item_id, old_price, new_price, effective_at}, OrderFilled {order_id, buyer_id, lines\[\]}
- Consumers: Notification Service, OpenSearch sync, FSMA audit log writer, partner webhook relay
- Dead letter queues on all consumers; CloudWatch alarms on DLQ depth

**Data Storage**

- DynamoDB: Inventory items (PK: supplier_id, SK: item_id), Orders (PK: order_id), Buyers and Suppliers (PK: user_id). GSIs on lot_number, commodity_code, buyer_id
- OpenSearch: full-text + structured inventory search. Fields: item_name, category, grade, price, quantity, supplier_id, lot_number, available_date
- S3: raw supplier upload files (retained 2+ years for FSMA audit), FSMA audit log exports, static assets
- ElastiCache (Redis): session tokens, API response cache for GET /inventory (2-minute TTL)
- Vector store: item and watchlist embeddings stored in OpenSearch kNN index (or pgvector if PostgreSQL is already in use -- avoid adding a dedicated vector DB for this scale)

**Security & Compliance**

- VPC: all services in private subnets; API Gateway and CloudFront as the only public entry points
- WAF: rate limiting, SQL injection protection, geo-blocking for non-US traffic if required
- IAM: least-privilege roles per service; no wildcard \* policies
- Encryption: S3 server-side encryption (AES-256), DynamoDB encryption at rest, TLS in transit
- FSMA audit log: append-only DynamoDB table (no delete IAM permissions) + S3 object lock for immutability
- Supplier upload: pre-signed S3 URLs scoped to supplier's own prefix; max file size enforced

**Observability**

- Structured JSON logging (correlation ID: request_id + job_id + supplier_id)
- Distributed tracing: AWS X-Ray across API Gateway, Lambda, ECS, DynamoDB
- CloudWatch Alarms: pipeline SLA (availability searchable within 10 min of upload), repricing job duration, DLQ depth, API p95 latency
- Dashboards: supplier upload success rate, watchlist match rate, order volume by hour, repricing job status
- AI pipeline metrics: LLM extraction success/failure rate, schema validation pass rate for LLM-extracted rows, watchlist match confidence distribution, false positive alert rate (buyer dismissed the notification)

## **6.2 Where AI Does NOT Belong in This System**

| **Component**                    | **Why NOT AI**                                                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------ |
| **USDA repricing calculation**   | Must be deterministic and auditable. LLM outputs are not reproducible. A \$0.01 pricing error at scale = significant revenue impact. |
| **Lot number format validation** | Regex is correct, fast, and 100% accurate for a known format. LLM adds latency, cost, and occasional errors.                         |
| **Auth & authorization**         | Security-critical. Must be deterministic. An LLM cannot reliably enforce RBAC.                                                       |
| **Order confirmation logic**     | Idempotency and consistency are required. LLM is inappropriate for transactional state machines.                                     |
| **FSMA audit log entries**       | Must be exact, tamper-evident, and machine-generated from source data. No LLM interpretation.                                        |
| **Spot order routing**           | Simple rules: spot &lt; \$5k sync, spot &gt;= \$5k async. No ambiguity requiring AI.                                                 |

# **Section 7: Evaluation Rubric**

Score each dimension 1--5. Total score of 40+ (out of 55) is a strong hire signal for a senior role.

| **Criteria**                        | **Needs Work (1-2)**                                                                   | **Acceptable (3)**                                                                           | **Strong (4-5)**                                                                                                                                                                                                                          |
| ----------------------------------- | -------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Requirements Elicitation**        | Dives into solution without asking questions. Assumes all constraints.                 | Asks 3--4 business questions. Mostly uncovers scale and basic functional needs.              | Asks 6+ targeted questions covering scale, SLOs, order types, compliance, and format variability. Probes AI-related constraints without being asked.                                                                                      |
| **Clarifying Question Quality**     | Asks surface-level or irrelevant questions ('What colour should the UI be?').          | Questions uncover enough to proceed but miss key constraints (e.g. FSMA, standing orders).   | Questions reveal deep domain instinct. Independently surfaces supplier format variability and semantic watchlist matching.                                                                                                                |
| **Architecture Breadth**            | Covers only 3--4 components. Misses async processing, events, or security.             | Covers most components but needs prompting for 2--3 areas (e.g. partner API, FSMA audit).    | Organically covers all 10+ architectural dimensions including AI components, without significant prompting.                                                                                                                               |
| **Component Design Depth**          | High-level only. Cannot explain how any component works internally.                    | Can explain 2--3 components in detail. Struggles with pipeline internals or event routing.   | Designs each component with specific technologies, failure modes, and data flows. Names trade-offs for every major choice.                                                                                                                |
| **Trade-off Reasoning**             | Presents one option per decision. No alternatives considered.                          | Mentions alternatives for 2--3 decisions but without structured comparison.                  | For every major decision (sync vs async, Kinesis vs SQS, SQL vs NoSQL), articulates trade-offs and defends their choice.                                                                                                                  |
| **Failure Mode & Resilience**       | Assumes the happy path throughout. No discussion of failures.                          | Mentions retry logic or DLQ but does not reason through specific failure scenarios.          | Proactively identifies failure modes for USDA outage, pipeline delays, notification service downtime. Proposes concrete mitigations.                                                                                                      |
| **Security & Compliance Awareness** | Mentions 'add auth' but no specifics. No FSMA discussion.                              | Adds auth and some encryption. FSMA raised only when prompted.                               | Independently raises FSMA lot-level traceability, supplier upload scoping, RBAC, and audit log immutability without prompting.                                                                                                            |
| **Data Modelling**                  | Proposes a single relational table or has no clear data model.                         | Reasonable model but misses GSIs for recall queries or does not separate hot/cold storage.   | Clear DynamoDB schema with GSIs for lot recall. Explains why OpenSearch supplements DynamoDB. Addresses retention tiers.                                                                                                                  |
| **API & Integration Design**        | Single REST API for everything. No discussion of versioning or partner integration.    | Distinguishes sync vs. async APIs. Partner API mentioned but without versioning or webhooks. | Full partner API design: OAuth2 scoping, rate limiting, versioning strategy, webhook delivery with retry and DLQ.                                                                                                                         |
| **Communication Clarity**           | Explanation is hard to follow. Cannot clearly map diagram to requirements.             | Adequate explanation. Some components need re-explanation when probed.                       | Explains architecture clearly to both technical and non-technical audiences. Handles follow-up questions without hesitation.                                                                                                              |
| **AI & ML Judgment (New)**          | Proposes AI for everything, including repricing and auth. No discussion of trade-offs. | Identifies 1--2 genuine AI opportunities; misses the 'when not to' probes.                   | Precisely identifies where AI adds value (non-standard ingestion, semantic search, watchlist matching) and explicitly rejects it for deterministic components (repricing, auth, FSMA log). Discusses validation strategy for LLM outputs. |

| **Total Score** | **Recommendation**                                             |
| --------------- | -------------------------------------------------------------- |
| **49 -- 55**    | Strong Hire -- push to staff engineer discussion               |
| **40 -- 48**    | Hire -- solid senior engineer; 1--2 areas to grow              |
| **29 -- 39**    | Lean No -- mid-level capability; gaps in breadth or depth      |
| **< 29**        | No Hire -- significant gaps in systems thinking or AI judgment |

# **Section 8: Interviewer Notes Template**

| **Field**                                   | **Notes**                                                                                                            |
| ------------------------------------------- | -------------------------------------------------------------------------------------------------------------------- |
| **Candidate Name**                          |                                                                                                                      |
| **Date**                                    |                                                                                                                      |
| **Interviewer**                             |                                                                                                                      |
| **Clarifying Questions Asked**              | List the questions the candidate asked organically. Note any important gaps.                                         |
| **Components Surfaced Organically**         | List components the candidate raised without prompting.                                                              |
| **Components Requiring Prompting**          | List components only surfaced after an interviewer probe.                                                            |
| **AI Opportunities Identified Organically** | e.g. 'Raised LLM for non-standard supplier sheets and semantic watchlist matching without prompting'                 |
| **AI Misapplications Caught / Not Caught**  | e.g. 'Proposed LLM for repricing -- probed and corrected' or 'Caught unprompted: said pricing must be deterministic' |
| **Strongest Area**                          |                                                                                                                      |
| **Notable Gap**                             |                                                                                                                      |
| **Communication Quality**                   | Clear / Adequate / Unclear                                                                                           |
| **Score (out of 55)**                       | /55                                                                                                                  |
| **Recommendation**                          | Hire / Lean Hire / Lean No / No Hire                                                                                 |
| **Summary**                                 | 2--3 sentence narrative for the debrief packet.                                                                      |

_\-- End of Interviewer Guide · FreshRoute V2 · Confidential --_