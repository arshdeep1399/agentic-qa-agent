# Example spec (test fixture)

An original, minimal spec written in the same Markdown shape the agent expects. Used only by the
unit tests so they run without the hackathon's own PRD.

## Purpose

This app is a practice target. It must never move real money.

## Users and access

- One fake customer exists for the demo.
- A visible button lets the visitor start the demo.

## Transfers between owned accounts

- Money can move between the two accounts.
- The form asks for source, destination and amount.
- Moving money into the same account is refused.
- Amounts must be above zero with at most two decimals.
- If funds are insufficient, the transfer is refused and nothing changes.

## Transaction history

- A list shows every transaction.
- Each row shows date, amount and status.
- Filters narrow the list.
- The start and end dates are inclusive.

## General interface behavior

- Pages work on a laptop screen.
- Controls have readable labels.
- Loading states are visible.
- Links go where their label says.
- Errors offer a way to retry.

## Seed data expectations

The demo begins with two accounts. Records should cover several dates.

## Out of scope

- Anything involving real banks.
