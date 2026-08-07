**TallyBoard**

Coding Interview Guide - Senior Engineer (Full-Stack)

Stack: Java 11 / Spring Boot · React 18 / TypeScript

**INTERVIEWER COPY - Do not share**

# **Setup Verification**

| **⚙ Before the interview**                                                     |
| ------------------------------------------------------------------------------ |
| Run mvn test in the project root - you should see 15 failing tests.            |
| Run cd frontend && npm install && npm run dev and confirm the UI loads.        |
| Start the backend: mvn spring-boot:run - API on <http://localhost:8080>.       |
| Confirm GET /api/projects returns 3 projects in your browser or Postman.       |
| Share the tallyboard/ folder with the candidate at session start (not before). |

# **Exercise Overview**

| **Part** | **Task**                                                               | **Time**    |
| -------- | ---------------------------------------------------------------------- | ----------- |
| **1**    | Bug fixes - 4 bugs in ExpenseService.java (read failing tests first)   | **~30 min** |
| **2**    | Implement ExpenseDashboard.tsx - React UI stub                         | **~30 min** |
| **3**    | Implement findTopExpenses + longestExpenseStreak - two Java algorithms | **~35 min** |
| **4**    | Code review / design discussion (if time allows)                       | **~15 min** |

# **Part 1 - Bug Fixes (~30 min)**

Ask the candidate to run mvn test and describe what the failing tests tell them. They should read tests in ExpenseServiceTest.java and locate the bugs in ExpenseService.java.

Do not point them to specific methods. Let them navigate the codebase.

| **📣 Interviewer verbal brief (share at start of Part 1)**                                                                                        |
| ------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Run mvn test - you'll see some failing tests. Read the test names to understand what's broken, then find and fix the bugs in the service layer." |
| "The tests are in src/test/java - start there before reading the source."                                                                         |
| "You have about 25 minutes. It's fine if you find 2 of the 3."                                                                                    |

## **Answer Key**

| **Bug 1** - ExpenseService.java - getApprovedTotal()                                                                                                                                                                          |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Buggy line:**<br><br>.mapToInt(e -> (int) e.getAmount()).sum()                                                                                                                                                              |
| **Fix:**<br><br>.mapToDouble(Expense::getAmount).sum()<br><br>mapToInt casts double to int, truncating decimal amounts. 45.50 + 30.75 becomes 45 + 30 = 75 instead of 76.25. Realistic mistake - mixing numeric stream types. |

| **Bug 2** - ExpenseService.java - canApprove()                                                                                                                                                                                                                            |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Buggy line:**<br><br>return expense.getStatus() != ExpenseStatus.PENDING;                                                                                                                                                                                               |
| **Fix:**<br><br>return expense.getStatus() == ExpenseStatus.PENDING;<br><br>Inverted condition - only PENDING expenses should be approvable. As written, it approves already-approved/rejected expenses and blocks new ones. Three tests fail: one for each status value. |

| **Bug 3** - ExpenseService.java - getByDateRange() (HARDEST)                                                                                                                                                                                                                                                                             |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Buggy line:**<br><br>.filter(e -> e.getDate().isAfter(from) && e.getDate().isBefore(to))                                                                                                                                                                                                                                               |
| **Fix:**<br><br>.filter(e -> !e.getDate().isBefore(from) && !e.getDate().isAfter(to))<br><br>LocalDate.isAfter() and isBefore() are strictly exclusive. Dates exactly on the from or to boundary are silently excluded. Fix uses negated isBefore/isAfter for inclusive comparison, or equivalent: isEqual(from)\|isAfter(from) pattern. |

| **Bug 4** - ExpenseService.java - getCurrentMonthExpenses() (HARDEST)                                                                                                                                                                                                                                                                                                                    |
| ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Buggy line:**<br><br>.filter(e -> e.getDate().getMonthValue() == today.getMonthValue())                                                                                                                                                                                                                                                                                                |
| **Fix:**<br><br>.filter(e -> e.getDate().getMonthValue() == today.getMonthValue() && e.getDate().getYear() == today.getYear())<br><br>getMonthValue() returns only the month number (1-12), ignoring the year. An expense from August 2025 passes the filter in August 2026. The fix adds a year check. Invisible to code review unless a reviewer thinks to ask 'what about last year?' |

