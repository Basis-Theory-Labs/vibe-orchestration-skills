# Auth Rate Diagnosis

## Purpose

Investigate authorization-rate movement and turn the analysis into likely causes,
next checks, and owner actions.

## When To Use

Use when the user asks why authorization rate changed, where declines are coming
from, or what to do after an auth-rate drop.

## Inputs

- current and baseline authorization rate
- time window
- payment method
- processor or acquirer
- region
- issuer or BIN range when available
- raw decline reasons
- retry and recovery metrics
- launches, incidents, fraud-rule changes, routing changes, or support signals

## Process

1. Quantify the movement and affected segment.
2. Split the data by method, processor, region, and raw decline reason.
3. Separate controllable issues from external issuer or network behavior.
4. Check for correlated product, fraud, routing, or processor changes.
5. Recommend next actions and owners.

## Output

Return:

- short diagnosis
- most likely causes
- data cuts needed next
- controllable vs external factors
- action list with owners
- executive summary
