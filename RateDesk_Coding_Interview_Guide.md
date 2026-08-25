**RateDesk Coding Exercise**

Interviewer Guide & Answer Key

_Round 2 · Hands-on Coding · TypeScript / Node.js / React · 90 minutes_

# **Exercise Overview**

<div class="joplin-table-wrapper"><table><tbody><tr><th><p><strong>Candidate receives:</strong></p><ul><li>A GitHub repo (or zip) containing backend/ and frontend/ directories</li><li>A README.md with setup instructions and the 3-part task description</li><li>No solution files, no hints beyond the TODO comments in the code</li></ul><p><strong>Candidate does NOT receive:</strong></p><ul><li>This document</li><li>The bug locations or fix details</li><li>The reference implementation of PriceCalculator.tsx or discountHandlers.ts</li></ul></th></tr></tbody></table></div>

| **Part**    | **Time**    | **Task**                                                      |
| ----------- | ----------- | ------------------------------------------------------------- |
| **Part 1**  | **~30 min** | Bug fixes — find and fix 3 planted bugs (4 failing tests)     |
| **Part 2**  | **~20 min** | Feature stub — implement the discount code handler            |
| **Part 3**  | **~40 min** | UI implementation — build the PriceCalculator React component |
| **Stretch** | **+10 min** | Discount code validation in the UI (optional)                 |

Setup command for the candidate:

cd backend && npm install && npm test # shows 6 failing tests

cd ../frontend && npm install && npm run dev

# **Part 1 — Bug Answer Key**

Three bugs are planted in the backend. A bonus fourth is present for strong candidates who finish early. Each maps to at least one failing test.

| **Bug 1 — Inverted date comparison accepts expired discount codes**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                               |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/services/PricingService.ts — validateDiscountCode()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                   |
| // An expired code has expiresAt in the PAST (< now)<br><br>// Bug: > instead of < — logic is backwards<br><br>if (discount.expiresAt && new Date(discount.expiresAt) > new Date()) {<br><br>throw new PricingError(\`Discount code '\${code}' has expired\`, 422);<br><br>}<br><br>// Effect: expired codes (past date) pass through as valid<br><br>// valid codes (future date) are rejected as 'expired'                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| if (discount.expiresAt && new Date(discount.expiresAt) < new Date()) {<br><br>throw new PricingError(\`Discount code '\${code}' has expired\`, 422);<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| **Explanation:**<br><br>A code is expired when its expiresAt timestamp is in the past — i.e. new Date(expiresAt) &lt; new Date(). The bug uses &gt; instead of &lt;, which inverts the logic: codes with a past expiry date evaluate to false (past < now is expected, but &gt; makes it false), so they slip through as valid. Meanwhile, codes with a future expiry date evaluate to true (future > now), so they get incorrectly rejected. The seed data makes this visible: EXPIRED5 (2025-01-01) should be rejected but is accepted, while SUMMER10 (2026-12-31) would be incorrectly rejected if tested. Fix: flip > to <.<br><br>**✅ Green flag:** Spots the operator mismatch quickly. Can articulate the before/after: 'past < now = expired, so I need <'. May also note that naming the condition to a boolean (const isExpired = ...) makes this class of bug easier to catch in review.<br><br>**❌ Red flag:** Changes the operator but cannot explain why it was wrong. Confuses 'expiresAt is in the past' with 'expiresAt is greater than now'. |

| **Bug 2 — applyDiscount returns the discount amount, not the discounted price**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/services/PricingService.ts — applyDiscount()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                           |
| // discount.rate stored as whole number: 10 = 10 %<br><br>// Bug: returns the AMOUNT saved, not the PRICE to charge<br><br>return subtotal \* (discount.rate / 100);<br><br>// e.g. 10 % of \$100 = \$10 ← the savings, not the charge                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| return subtotal \* (1 - discount.rate / 100);<br><br>// e.g. \$100 × (1 − 0.10) = \$90 ← the amount to charge                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| **Explanation:**<br><br>The formula subtotal × (rate/100) computes the discount amount (the money saved), not the discounted price (the money owed). For a 10% discount on a \$100 order, the buggy result is \$10 instead of \$90. Downstream, the handler uses this as \`total\`, so the customer is charged only the savings amount rather than the remaining balance. The correct formula is subtotal × (1 − rate/100), which scales the price down by the discount percentage.<br><br>**✅ Green flag:** Immediately distinguishes 'amount saved' from 'price after discount'. Might note that storing rate as 10 (not 0.10) is a design choice and that the comment in the code makes this explicit — reading the JSDoc is part of debugging.<br><br>**❌ Red flag:** Fixes the test by trial and error (tries different formulas until the numbers match) without explaining the conceptual error. |