| **✅ / ❌ Green and Red Flags - Part 1**                                                      |
| --------------------------------------------------------------------------------------------- |
| GREEN: Reads failing tests first, infers bug location from test names before browsing source. |
| GREEN: Catches Bug 2 by tracing canApprove logic step by step.                                |
| GREEN: Explains why isAfter/isBefore are exclusive (Java LocalDate semantics).                |
| GREEN: Spots Bug 4's missing year check without being told - very strong signal.              |
| GREEN: Finds all 4 independently - exceptional at this level.                                 |
| AMBER: Fixes Bug 1 by wrapping in (double) cast - correct result, not idiomatic.              |
| AMBER: Needs a hint for Bug 3 or Bug 4 but fixes correctly once pointed at the method.        |
| RED: Goes straight to source code before reading tests - reactive, not systematic.            |
| RED: Fixes bugs by changing test assertions instead of fixing production code.                |
| RED: Finds only 1-2 bugs after 30 minutes even with gentle direction.                         |

Probe after Part 1: "Bug 2 - could this have been caught in code review? What would you look for in a PR that touched this method?"

# **Part 2 - React UI Stub (~30 min)**

The candidate implements ExpenseDashboard.tsx. The TODOs in the component are brief - no step-by-step hints. Give the verbal brief below, then let them work.

| **📣 Interviewer verbal brief (share at start of Part 2)**                                                                                             |
| ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| "The backend is running. Open ExpenseDashboard.tsx - it's a stub with TODO comments."                                                                  |
| "Implement the four TODOs: fetch projects on mount, fetch expenses when project changes, approve/reject handlers, and the filter + total derivations." |
| "The API functions are already wired in api.ts - check the function names there."                                                                      |
| "The UI shell is in place - don't redesign it, just make it work."                                                                                     |

## **Answer Key - ExpenseDashboard.tsx**

Fetch projects on mount:

| useEffect(() => {                                                |
| ---------------------------------------------------------------- |
| getProjects().then(setProjects).catch(e => setError(e.message)); |
| }, \[\]);                                                        |

Fetch expenses when project changes:

| useEffect(() => {                                      |
| ------------------------------------------------------ |
| if (!selectedProjectId) { setExpenses(\[\]); return; } |
| setLoading(true); setError(null);                      |
| getProjectExpenses(selectedProjectId)                  |
| .then(data => setExpenses(data))                       |
| .catch(e => setError(e.message))                       |
| .finally(() => setLoading(false));                     |
| }, \[selectedProjectId\]);                             |

Approve handler (reject follows the same pattern):

| async function handleApprove(expenseId: string) {         |
| --------------------------------------------------------- |
| try {                                                     |
| await approveExpense(expenseId);                          |
| const data = await getProjectExpenses(selectedProjectId); |
| setExpenses(data);                                        |
| } catch (e: any) { setError(e.message); }                 |
| }                                                         |

Filter and total derivations (derived from state - no extra useEffect needed):

| const filteredExpenses = statusFilter === 'ALL'    |
| -------------------------------------------------- |
| ? expenses                                         |
| : expenses.filter(e => e.status === statusFilter); |
|                                                    |
| const approvedTotal = expenses                     |
| .filter(e => e.status === 'APPROVED')              |
| .reduce((sum, e) => sum + e.amount, 0);            |

| **✅ / ❌ Green and Red Flags - Part 2**                                                                     |
| ------------------------------------------------------------------------------------------------------------ |
| GREEN: Reads api.ts before implementing - understands the contract without being told.                       |
| GREEN: Derives filteredExpenses and approvedTotal from state without an extra useEffect.                     |
| GREEN: Adds loading and error state handling on first pass without prompting.                                |
| GREEN: Refreshes the expense list after approve/reject (not just updating local state).                      |
| AMBER: Uses a separate useEffect to recalculate approvedTotal - correct but redundant.                       |
| AMBER: Needs prompting to add finally() for loading cleanup.                                                 |
| RED: Calls getApprovedTotal() API endpoint from frontend - misses that it can be derived from existing data. |
| RED: No error handling on any fetch call.                                                                    |
| RED: Puts approve logic directly in onClick, bypasses the handler pattern.                                   |

# **Part 3 - Two Java Algorithms (~35 min)**

Both stubs are in ExpenseAnalyzer.java. Give the verbal brief for each in sequence - do not reveal both at once. Allow ~15 min per problem.

