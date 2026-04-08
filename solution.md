# Solution

## Q1: Design Choices - The Settlement Contract

### 1.

The frontend fetches worklogs and the backend re-fetches independently when admin clicks confirm. Anything that changes in between causes a mismatch.

- **New worklogs created** - backend picks them up, but the admin never saw them. Total goes up.
- **Segments added** - backend calculates fresh, so per-worklog amounts differ from preview.
- **Adjustments added** - backend calculates fresh, so per-worklog amounts differ from preview.

### 2.

No - the admin gets zero signal.

The confirm button calls the API with an empty body. Backend returns the total amount. The hook stores this, and the success screen renders that backend number - not the total that the admin reviewed.

The preview screen is fully replaced by the success screen. There's no comparison, no warning, no diff. If the admin approved $1,200 but the backend settled $1,450, the screen just shows "$1,450.00" with no indication anything changed.

### 3.

**Retry problem:** The POST carries no worklog IDs, no amounts, no idempotency key - backend can't tell a retry from a new request.

If the first request succeeds but the response is lost, the retry now returns empty (all SETTLED). Backend returns a "no open worklogs" error, but the admin expects an amount which they had approved recently. The admin sees an error for a settlement that already went through successfully. If new worklogs arrived between requests, the retry settles them into a second remittance - without admin review.

**The in-memory Set:**

- **Helps** within one server process - skips already-settled IDs.
- **Doesn't help** across multiple instances or after restarts - the Set is in memory only.
- **Causes a bug** - the Set is never cleared. When a worklog gets reopened, the ID stays in the Set. That worklog is skipped forever until the server restarts.

### 4.

**Frontend sends:**

```json
{
  "idempotencyKey": "uuid-v4",
  "userId": "user-123",
  "worklogIds": ["WL-001", "WL-003"],
  "expectedTotal": 1250.0
}
```

**Backend validates before executing:**

1. **Idempotency key** - check a persistent DB table. If the key exists, return an error message with proper info so the admin knows what happened. Retries are safe.
2. **Ownership + status** - all worklogIds belong to userId and are still OPEN. Reject if any changed.
3. **Amount check** - recalculate total from current segments/adjustments. If it doesn't match `expectedTotal`, reject so admin can re-review.
4. **Single transaction** - mark worklogs SETTLED, create remittance, create line items all in one DB transaction. Currently these run separately, so a crash mid-way leaves worklogs SETTLED with no remittance.

**Retries:** Frontend generates a UUID per confirm click, reuses it on retry. Backend stores logs with `key` after success, returns a proper error message on duplicate key. Remove the in-memory Set entirely - it's unreliable and blocks reopened worklogs. If we need performance, we should cache this info inside Redis so all server instances use the same data, and we also need to update this Set from time to time - like when any worklog gets reopened, we need to update this Set.

---

## Q2: Trace the State

### 1.

The poll hasn't fired yet, so the component still has the old data in its local state.

The confirm button still shows **"Confirm Settlement - $50.00"** because previewTotal is -$250 + $300 = $50.

The admin has no idea that ADJ-004 (+$300) now exists in the database. The frontend is showing stale data.

### 2.

The backend fetches fresh data from the database. For WL-003 it now gets three adjustments:

- Segments: (2 × $50) + (3 × $50) = $250
- ADJ-002: -$500
- ADJ-004: +$300 (the new one)
- WL-003 total: $250 - $500 + $300 = **$50.00**

For WL-004 nothing changed:

- Segments: 5 × $60 = $300
- No adjustments
- WL-004 total: **$300.00**

Total remittance amount: $50 + $300 = **$350.00**

### 3.

The admin sees **$350.00** on the success screen.

They reviewed and approved **$50.00**, but the actual settlement was $350.00 - that's a $300 difference. The success screen just shows the backend's number with no comparison to what was previewed.

This could have been caught or prevented in a few places:

- **Frontend** - the hook could compare the returned total against previewTotal before showing the success screen. If they don't match, show a warning instead of silently accepting it.
- **Backend** - the API could accept an expectedTotal from the frontend. Before executing the settlement, the backend recalculates and compares. If they differ, reject the request and ask the admin to re-review.
- **Frontend** - stop the 30s poll from silently updating state. Or at least show a visual alert when data changes while the admin is reviewing.

### 4.

This is not just a UX problem - it can cause real financial harm.

**Example scenario:** Say instead of a +$300 corrective adjustment, someone adds a large fraudulent or mistaken adjustment of +$50,000 to WL-003 while the admin is reviewing. The admin sees $50.00 on screen and thinks "that's fine, small amount, let me approve it." They click confirm. The backend settles for $50,050.00 because it picks up the new adjustment. A $50,050 remittance gets created and potentially paid out to Bob - all because the admin thought they were approving $50.

Another example: the admin sees a negative total (say -$200) and decides to hold off and not confirm. But if a large positive adjustment gets added right before they click, the total flips to +$5,000. If they had seen the real number, they might want to verify why such a big adjustment was added before approving. Instead, the system settles it silently.

