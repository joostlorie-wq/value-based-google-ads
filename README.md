# value-based-google-ads

How to steer Google Ads on **customer value (recurring revenue)** instead of lead volume, by sending offline conversions from your CRM back to Google Ads.

Worked out for a B2B SaaS company using HubSpot, but the design applies to any CRM that can call a webhook or run custom code.

## The problem

Optimising on form fills rewards cheap, low-value leads. Customer value is very uneven: a small share of customers usually carries most of the recurring revenue. If every lead counts as 1, the bidder can't tell a large account from a small one.

**Goal:** make the bidder weigh a high-value conversion many times heavier than a low-value one.

## Design decisions

### 1. Match on hashed email, not on the click ID

Two ways to link a CRM outcome to an ad click:

| | Click ID (gclid) | Enhanced Conversions for Leads (hashed email) |
|---|---|---|
| Needs the ID stored at form submit | Yes | No |
| Breaks when tracking or consent changes | Often | Rarely |
| Match rate depends on | Capturing the ID reliably | The lead giving the same email they used on Google |

We chose **Enhanced Conversions for Leads** because the click ID storage had already proven fragile in our setup (a tracking break had silently dropped it). Hashed email lives in the CRM anyway.

### 2. Choose the value field carefully

Test what actually predicts value before you bid on it. In our data:

- **Company size bands** were **not** a good value predictor: realised revenue was almost flat across size. Don't bid on size.
- **Value-based customer segments** (bands of recurring revenue) are known only **after** someone becomes a customer. Use them for reporting, not as a bidding input.
- **Recurring revenue per account** is the real value. Use `12 × monthly recurring revenue` as the conversion value.
- An **ICP-fit flag** is a good filter: send conversions only for accounts that match your ideal customer profile, and nothing for the rest.

### 3. Put the value on the same object as the email

The conversion needs email and value together. Revenue lives on the company, the email lives on the contact. So:

1. Sync company-level values down to the contact (new contact fields: value, ICP fit, size band).
2. A contact-level workflow fires the conversion when the contact reaches the right lifecycle stage.

### 4. Two conversion actions, not one

- **Qualified lead** (sales-qualified stage), with an estimated value, for early signal
- **New customer**, with realised value, for ground truth

Upload both so the bidder learns from the early event and gets corrected by the real one.

## Architecture

```
CRM lifecycle change
        │
        ▼
Workflow (contact-level)  ── conditions: ICP fit set, value > 0, not uploaded yet
        │
        ▼
Custom code action  ──►  Google Ads API  uploadClickConversions
                          (hashed email, conversion time, value, currency)
        │
        ▼
Write back "uploaded at" timestamp on the contact  (prevents double uploads)
```

No middleware tool needed if your CRM can run custom code. See [`docs/upload-sketch.md`](docs/upload-sketch.md).

## Rollout order

1. Verify the data model: which field really holds recurring revenue, how many accounts have it filled
2. Create the contact fields and the "uploaded" timestamp fields
3. Backfill history in a one-off batch to give the bidder something to learn from
4. Turn on the workflow for the qualified-lead event, check the Diagnostics tab in Google Ads for match rate
5. Add the new-customer event
6. Switch the campaign bidding to value-based (tROAS or max conversion value) only once enough conversions with value have accumulated

## Measure

- Match rate in Google Ads conversion diagnostics (target: stable and above ~50%)
- Share of conversions with a value attached
- Average realised value per acquired customer, before and after
- Cost per qualified lead stays flat while average value rises: that is the win

## Pitfalls

- A broken base tracking setup will make everything above useless. Audit that first.
- Don't send revenue for non-ICP accounts. It teaches the bidder to find the wrong people.
- Delay between click and customer can be months. Keep the conversion window long enough, and use the earlier qualified-lead event for faster learning.
- Respect consent: only send what the person agreed to for ads purposes.