## **Part 3a - findTopExpenses (~15 min)**

| **📣 Interviewer verbal brief (share at start of 3a)**                                                                     |
| -------------------------------------------------------------------------------------------------------------------------- |
| "Implement findTopExpenses - given a list of expenses and k, return the k highest-amount expenses sorted descending."      |
| "The Javadoc has two worked examples. Aim for better than O(n log n) - there's a clean O(n log k) solution."               |
| "Hint if stuck after 5 minutes: think about what data structure efficiently maintains the k largest elements as you scan." |

## **Answer Key - findTopExpenses (min-heap, O(n log k))**

| public List&lt;Expense&gt; findTopExpenses(List&lt;Expense&gt; expenses, int k) { |
| --------------------------------------------------------------------------------- |
| if (expenses == null \| expenses.isEmpty() \| k <= 0) {                           |
| return Collections.emptyList();                                                   |
| }                                                                                 |
| // Min-heap: smallest amount sits at the top                                      |
| PriorityQueue&lt;Expense&gt; minHeap = new PriorityQueue<>(                       |
| Comparator.comparingDouble(Expense::getAmount)                                    |
| );                                                                                |
| for (Expense e : expenses) {                                                      |
| minHeap.offer(e);                                                                 |
| if (minHeap.size() > k) {                                                         |
| minHeap.poll(); // evict the smallest, keeping only top-k                         |
| }                                                                                 |
| }                                                                                 |
| List&lt;Expense&gt; result = new ArrayList<>(minHeap);                            |
| result.sort(Comparator.comparingDouble(Expense::getAmount).reversed());           |
| return result;                                                                    |
| }                                                                                 |

Acceptable alternative - sort the full list, take first k:

| expenses.stream()                                                  |
| ------------------------------------------------------------------ |
| .sorted(Comparator.comparingDouble(Expense::getAmount).reversed()) |
| .limit(k)                                                          |
| .collect(Collectors.toList());                                     |

O(n log n) - correct and readable, not optimal. Accept it; probe on complexity trade-off.

| **✅ / ❌ Green and Red Flags - findTopExpenses**                                          |
| ------------------------------------------------------------------------------------------ |
| GREEN: Reaches for PriorityQueue and explains why min-heap is correct here (not max-heap). |
| GREEN: Handles edge cases: k=0, empty list, k > list size.                                 |
| GREEN: Explains O(n log k) vs O(n log n) - knows when it matters.                          |
| AMBER: Sorts and slices - correct, not optimal. Ask about complexity.                      |
| AMBER: Forgets to sort the final result before returning (heap order is not sorted).       |
| RED: Cannot explain why a min-heap and not a max-heap.                                     |
| RED: No edge case handling - crashes on empty list or k=0.                                 |

## **Part 3b - longestExpenseStreak (~20 min)**

| **📣 Interviewer verbal brief (share at start of 3b)**                                                                                                                    |
| ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| "Implement longestExpenseStreak - given a list of expenses, find the length of the longest run of consecutive calendar days on which at least one expense was submitted." |
| "Multiple expenses on the same day count as a single day. The Javadoc has three worked examples."                                                                         |
| "An O(n log n) sort-based approach is acceptable. Hint if stuck after 5 minutes: what if you put all dates in a Set first?"                                               |

## **Answer Key - longestExpenseStreak (HashSet, O(n))**

| public int longestExpenseStreak(List&lt;Expense&gt; expenses) { |
| --------------------------------------------------------------- |
| if (expenses == null \| expenses.isEmpty()) return 0;           |
| Set&lt;LocalDate&gt; dates = expenses.stream()                  |
| .map(Expense::getDate)                                          |
| .collect(Collectors.toSet());                                   |
| int max = 0;                                                    |
| for (LocalDate d : dates) {                                     |
| // Only start counting from the beginning of a streak           |
| if (!dates.contains(d.minusDays(1))) {                          |
| int len = 1;                                                    |
| while (dates.contains(d.plusDays(len))) len++;                  |
| max = Math.max(max, len);                                       |
| }                                                               |
| }                                                               |
| return max;                                                     |
| }                                                               |

Key insight: only process a date if the previous day is NOT in the set - this ensures each streak is counted exactly once from its start. Without this guard, every date in a run of n would trigger a full O(n) scan, giving O(n²). With the guard, the total inner-loop iterations across all starts equals exactly n, giving O(n) overall.

