# Example WBR Agenda

## Input

```text
Auth rate dropped from 91.8% to 90.6%.
Retry success improved from 18% to 22%.
Top final declines increased in LATAM.
Support tickets mention checkout timeouts after Thursday's release.
Network-token volume is flat.
```

## Output

### Payment WBR

Audience: payments, checkout/product, fraud, support, and finance operator.

Data basis: user-provided weekly metric snapshot. Missing: processor split,
region-level raw decline reasons, and support ticket counts.

1. What changed this week?
   - Authorization rate declined 1.2 points.
   - Retry success improved, which partially offset the drop.
   - LATAM final declines increased.
   - Support is seeing checkout-timeout reports after Thursday's release.

2. What is controllable?
   - Controllable: checkout timeout investigation, retry tuning, customer outreach
     cadence, network-token adoption.
   - Less controllable: issuer behavior in LATAM until raw decline reasons confirm
     the pattern.

3. What are we doing?
   - Checkout team: review Thursday release timing against timeout spike.
   - Payments team: split LATAM declines by processor, issuer, and raw reason.
   - Support: tag all timeout-related tickets for the next week.
   - Product: review whether network-token enrollment is blocked in the current
     checkout flow.

4. Exec-ready summary
   - Payment success softened this week, but the early signal points to a
     controllable checkout issue plus a regional decline-pattern change. The team
     is separating internal release impact from external issuer behavior before
     changing routing or retry rules.

5. Next data pulls
   - Raw decline reasons for LATAM by processor and issuer.
   - Timeout rate before and after Thursday's checkout release.
   - Support ticket count and examples tagged as checkout timeout.
   - Recovery impact by retry attempt.

6. Owner actions
   - Checkout/product: validate timeout spike against release timing.
   - Payments: split LATAM declines by processor, issuer, and raw reason.
   - Support: tag timeout-related tickets and report trend next Monday.
   - Finance: review whether the net sales impact is material enough for the MBR.

## No Metrics Provided Behavior

If the user only says:

```text
Build a Monday WBR agenda from these payment metrics.
```

and does not include metrics, the copilot should first try available data sources.
If it cannot access payment metrics, it should ask for the minimal metric pack:

- current week vs prior week payment success or authorization rate
- top 3 movement areas by method, processor, region, or decline reason
- retry/recovery signal if available
- known launches, incidents, or support themes
