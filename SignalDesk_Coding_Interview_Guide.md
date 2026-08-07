**SignalDesk - Senior Engineer Interview Guide**

Interviewer Guide & Answer Key

_Solution Design + Coding · TypeScript / Node.js · 90 minutes_

# **Exercise Overview**

| **Round**         | **Time**    | **Task**                                                |
| ----------------- | ----------- | ------------------------------------------------------- |
| **Design Round**  | **~25 min** | Webhook classification system - design the architecture |
| **Coding Part 1** | **~15 min** | Bug fixes - find and fix planted bugs (aim: 2 or more)  |
| **Coding Part 2** | **~30 min** | Implement detectCorrelation (2 failing tests to pass)   |
| **Code Review**   | **~20 min** | Review summarizer.ts - identify what to change and why  |

Setup command for the candidate:

cd backend && npm install && npm test # shows 5 failing tests

**ℹ️** 5 failing tests: 3 from source bugs (one per bug) + 2 from the unimplemented correlator stub. The candidate is told there are 3 bugs; finding 2 in the time available is a strong result.

**Candidate context:  
This round tests architecture depth in an event-driven TypeScript/Node.js system, and calibrated judgment about when NOT to use AI - independent of whatever AI experience the candidate's resume does or doesn't claim. Probe both dimensions directly rather than relying on their stated background.**

# **Design Round - The Problem**

Present this design problem verbatim:

**_"We receive webhook events from 40+ SaaS integrations - Stripe, GitHub, PagerDuty, and others. Each event needs to be classified into one of five internal categories (billing, auth, usage, error, config) and routed to the appropriate downstream handler. Design the classification and routing system."_**

## **What to look for**

- Rules engine vs LLM classifier - this is the key fork. Does the candidate reach for an LLM immediately, or do they ask whether the mapping is stable and known?
- How multiple-match conflicts are resolved when an event type matches more than one prefix.
- Extensibility: how do new integrations get added without a redeployment? A config file, a database table, an admin UI?

**✅ Green flag:** Builds a deterministic lookup/rules table first, only considers LLM for novel or unmapped events.

**❌ Red flag:** Immediately proposes an LLM classifier for all events without questioning whether the mapping is known and stable.

## **Follow-up probe**

**_"Same system - when three correlated error events arrive within 60 seconds, we need to generate a one-paragraph incident summary for the on-call engineer. How does this part change your approach?"  
Expected shift: LLM IS appropriate here (synthesising structured data into natural language). Watch whether the candidate switches cleanly and articulates WHY the boundary falls between the two parts._**

## **The AI Tool-Selection Probe**

**During debrief - ask this AI tool-selection question:  
"Suppose you needed to coordinate a multi-step automated pipeline - several dependent agents or jobs running in sequence. Would you build the coordination logic as a deterministic control plane, or hand orchestration decisions to an LLM? Walk me through the trade-off."  
This directly tests whether the candidate's AI instinct is calibrated or just enthusiastic, independent of whatever AI experience their resume claims.  
Strong answer: names concrete trade-offs - cost (a deterministic state machine vs. tens of thousands of tokens per orchestration decision), determinism, debuggability, auditability.  
Weak answer: hand-waves, or defaults to 'an LLM is more flexible' without naming a real trade-off.**

# **Coding Exercise Answer Key**

Three bugs are planted in the backend source files. Finding and fixing two of the three within the session is a strong result - the third is intentionally harder to surface. Each bug has exactly one failing test.