Acceptable O(n log n) alternative - sort unique dates, iterate linearly:

| List&lt;LocalDate&gt; sorted = expenses.stream()                       |
| ---------------------------------------------------------------------- |
| .map(Expense::getDate).distinct()                                      |
| .sorted().collect(Collectors.toList());                                |
| int max = 1, cur = 1;                                                  |
| for (int i = 1; i < sorted.size(); i++) {                              |
| cur = sorted.get(i).equals(sorted.get(i-1).plusDays(1)) ? cur + 1 : 1; |
| max = Math.max(max, cur);                                              |
| }                                                                      |
| return sorted.isEmpty() ? 0 : max;                                     |

| **✅ / ❌ Green and Red Flags - longestExpenseStreak**                                                     |
| ---------------------------------------------------------------------------------------------------------- |
| GREEN: Puts dates in a Set and uses the 'only process streak starts' trick - explains why.                 |
| GREEN: Handles duplicates naturally via the Set (no special case needed).                                  |
| GREEN: Explains O(n) amortised - total inner-loop work is bounded by n regardless of streak length.        |
| AMBER: Uses sort + linear scan - correct O(n log n), show them the O(n) approach after.                    |
| AMBER: Arrives at correct answer via brute-force O(n²) - flag the complexity, not a blocker at this level. |
| RED: Mutates the input list (sorts in-place) without acknowledging it.                                     |
| RED: Off-by-one: returns n−1 when all dates are consecutive.                                               |

# **Part 4 - Code Review Discussion (~15 min, if time allows)**

Walk through these discussion points. No coding required - verbal answers only.

## **Topic A - Thread Safety**

"InMemoryStore uses a plain ArrayList. What happens if two requests try to approve the same expense simultaneously?"

Looking for: race condition on store.updateExpense - last-write-wins. Fix: ConcurrentHashMap keyed by id, or synchronized blocks, or optimistic locking with a version field.

## **Topic B - Persistence**

"Right now all data resets when the server restarts. How would you replace InMemoryStore with a real database while keeping the service layer unchanged?"

Looking for: Spring Data JPA / repository pattern - service layer calls a repository interface; swap implementation without touching ExpenseService. Bonus: mentions H2 for tests.

## **Topic C - AI and financial figures**

"You've worked on fintech systems. Imagine a product manager asks you to use an LLM to auto-categorise submitted expenses. Where does that fit - and where does it definitely not fit?"

Looking for: LLM fine for suggested category label that a human can override. NOT for computing totals, running approval logic, or any number that lands in a ledger. Hallucination risk in financial figures is a hard boundary.

# **Scorecard**

| **Area**                 | **Weight** | **Strong Hire**                                                        | **Hire**                                               | **Pass**                                       |
| ------------------------ | ---------- | ---------------------------------------------------------------------- | ------------------------------------------------------ | ---------------------------------------------- |
| **Bug Fixes (Part 1)**   | 25%        | Finds all 4, explains root cause clearly                               | Finds 3, logical approach                              | Finds ≤2 or needs heavy hinting for each       |
| **React UI (Part 2)**    | 30%        | Full implementation, derived state, error handling                     | Core fetch + render working, minor gaps                | Stub mostly incomplete or broken               |
| **Algorithms (Part 3)**  | 25%        | Both solved; min-heap + HashSet streak trick with complexity reasoning | One fully solved, second partially or via brute-force  | Cannot complete either algorithm independently |
| **Code Review (Part 4)** | 10%        | Spots thread safety + persistence gaps unprompted                      | Answers questions correctly when asked                 | Surface-level answers only                     |
| **AI Judgment**          | 10%        | Clear boundary: LLM for labelling, NOT for financial computation       | Correct but needs prompting to articulate the boundary | No clear mental model for LLM risks in fintech |

| **💡 Hiring bar for this role**                                                                                                                   |
| ------------------------------------------------------------------------------------------------------------------------------------------------- |
| Strong Hire: 2+ of 3 bugs found independently, working React UI with proper async handling, heap solution or strong rationale for sort-and-slice. |
| Hire: At least 2 bugs found, UI mostly functional (fetch works, filter works), algorithm produces correct output.                                 |
| Pass: Finds only 1 bug with hints, UI stub largely incomplete, no algorithm progress after 10 minutes.                                            |