| **Bug 3 — reduce accumulator initialised to 1 instead of 0**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **File:** src/handlers/quoteHandlers.ts — batchQuote()                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                       |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                              |
| // BUG: initial value is 1, not 0<br><br>const subtotal = lines.reduce((sum, l) => sum + l.lineTotal, 1);                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     |
| const subtotal = lines.reduce((sum, l) => sum + l.lineTotal, 0);                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| **Explanation:**<br><br>Array.prototype.reduce(fn, initialValue) starts the accumulation from initialValue. Setting it to 1 inflates every subtotal by exactly \$1 regardless of line count — a 5-unit order at \$25 returns \$126 instead of \$125. This is particularly hard to catch in casual testing because the error is constant (always +\$1) rather than proportional. The test catches it because it asserts an exact value.<br><br>**✅ Green flag:** Spots the wrong initial value immediately when reading the reduce call. Understands that the error is constant, not proportional, and can explain why that makes it harder to notice through visual inspection of totals.<br><br>**❌ Red flag:** Finds it only after a long debugging session. Alternatively: changes the expected value in the test to 126 rather than fixing the source. |

| **Bug 4 — Wrong HTTP status code in error handler**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                         |
| --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **File:** src/handlers/quoteHandlers.ts — batchQuote() catch block                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| **Buggy code:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                             |
| } catch (err) {<br><br>if (err instanceof PricingError) {<br><br>// Always returns 400, ignoring err.statusCode<br><br>return badRequest(err.message);<br><br>}<br><br>throw err;<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
| **Fix:**                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| } catch (err) {<br><br>if (err instanceof PricingError) {<br><br>if (err.statusCode === 404) return notFound(err.message);<br><br>if (err.statusCode === 400) return badRequest(err.message);<br><br>return unprocessable(err.message);<br><br>}<br><br>throw err;<br><br>}                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| **Explanation:**<br><br>PricingError carries a statusCode (400, 404, or 422) to distinguish between bad input, a missing resource, and a business-rule violation. Blindly returning 400 for all cases breaks REST semantics — callers cannot tell 'the rate card ID you sent does not exist' (404) from 'the JSON you sent is malformed' (400), which matters for retry logic, error display, and API clients that branch on status codes. The fix routes each statusCode to the appropriate response helper.<br><br>**✅ Green flag:** Understands HTTP status code semantics without prompting. Notes that 404 vs. 400 matters for idempotent retries. May mention that PricingError.statusCode is part of the public contract of the service layer.<br><br>**❌ Red flag:** Changes the status code to 404 everywhere (overcorrects). Or fixes the test by changing the assertion to 400 rather than fixing the handler. |

# **Part 2 — Discount Handler Stub Answer**

The stub is in src/handlers/discountHandlers.ts. The complete implementation:

export async function getDiscountCode(event: LambdaEvent): Promise&lt;LambdaResponse&gt; {

const code = event.pathParameters?.\['code'\] ?? '';

const discount = store.discountCodes.findByCode(code);

if (!discount || !discount.active) {

return notFound(\`Discount code '\${code}' not found\`);

}

if (discount.expiresAt && new Date(discount.expiresAt) < new Date()) {

return notFound(\`Discount code '\${code}' has expired\`);

}

// Return only public metadata — do NOT expose the rate

return ok({

code: discount.code,

description: discount.description,

valid: true,

});

}

<div class="joplin-table-wrapper"><table><tbody><tr><th><p><strong>What to look for:</strong></p><ul><li>Extracts the code from pathParameters, not from the query string.</li><li>Handles both inactive codes (!discount.active) and expired codes (expiresAt &lt; now).</li><li>Does NOT expose discount.rate in the response — a security-conscious candidate will notice this is intentionally absent from the API spec.</li><li>Follows the same ok() / notFound() helper pattern as the other handlers.</li></ul></th></tr></tbody></table></div>

**✅ Strong:** Writes the expiry check without being prompted. Notices the rate field is absent from the README spec and deliberately omits it.

**⚠️ Weak:** Implements the happy path only and misses the expiry check. Returns the rate in the response payload.

# **Part 3 — UI Stub Answer (PriceCalculator.tsx)**

The complete implementation of PriceCalculator.tsx, replacing all TODO placeholders:

import { useState, useEffect } from 'react';

import { RateCard, QuoteResult, fetchRateCards, calculateQuote } from '../api/client';

import { RateCardList } from './RateCardList';

import { QuoteResultPanel } from './QuoteResult';

export function PriceCalculator() {

const \[rateCards, setRateCards\] = useState&lt;RateCard\[\]&gt;(\[\]);

const \[selectedCardId, setSelectedCardId\] = useState('');

const \[quantity, setQuantity\] = useState(1);

const \[discountCode, setDiscountCode\] = useState('');

const \[quote, setQuote\] = useState&lt;QuoteResult | null&gt;(null);

const \[loading, setLoading\] = useState(false);

const \[error, setError\] = useState&lt;string | null&gt;(null);

// TODO 1 — fetch rate cards on mount

useEffect(() => {

fetchRateCards()

.then(setRateCards)

.catch((e: Error) => setError(e.message));

}, \[\]);

// TODO 2 — calculate quote

async function handleCalculate() {

if (!selectedCardId) return setError('Please select a rate card');

if (quantity < 1) return setError('Quantity must be at least 1');

setLoading(true);

setError(null);

try {

const result = await calculateQuote(

\[{ rateCardId: selectedCardId, quantity }\],

discountCode || undefined,

);

setQuote(result);

} catch (e: unknown) {

setError(e instanceof Error ? e.message : 'An error occurred');

} finally {

setLoading(false);

}

}

// JSX — state wiring is otherwise unchanged from the stub

// (RateCardList, quantity input, button, error, QuoteResultPanel

// are already in the stub — just remove the void suppressors)

}

<div class="joplin-table-wrapper"><table><tbody><tr><th><p><strong>What to look for:</strong></p><ul><li>useEffect with empty dependency array [] — correctly runs only on mount.</li><li>Proper loading / error state management — sets loading before the call, clears it in finally.</li><li>Passes discountCode || undefined (not an empty string) — avoids sending an empty discount code to the API.</li><li>Error handling: catches both API errors (thrown by the client) and unexpected exceptions.</li><li>(Stretch) validateDiscountCode() called on the Validate button, result shown inline before submit.</li></ul></th></tr></tbody></table></div>

**✅ Strong:** Wires loading/error correctly, uses finally, avoids passing empty string as discountCode. May add a loading spinner or disabled button state without being asked.

**⚠️ Weak:** Forgets to clear loading on error (no finally). Does not handle the error state. Passes empty discountCode string to the API.

# **Interviewer Probes — During Code Review**

Ask at least one of these depending on which parts the candidate completed. These distinguish someone who got to working code from someone who understands what they wrote.

## **1\. On Bug 1 (inverted expiry)**

**_""How would you write a unit test for the fixed validateDiscountCode that guards against this bug being reintroduced?""_**

**Strong answer:** Should mention a test with a code whose expiresAt is 1 second in the past (using new Date(Date.now() - 1000).toISOString()), and a separate test with a code expiring 1 second in the future. Both boundary cases together pin down the &lt; vs &gt; distinction. Strong candidates may also suggest a named boolean like const isExpired = new Date(expiresAt) < new Date() to make the intent self-documenting.

## **2\. On Bug 2 (discount formula)**

**_""What would happen to the discountAmount line if the applyDiscount bug is fixed but subtotal has the reduce bug (Bug 3) adding \$1?""_**

**Strong answer:** With Bug 3 present, subtotal = \$126 (not \$125). After fixing Bug 2, total = \$126 × 0.90 = \$113.40. discountAmount = \$126 − \$113.40 = \$12.60. But the correct answer is \$11.25 (10% of \$112.50). Both bugs compound — fixing one reveals the other. Strong candidates trace the full data flow across all bugs.

## **3\. On Bug 3 (reduce)**

**_""Why did you set the initial value to 0 rather than omitting it entirely?""_**

**Strong answer:** Without an initial value, reduce uses the first element as the accumulator. For an array of QuoteLineResult objects, that would accumulate a QuoteLineResult as the 'sum' — immediate type error. Always providing an initial value for reduce is defensive and correct. Strong candidates also note that reduce with no initial value throws on an empty array.

## **4\. On Bug 4 (HTTP status codes)**

**_""In a real API Gateway + Lambda deployment, what happens to unhandled exceptions thrown from a Lambda handler?""_**

**Strong answer:** API Gateway maps unhandled Lambda errors to 502 Bad Gateway (or 500 depending on configuration), not 500. The error message is not forwarded to the client. This is why the try-catch in the handler is important — to produce a meaningful, client-safe response rather than a generic 502.

## **5\. On the UI**

**_""Your useEffect fetches rate cards on mount. What happens if the user navigates away before the request completes and the component unmounts?""_**

**Strong answer:** The async callback will still run and call setRateCards on an unmounted component — React will log a warning. The fix is to use an AbortController or a cancelled flag: if cancelled, skip setState. Strong candidates know the pattern; very strong ones implement it.

# **Evaluation Scorecard**

Score each dimension 1–5 using observations from the exercise. Annotate with specific evidence.

| **Dimension**                     | **Score** | **Notes**                                                                             |
| --------------------------------- | --------- | ------------------------------------------------------------------------------------- |
| **Async / Promise fluency**       | / 5       | Correctly identified async forEach? Explained why? Considered Promise.all?            |
| **TypeScript confidence**         | / 5       | Used types naturally. Understood the data model from types.ts without guidance.       |
| **Debugging methodology**         | / 5       | Systematic (test → trace → fix) or random thrashing?                                  |
| **Boundary / edge case thinking** | / 5       | Found and fixed Bug 2 tier boundary. Checked for null (Bug 4 bonus)?                  |
| **API design awareness**          | / 5       | Noticed the rate field omission in the discount response spec. Handled 4xx correctly. |
| **React fundamentals**            | / 5       | useEffect dependency array correct. Loading/error state clean. Avoids memory leaks.   |
| **Code quality & style**          | / 5       | Follows the existing patterns in the repo. Clean, readable additions.                 |
| **Communication during review**   | / 5       | Explains their thinking when asked probes. Honest about what they don't know.         |

| **Total Score (out of 40)** | **Recommendation**                                             |
| --------------------------- | -------------------------------------------------------------- |
| **34–40**                   | Strong — advance to system design round                        |
| **27–33**                   | Advance with notes — one or two gaps to probe in system design |
| **20–26**                   | Borderline — discuss with team before final round              |
| **< 20**                    | Do not advance — fundamental gaps in async/TS/React            |

| **Field**             | **Notes**                                               |
| --------------------- | ------------------------------------------------------- |
| **Interviewer**       |                                                         |
| **Date**              |                                                         |
| **Bugs found**        | Bug 1 (async) Bug 2 (tier) Bug 3 (discount) Bug 4 bonus |
| **Stub completed?**   | Yes / Partial / No                                      |
| **UI completed?**     | Yes / Partial / No — stretch goal?                      |
| **Strongest moment**  |                                                         |
| **Biggest gap**       |                                                         |
| **Score (out of 40)** |                                                         |
| **Recommendation**    | Advance / Hold / Do Not Advance                         |
| **Summary**           |                                                         |

_— End of Interviewer Guide —_