| **Bug 1 - Route rules sort ascending - lowest priority wins**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/classifier.ts - classify()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| const sorted = \[...ROUTE_RULES\].sort((a, b) => a.priority - b.priority);                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| const sorted = \[...ROUTE_RULES\].sort((a, b) => b.priority - a.priority);                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Explanation:**<br><br>ROUTE_RULES are sorted before finding the first match. The intent is that higher-priority rules win when an event matches multiple prefixes - e.g., 'charge.failed' matches both the generic 'charge.' billing rule (priority 1) and the specific 'charge.failed' error rule (priority 3). With ascending sort, priority 1 appears first and wins. Changing to descending (b.priority - a.priority) makes priority 3 appear first and win - correct behaviour.<br><br>**✅ Green flag:** Reads the sort comparator immediately and spots the ascending vs descending issue. Can articulate the compounding effect: as more specific rules are added at higher priorities, the bug silently misfires more events.<br><br>**❌ Red flag:** Changes the test assertion to match the wrong behaviour. Or fixes the bug but cannot explain why the sort direction matters. |

| **Bug 2 - getAll exposes the internal events array by reference**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/store.ts - EventStore.getAll()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| getAll(): ClassifiedEvent\[\] {<br><br>return this.events;<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| getAll(): ClassifiedEvent\[\] {<br><br>return \[...this.events\];<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Explanation:**<br><br>Returning this.events directly exposes the store's internal array. Any caller that sorts, splices, or pushes onto the returned array silently corrupts the store's state - future getAll() calls return the mutated version. The fix is a shallow spread copy. This class of bug is subtle because the return type is correct (ClassifiedEvent\[\]) and the method name gives no indication of whether it returns a copy or a reference. TypeScript's type system does not help here.<br><br>**✅ Green flag:** Spots the missing spread immediately. Explains the encapsulation principle: a class should never expose a reference to its own private mutable state. May note that getRecent() has the same pattern and proactively fixes it too.<br><br>**❌ Red flag:** Passes the test by rewriting the test assertion to match the mutated state, or does not understand why returning this.events is a problem. |

| **Bug 3 - normalizeEventType discards the toLowerCase() return value**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/classifier.ts - normalizeEventType()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| function normalizeEventType(type: string): string {<br><br>type.toLowerCase();<br><br>return type;<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| function normalizeEventType(type: string): string {<br><br>return type.toLowerCase();<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Explanation:**<br><br>Strings in JavaScript are immutable. String methods like toLowerCase() do not modify the string in place - they return a new string. The bug calls type.toLowerCase() and discards the result, then returns the original mixed-case string. The function silently becomes a no-op. This only manifests when an event type arrives with mixed casing (e.g., 'Invoice.payment_succeeded') - standard all-lowercase types pass through correctly, which is why the other tests still pass. ESLint's no-unused-expressions rule would catch this; TypeScript's type checker will not.<br><br>**✅ Green flag:** Identifies this as the immutable string pattern without needing the failing test as a hint - finds it through code reading alone. Knows that ESLint can catch this class of mistake automatically and asks whether the project runs no-unused-expressions.<br><br>**❌ Red flag:** Cannot find this bug from code reading - only finds it after seeing the test output and binary-searching the file. Or confuses this with a mutation issue (strings are not mutable in JS/TS). |

# **Part 1.5 - Correlator Implementation (Interviewer Instructions)**

The correlator stub is left unimplemented. The interviewer verbally gives the candidate this brief. Do not share the code - describe it in words only.

<div class="joplin-table-wrapper"><table><tbody><tr><th><ul><li>The function receives a flat list of ClassifiedEvents that may span multiple categories and may arrive out of order.</li><li>Group events by category. Within each category, sort by timestamp ascending.</li><li>Slide a window across the sorted group: find the first position where the time between the first and last event in a window of CORRELATION_THRESHOLD events is within CORRELATION_WINDOW_MS.</li><li>Return an IncidentSignal for the first qualifying window found across any category, or null if none exists.</li><li>Passing the two failing correlator tests is the success criterion.</li></ul></th></tr></tbody></table></div>

**✅ Strong:** implements in a single clean pass - map + sort + sliding window loop. Writes a defensive spread copy before sorting. Returns null explicitly rather than falling through.

⏱ If the candidate is stuck after 10 minutes, the interviewer may share that step 1 is grouping by category and step 2 is sorting within each group. Do not give the sliding window logic.

## **Correlator Implementation Answer**

The complete correct implementation of detectCorrelation:

export function detectCorrelation(events: ClassifiedEvent\[\]): IncidentSignal | null {

// Step 1: group by category

const byCategory = new Map&lt;EventCategory, ClassifiedEvent\[\]&gt;();

for (const event of events) {

const group = byCategory.get(event.category) ?? \[\];

group.push(event);

byCategory.set(event.category, group);

}

// Step 2: for each category, check for a qualifying window

for (const \[category, categoryEvents\] of byCategory) {

const sorted = \[...categoryEvents\].sort(

(a, b) => new Date(a.timestamp).getTime() - new Date(b.timestamp).getTime(),

);

for (let i = 0; i <= sorted.length - CORRELATION_THRESHOLD; i++) {

const windowMs =

new Date(sorted\[i + CORRELATION_THRESHOLD - 1\].timestamp).getTime() -

new Date(sorted\[i\].timestamp).getTime();

if (windowMs <= CORRELATION_WINDOW_MS) {

return {

category,

events: sorted.slice(i, i + CORRELATION_THRESHOLD),

detectedAt: new Date().toISOString(),

message: \`\${CORRELATION_THRESHOLD} \${category} events detected within 60 seconds\`,

};

}

}

}

return null;

}

<div class="joplin-table-wrapper"><table><tbody><tr><th><p><strong>What to look for:</strong></p><ul><li>Groups by category before windowing - not just sorting all events together.</li><li>Spreads into a new array before sorting (avoids mutating the input).</li><li>Sliding window with index math rather than nested O(n²) timestamp comparisons.</li><li>Returns null explicitly at the end (not an implicit undefined).</li><li><strong>Strong: </strong>suggests Promise.all if this ever becomes async, or notes that Map iteration order is insertion order (implementation detail worth knowing).</li></ul></th></tr></tbody></table></div>

# **Code Review Answer - summarizer.ts**

Three functions in summarizer.ts involve LLM calls. The candidate should identify which are appropriate and which are anti-patterns.

| **Function**            | **LLM usage**                                     | **Verdict**      | **Fix**                                                     |
| ----------------------- | ------------------------------------------------- | ---------------- | ----------------------------------------------------------- |
| getEventSource          | Ask LLM to extract event.source from the struct   | **Anti-pattern** | return event.source                                         |
| isValidCategory         | Ask LLM if category is in a known list            | **Anti-pattern** | return VALID_CATEGORIES.includes(category as EventCategory) |
| generateIncidentSummary | Ask LLM to write narrative from structured signal | **Appropriate**  | Keep as-is                                                  |

**Why getEventSource and isValidCategory are anti-patterns:**

Both answers are deterministic, already known, and correct 100% of the time without an LLM. Using an LLM adds: ~200-500ms latency per call, API cost on every event processed, non-determinism (the LLM might return 'Stripe' vs 'stripe' vs 'the stripe integration'), and a new failure mode (API down = pipeline down).

generateIncidentSummary is the right use: there's no deterministic way to write good natural-language prose from structured data.

**✅ Strong:** Spots both anti-patterns without prompting, articulates cost + reliability + determinism. May propose a combined refactor that passes event.source directly into the summary call to remove getEventSource entirely.

**❌ Weak:** Focuses only on error handling, misses the architectural problem. Or says 'using a smaller model would fix it' rather than removing the LLM calls entirely.

# **Interviewer Probes**

Ask at least two of these depending on how the conversation flows. They distinguish someone who got to working code from someone who understands what they built.

## **1\. On the classifier (build-vs-buy judgment)**

**_"Suppose we needed to add percentage rollouts or A/B splits to this event routing system. How would you approach build-vs-buy - build a lightweight feature-flagging layer in-house, or adopt an existing platform?"  
Strong answer: Knows the complexity of a real experimentation platform, gives a concrete decision framework, not just 'use LaunchDarkly' as a one-line answer._**

## **2\. On the correlator (scale)**

**_"The current implementation is O(n × k) where n is events per category and k is categories. At 100K events/minute, where does this break and what's your next move?"_**

**Strong answer:** Identifies that grouping is fine, the inner sliding window is fine for reasonable n, but memory becomes the real constraint - proposes a ring buffer or time-bucketed store with TTL eviction.

## **3\. On AI judgment (unknown events)**

**_"If a new integration adds an event type we've never seen before, how does it get classified?"_**

**Strong answer:** Proposes an LLM-as-fallback pattern - deterministic rules first, LLM only for unmatched events, then human review to promote to a deterministic rule. This is the 'calibrated' answer.

## **4\. On AI tool selection (if not already covered in Design Round)**

**_"Suppose you needed to coordinate a multi-step automated pipeline - several dependent agents or jobs running in sequence. Would you build the coordination logic as a deterministic control plane, or hand orchestration decisions to an LLM? Walk me through the trade-off."  
Strong answer: Names concrete trade-offs - cost (a deterministic state machine vs. tens of thousands of tokens per orchestration decision), determinism, debuggability, auditability.  
Weak answer: Hand-waves, or defaults to 'an LLM is more flexible' without naming a real trade-off._**

# **Evaluation Scorecard**

Score each dimension 1-5 using observations from the exercise. Annotate with specific evidence.

| **Dimension**                     | **Score** | **Notes**                                                                                                                    |
| --------------------------------- | --------- | ---------------------------------------------------------------------------------------------------------------------------- |
| **Architectural reasoning**       | / 5       | ADR-quality thinking? Named the trade-offs clearly? Rules vs LLM fork handled correctly?                                     |
| **AI tool selection**             | / 5       | Drew the line correctly between deterministic and LLM? Calibrated instinct?                                                  |
| **Distributed systems depth**     | / 5       | Windowing, memory constraints, at-scale reasoning?                                                                           |
| **Code quality**                  | / 5       | Clean implementation, defensively written, follows existing patterns?                                                        |
| **Build-vs-buy judgment**         | / 5       | Knows when to reach for existing infrastructure vs building?                                                                 |
| **Communication / influence**     | / 5       | Explained their thinking clearly when probed?                                                                                |
| **Debugging methodology**         | / 5       | Systematic or random thrashing on the classifier bug?                                                                        |
| **Boundary / edge case thinking** | / 5       | Found the store reference bug unprompted? Identified the normalizeEventType no-op from code reading rather than test output? |
| **React / frontend**              | / 5       | N/A for this round - assess in separate UI round if advancing.                                                               |

| **Total Score (out of 40)** | **Recommendation**                                                           |
| --------------------------- | ---------------------------------------------------------------------------- |
| **34-40**                   | Strong - advance to system design round                                      |
| **27-33**                   | Advance with notes - one or two gaps to probe in system design               |
| **20-26**                   | Borderline - discuss with team before final round                            |
| **< 20**                    | Do not advance - fundamental gaps in architecture / AI judgment / TypeScript |

| **Field**                                    | **Notes**                                                                       |
| -------------------------------------------- | ------------------------------------------------------------------------------- |
| **Interviewer**                              |                                                                                 |
| **Date**                                     |                                                                                 |
| **Bugs found**                               | ☐ Bug 1 (sort direction) ☐ Bug 2 (store reference) ☐ Bug 3 (normalizeEventType) |
| **Stub completed?**                          | Yes / Partial / No                                                              |
| **AI anti-patterns spotted?**                | Both / One / None                                                               |
| **AI tool-selection probe - answer quality** | Calibrated / Enthusiastic / No answer                                           |
| **Strongest moment**                         |                                                                                 |
| **Biggest gap**                              |                                                                                 |
| **Score (out of 40)**                        |                                                                                 |
| **Recommendation**                           | Advance / Hold / Do Not Advance                                                 |
| **Summary**                                  |                                                                                 |

_- End of Interviewer Guide -_