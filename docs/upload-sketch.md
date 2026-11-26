# Upload sketch (untested outline)

This is an outline of the custom-code step, not production code. Check the current Google Ads API version and field names in the official docs, and test in a sandbox account first.

## Inputs on the contact

- `email`
- `value` (annual: 12 × monthly recurring revenue)
- `conversion_time` (when the lifecycle stage changed)
- `uploaded_at` (empty until sent)

## Steps

1. Normalise the email: trim, lower-case. For Gmail addresses, Google also strips dots and plus-suffixes before hashing, so apply the same.
2. Hash with SHA-256, hex-encoded.
3. Get an access token from your stored refresh token.
4. Call `customers/{customerId}:uploadClickConversions` with `partialFailure: true`.
5. Read the response, handle partial failures per conversion.
6. On success, write the timestamp back to `uploaded_at`.

## Request shape

```json
{
  "conversions": [
    {
      "conversionAction": "customers/{customerId}/conversionActions/{id}",
      "conversionDateTime": "2026-01-15 09:30:00+01:00",
      "conversionValue": 1200.0,
      "currencyCode": "EUR",
      "userIdentifiers": [{ "hashedEmail": "<sha256 hex>" }]
    }
  ],
  "partialFailure": true
}
```

Headers: `Authorization: Bearer <access token>`, `developer-token: <token>`, and `login-customer-id` when you work through a manager account.

## Secrets

Keep the developer token, client secret and refresh token in your CRM's secret store or environment, never in code and never in a repository.

## Gotchas

- Conversions can take up to a day to show up in reports.
- Upload only conversions for leads that came from Google Ads, otherwise you waste quota and inflate noise. If you can't tell the source, rely on the match: unmatched rows simply won't attribute.
- Idempotency: the `uploaded_at` check is your protection against double counting. Don't skip it.