The core issue is that the admin's approval is meaningless - they're approving one number but the system executes a completely different one. Any time the admin's decision depends on the amount (approve vs. reject, escalate for review, etc.), this divergence can lead to wrong payouts.

---

## Q3: Predict the Failure Mode

### 1.

Both WL-003 and WL-004 are marked **SETTLED** in the database - that happened before the crash. But no Remittance record exists because the crash happened when trying to create it.

So Bob's worklogs are marked as settled, but he has **not been paid anything**. The money is effectively lost - the system thinks the work is done, but no payment was created.

### 2.

The POST returns a 500 error. The catch block in the hook sets the error message to "Settlement failed. Please try again." and sets isSettling back to false. The worklogs are NOT cleared from local state because that only happens inside the try block.

So the admin still sees the same table of worklogs with the confirm button enabled again, plus an error banner saying "Settlement failed. Please try again." They naturally do what the message says - they click Confirm again.

### 3.

The retry calls settlement API which queries for worklogs with status = 'OPEN'. But both WL-003 and WL-004 are now SETTLED in the database. The query returns an empty array.

Settlement API sees zero worklogs and returns null. The API handler sends back `"No open worklogs found for this user."` with a 200 status.

### 4.

The frontend receives a 200 response (so it doesn't hit the catch block). It tries to read `result.remittance.totalAmount`, but the response has no `remittance` field - it only has a `message` field. Accessing `.remittance` gives undefined, and then `.totalAmount` on undefined throws a TypeError.

This uncaught error either crashes the component (white screen if there's no error boundary) or gets swallowed depending on the React setup. Either way, the admin sees something broken - either a blank page or a generic React error. No helpful information about what went wrong.

### 5.

The full picture after the retry:

- **Database:** WL-003 and WL-004 are both SETTLED. No Remittance exists. No line items. Bob was never paid.
- **In-memory:** `settledWorkLogIds` contains both "WL-003" and "WL-004". Even if someone manually fixes the database status back to OPEN, the settlement engine will skip them because they're in the Set.
- **Frontend:** The admin sees either a crashed component or a confusing error. They have no idea what actually happened.
- **New admin:** If a fresh admin opens SettlementReview for Bob, the API fetches OPEN worklogs. Both are SETTLED, so the screen shows "No open worklogs to settle." It looks like Bob has already been paid, but he hasn't.
- **Recovery:** These worklogs can't be settled through normal operation. Even if you manually UPDATE the database to set them back to OPEN, the in-memory Set still blocks them. You'd need to either restart the server (to clear the Set) or directly insert the remittance and line items into the database by hand. There's no admin tool or API to fix this.

---

## Q4: What Happens With THIS Input?

### Step 1 - The POST

The segment endpoint inserts a new time segment (1.5 hrs × $75) for WL-002. Then it calls `reopenWorkLog("WL-002")` which sets the status back to OPEN. So WL-002 is now **OPEN** again even though it was previously settled.

### Step 2 - The review screen

The admin opens SettlementReview for Alice (USR-001). The frontend fetches open worklogs. The query finds all worklogs for USR-001 with status = 'OPEN'. That returns:

- **WL-001** (API Integration) - was always OPEN
- **WL-002** (Bug Fix Sprint) - just reopened

Both show up on the review screen.

### Step 3 - Amount calculation

**WL-001:**

- TS-001: 4 hrs × $75 = $300
- TS-002: 3.5 hrs × $75 = $262.50
- No adjustments
- **Total: $562.50**

**WL-002:**

- TS-003: 6 hrs × $75 = $450
- TS-004: 2 hrs × $75 = $150
- New segment: 1.5 hrs × $75 = $112.50
- ADJ-001: -$150
- **Total: $450 + $150 + $112.50 - $150 = $562.50**

### Step 4 - The review screen renders

The admin sees:

| WorkLog                  | Amount  |
| ------------------------ | ------- |
| WL-001 (API Integration) | $562.50 |
| WL-002 (Bug Fix Sprint)  | $562.50 |

previewTotal = $562.50 + $562.50 = **$1,125.00**

The confirm button says **"Confirm Settlement - $1,125.00"**

### Step 5 - Settlement execution

The backend fetches the same worklogs and runs `calculateWorkLogAmount` for each. The calculation logic is the same as the GET handler - both use `hours × ratePerHour` for segments and sum adjustments. So the amounts match:

- WL-001: $562.50
- WL-002: $562.50
- Remittance total: **$1,125.00**

The preview and the settlement agree this time.

### Step 6 - The ledger

- REM-001 (January): **$450.00** paid to Alice (COMPLETED)
- REM-002 (February): **$1,125.00** paid to Alice (PENDING)
- **Total paid: $1,575.00**

Now let's calculate what Alice should have actually been paid across both settlements:

- WL-001: $300 + $262.50 = $562.50
- WL-002: $450 + $150 + $112.50 - $150 = $562.50
- **Correct total: $1,125.00**

**Overpayment: $1,575.00 - $1,125.00 = $450.00**

The $450 overpayment happens because the January settlement already paid Alice $450 for WL-002's original segments and adjustment. When WL-002 got reopened, the February settlement recalculated the entire worklog from scratch - including the segments that were already paid in January. So those original segments got paid twice.

### Step 7 - The silent agreement

The preview showed $1,125.00 and the backend settled $1,125.00. They match perfectly. The admin reviewed the number, it matched what got executed, and they have no reason to question it.

This is actually **worse** than if the numbers had disagreed. When numbers disagree, at least there's a chance the admin notices something is off. But here, both the frontend and backend made the same mistake - they both included the already-paid segments in their calculation. The matching numbers give the admin false confidence that everything is correct.

The signal the admin loses is the **double-payment warning**. There's nothing in the system that says "hey, $450 of this was already paid in January via REM-001." The admin sees $562.50 for WL-002 and it looks right because that's what the segments add up to. But nobody subtracted what was already settled. The system has no concept of "previously paid amount" for a reopened worklog - it just treats it like a fresh worklog every time.

---

## Q5: Fix Evaluation

### Fix A - Backend: Use `filterNewSegments`

### 1.

`lastSettledAt` for WL-002 is `"2025-01-31T18:00:00Z"`. `filterNewSegments` keeps only segments where `createdAt > lastSettledAt`.

- TS-003 (created Jan 12) - **discarded** (before Jan 31)
- TS-004 (created Jan 13) - **discarded** (before Jan 31)
- New segment (created February) - **kept** (after Jan 31)

So only the new 1.5 hr segment survives the filter.

### 2.

The fix filters segments but NOT adjustments. So we get:

- newSegments: just the new one → 1.5 × $75 = $112.50
- adjustments: ADJ-001 still included → -$150
- `calculateWorkLogAmount(newSegments, adjustments)` = $112.50 + (-$150) = **-$37.50**

That's wrong. Alice should be paid $112.50 for the new work. The -$150 penalty was already accounted for in the January settlement ($600 - $150 = $450). Now it's being subtracted again. Alice is underpaid by $150.

So the fix swings from overpayment to underpayment.

### 3.

Say a segment is created at exactly `"2025-01-31T18:00:00Z"` - the same timestamp as `lastSettledAt`. The filter uses strict `>`, so `createdAt > lastSettledAt` is false. That segment gets excluded.

This can happen if the segment was created in the same second (or millisecond depending on DB precision) as when the settlement ran and set `lastSettledAt`. It's a race condition - unlikely in normal usage, but in a system doing batch processing it could realistically happen. The financial impact is that a legitimate segment gets silently dropped and never paid. The freelancer loses money and nobody gets an error or warning about it.

---

### Fix B - Frontend: Pre-Flight Validation

### 4.

It partially helps with the Q2 scenario. If ADJ-004 was added between page load and confirmation, the pre-flight re-fetch would pick up the new adjustment, calculate a different total, and show the warning. The admin would see the updated data and can re-review before confirming.

But it only catches changes that happen before the pre-flight fetch. It doesn't prevent changes that happen after it.

### 5.

Yes - data can change between the pre-flight fetch and the backend's execution. The pre-flight says "looks good," then a new segment or adjustment gets added, and the backend settles a different amount.

This is called a **TOCTOU (Time of Check to Time of Use)** bug. The check and the action happen at different times, so the check can't guarantee what the action will see.

In a financial system, this window is not small enough to ignore. Even if it's a few hundred milliseconds, in a busy system with multiple admins and freelancers adding segments, the risk is real. You can't rely on "it's fast enough" when money is involved.

### 6.

They are NOT guaranteed to compute the same amounts.

The GET handler in the API calculates amounts inline using `segments.reduce((sum, s) => sum + s.hours * s.ratePerHour, 0)` plus adjustments. The settlement engine uses `calculateWorkLogAmount` from the calculations module, which internally calls `calculateSegmentAmount` (also `hours * ratePerHour`) and sums adjustments.

Right now the math happens to be the same. But it's two separate implementations of the same logic - one is inline in the API handler, the other is in the calculations module. If someone updates one and forgets the other, they'll silently diverge. The pre-flight check would pass (because it compared against the API's calculation), but the backend would settle a different amount using its own calculation. This makes the pre-flight check unreliable.

---

### The Root Cause

### 7.

The fundamental flaw is that the system **never records which segments and adjustments were included in a settlement**. It only records that a worklog was settled and what the total was. When a worklog gets reopened, there's no way to know what was already paid and what's new.

The structural fix needed:

- **Data model:** Each remittance line item should record the specific segment IDs and adjustment IDs that were included in that settlement. Something like a `settled_segments` and `settled_adjustments` join table, or simply storing the IDs on the line item. This creates an audit trail of exactly what was paid.
- **Backend:** When calculating the amount for a reopened worklog, the settlement engine should look up which segments and adjustments were already included in a previous remittance and exclude them. Only unsettled segments and adjustments should be included in the new calculation.
- **Frontend:** The review screen should show which segments are new vs. already paid. The admin should clearly see "these segments were already settled in REM-001, only these new ones will be included." This is optional if needed then only we show this details otherwise not needed.

This way, no matter how many times a worklog gets reopened or how many settlements run, each segment and adjustment is only ever counted once. The system tracks what it paid at the item level, not just the worklog level.
