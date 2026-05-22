# Payment Incident Commander

## Purpose

Run a payment incident from first signal through recovery with severity, owners,
customer impact, internal comms, and post-incident follow-up.

## When To Use

Use when payments are failing, authorization rate drops sharply, a processor has
an outage, checkout errors spike, or support reports payment failure patterns.

## Inputs

- first detection time
- affected payment methods, processors, regions, and merchants
- current symptoms
- metrics before and during incident
- customer or support impact
- known recent changes
- available owners

## Process

1. Classify severity by revenue, customer impact, and blast radius.
2. Establish current state and timeline.
3. Assign owners for diagnosis, mitigation, comms, and validation.
4. Separate internal causes from provider, issuer, or network causes.
5. Track mitigation and recovery checks.
6. Produce an incident summary and follow-up action list.

## Output

Return:

- severity
- incident summary
- live checklist
- owner map
- internal update draft
- recovery criteria
- post-incident actions
