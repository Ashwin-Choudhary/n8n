# Failure scenarios for HVAC V3.1

A workflow is ready when every branch has been forced to run, not when the normal path works.
This is the list of scenarios `HVAC V3.1 - Recovery Testing.json` is checked against. Each one
says what to break, and what must happen to the webhook caller, the team and the claim row.

To force a failure in your own instance, point the named node at a wrong sheet / wrong
credential, or pin an error on it, then send the lead.

| #   | Scenario                                                       | Webhook answer       | What must happen                                                                |
| --- | -------------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------- |
| 1   | Normal warm lead                                               | `200` success        | Customer email, sheet row, claim `completed` / `sent`                           |
| 2   | Name and email missing                                         | `400` rejected       | Body lists `missing_fields`; no claim                                           |
| 3   | Same `submission_id` delivered twice                           | `200` duplicate      | Second delivery does nothing                                                    |
| 4   | Sheet already has this submission                              | `200` duplicate      | Claim `failed` / `duplicate_rejected`                                           |
| 5   | Same as 4, lead has no `submission_id`                         | `200` duplicate      | Claim (fingerprint key) `failed` / `duplicate_rejected`                         |
| 6   | `Check Existing Lead` fails                                    | `200` success        | Lead still processed; "Duplicate check skipped" email to the team               |
| 7   | `AI Classification` fails, then the lead is sent again         | `500`, then `200`    | First: failure row + error email, claim `failed` / `pending`. Second: processed |
| 8   | Same as 7, lead has no `submission_id`                         | `500`, then `200`    | Retry is processed, not rejected as a duplicate of its own failure row          |
| 9   | AI fails and the claim cannot be marked retryable              | `500`                | "AI recovery needs manual review" email; no failure row written as if retryable |
| 10  | AI fails and `Log New Lead` fails                              | `500`                | One error email, saying the lead was **not** saved                              |
| 11  | Gas leak message                                               | `200` success        | Team safety alert, customer safety email, safety sheet row                      |
| 12  | Gas leak, `Send Safety Email` fails                            | `500`                | Team safety alert still sent; error email; sheet row                            |
| 13  | Gas leak, `AI Classification` fails                            | `200` success        | Same as 11 (the keyword check is enough)                                        |
| 14  | Gas leak, `Log Safety Lead` fails                              | `200` success        | Team safety alert and customer email still sent; error email                    |
| 15  | AI asks for human review                                       | `202` pending review | Review alert, review sheet row                                                  |
| 16  | Human review, `Send Human Review Alert` fails                  | `202` pending review | Review sheet row still written; error email                                     |
| 17  | `Save Raw Lead` fails                                          | `200` success        | Lead still processed; "Raw Lead Storage Failed" email with the real reference   |
| 18  | `Send Lead Response` fails                                     | `500`                | Claim `failed` / `uncertain`; tracking-failure email                            |
| 19  | Sheet log fails after the customer email, then lead sent again | `500`, then `202`    | Claim `failed` / `sent`; the retry goes to manual review, no second email       |
| 20  | Same as 19, `Claim Recovery Manual Review Alert` fails         | `500`, then `202`    | Caller still gets the manual-review answer                                      |
| 21  | Same customer, different request within 12 hours               | `202` pending review | Possible-duplicate sheet row and alert                                          |

`500` means "received, but a person has to look at it". The caller may send the lead again;
the claim decides whether that retry is processed, sent to manual review or dropped as a
duplicate.

## Still open

- Claims for safety, human-review and possible-duplicate leads stay `claimed` / `pending`. They
  look the same as a lead whose execution died half way.
- The human-review branch prepares a customer reply (`response`) but no node sends it.
- Every handled failure finishes as a successful execution, so the error workflow does not fire
  and the executions list stays green. The alert emails are the only signal.
