# Product Support Engineer – Take-Home Assignment

**Candidate:** Snehitha Myadarapu  
**Date submitted:** 14 November 2026  
**Repository:** https://github.com/snehithadevs/pse-support-lab  
**Total active time:** Not reliably measurable from the available work records. No dedicated timer was used.

## 1. Introduction

This submission covers my approach to investigating customer-reported issues in a product support environment.

I approached the tickets from an L1 Product Support Engineer perspective: understand the customer impact, check the available evidence, identify what can and cannot be confirmed, communicate clearly, and escalate when engineering investigation is required.

I have avoided treating assumptions as confirmed root causes. Where the available evidence was insufficient, I have mentioned the additional checks needed.

## 2. Ticket Prioritisation

| Ticket | Priority | Reason |
|---|---|---|
| API service restarts and 502 errors | P1 – Critical | Production API instability may affect multiple customers and invoice processing |
| Oasis Builders – stuck processing | P1 – High | Repeated retries and an existing document number create a risk of duplicate submission |
| Sahara – submission delays | P1 – High | Rate limiting may have interrupted submissions for multiple invoices |
| Al Noor – invoice validation failures | P2 – High | Some invoices are rejected due to small VAT calculation differences |
| Ghaf – date validation error | P2 – Normal | The reported date format does not match the expected format |

I would review the P1 incidents first because of their potential impact on production processing and customer operations. I would keep the affected customers updated while investigating.

## 3. Investigation 1 – API Service Restarts and 502 Errors

**Priority:** P1 – Critical

### Customer impact

The reported production issue includes repeated API restarts and multiple HTTP 502 errors. Invoice uploads may be affected, and invoices have reportedly remained in the Validated state since approximately 09:30.

This needs immediate attention because customers may be unable to complete their normal invoice submission workflow.

### Investigation performed

I reviewed the available lab evidence, including the container state, restart information and API logs.

The local lab inspection showed:

- Restart count: 2
- OOMKilled: false
- Exit code: 0
- No clear crash or panic message in the inspected API log output

I also reviewed the available synthetic load-test output:

- 217 of 240 requests succeeded.
- 23 requests failed.
- Failures included 15 EOF errors, 5 connection-refused errors, 2 DNS errors and 1 TCP read error.

### Findings

The test results show intermittent request failures in the lab environment. However, they do not establish the root cause of the reported production incident.

The available evidence does not confirm whether the production issue is caused by application behaviour, resource constraints, networking, or an infrastructure dependency.

### Next steps

1. Escalate the incident to the on-call engineering or infrastructure team as a P1.
2. Request production pod and container restart history.
3. Check CPU and memory usage around the reported failure period.
4. Correlate API errors with deployment events and dependency health.
5. Check whether the affected invoice uploads are still pending or have failed.
6. Share progress with the customer at regular intervals.

### Customer communication

We are investigating the API instability and the reported 502 errors. Our team is checking the service health and related logs to understand what is causing the interruptions. We understand that this is affecting invoice processing and have escalated the issue for further investigation. We will share another update as soon as we have confirmed findings.

## 4. Investigation 2 – Al Noor: Invoice Validation Failures

**Priority:** P2 – High

### Customer impact

The customer reported invoice validation failures during an import. The available evidence shows that some invoices were accepted while others were rejected due to a VAT calculation difference.

### Investigation performed

Import ID:

`55b15f82-fa7f-49bc-9fe2-c9aedd1f938f`

The import contained 6 invoices:

- 3 accepted
- 3 rejected

The rejected invoices showed the following differences:

| Invoice | Submitted amount | Expected amount | Difference |
|---|---:|---:|---:|
| 0042 | 15022.42 | 15022.43 | 0.01 |
| 0044 | 11817.57 | 11817.59 | 0.02 |
| 0046 | 5610.99 | 5611.00 | 0.01 |

I reviewed the available code in `services/api/internal/pint/pint.go`. The `round2` calculation and the separate VAT and gross-total calculations are relevant areas to investigate.

### Findings

The small differences suggest a possible rounding inconsistency between line-level VAT calculations and the final gross amount.

This is a likely area to investigate, but I would not confirm it as the root cause without checking the applicable PINT validation rule and the exact calculation path.

### Next steps

1. Share the affected invoice values and import ID with engineering.
2. Ask engineering to verify the calculation against the applicable PINT rule.
3. Confirm whether the rounding approach is consistent across line-level and invoice-level calculations.
4. Ask the customer to avoid repeatedly resubmitting the same invoices until the expected correction is confirmed.
5. Retest with the affected invoice examples after a fix, if required.

No code changes were made as part of this investigation.

### Customer communication

We reviewed the import and found that three invoices were accepted while three were rejected due to small differences in the calculated VAT amounts. We are checking the calculation behaviour with our technical team to confirm the reason for these differences. We will share the recommended correction once the validation behaviour has been confirmed.

## 5. Investigation 3 – Sahara: Submission Delays and HTTP 429 Responses

**Priority:** P1 – High

### Customer impact

The customer reported delays in invoice submissions. The main concern is that some invoices may not have reached the external submission system, even though the application has already recorded them as submitted.

### Investigation performed

Import ID:

`4e36d373-72b1-46b0-866c-eb656f8d7143`

The import contained 40 invoices:

- 8 accepted
- 32 submitted
- 0 rejected
- 0 failed

The available logs showed eight acceptance events, followed by rate-limit warnings and HTTP 429 responses with `Retry-After` information.

I also reviewed `services/api/internal/submit/submit.go`. The available code does not appear to have dedicated handling that captures and applies the `Retry-After` value for HTTP 429 responses.

### Findings

The logs indicate that the external system returned rate-limit responses. This may explain the delayed processing.

There is also a status-verification concern: the application records invoices as Submitted before the external submission outcome is fully confirmed. Therefore, the Submitted status alone should not be treated as proof that the external system received every invoice.

### Next steps

1. Check the external system for the status of the 32 invoices.
2. Compare invoice identifiers and external references to identify which submissions were received.
3. Do not resend the full batch without checking the existing submission status.
4. Retry only invoices confirmed as not received, following the external system's rate-limit guidance.
5. Escalate the HTTP 429 handling and status-update behaviour to engineering.
6. Recommend proper retry handling using the `Retry-After` value, with suitable backoff and duplicate protection.

### Customer communication

We found rate-limit responses from the external submission system during processing. We are checking the status of the affected invoices before taking further action. To avoid duplicate submissions, we will verify which invoices were received and retry only those confirmed as not received.

## 6. Investigation 4 – Oasis Builders: Stuck Processing

**Priority:** P1 – High

### Customer impact

The customer reports that an invoice is stuck during processing. The available ticket information mentions three retries and a message indicating that the document number already exists.

This needs careful handling because another submission could create a duplicate.

### Initial assessment

The existing-document message suggests that the external system may already have received or registered the document. However, this cannot be confirmed from the ticket information alone.

### Next steps

1. Check the import record and current invoice status.
2. Review the processing and retry history.
3. Search for the document number in the external system.
4. Check whether an external reference or acknowledgement was generated.
5. Avoid manually resubmitting the invoice until its existing status is confirmed.
6. Escalate to engineering if the application status does not match the external system's status.

### Customer communication

We are checking the invoice processing history and the document number reported in the error. Since the system indicates that the document number already exists, we want to confirm whether the invoice has already been received before attempting another submission. This will help avoid duplicate processing.

## 7. Investigation 5 – Ghaf: Date Validation Error

**Priority:** P2 – Normal

### Customer impact

The customer is experiencing a validation error related to the invoice date.

### Initial assessment

The provided date is:

`13/11/2026`

The expected format is:

`YYYY-MM-DD`

The equivalent date in the expected format is:

`2026-11-13`

### Next steps

1. Confirm that the invoice date is correct.
2. Ask the customer to use the expected `YYYY-MM-DD` format.
3. Retry the affected invoice after correcting the date.
4. If the error continues, collect the validation message and invoice reference for further investigation.

### Customer communication

The date appears to be in a different format from the one accepted by the system. Please use `2026-11-13` instead of `13/11/2026` and try again. If the same error continues, please share the updated error message so we can investigate further.

## 8. SLA and Escalation Approach

I would use the following response and update targets as an operational guide, while checking the company's actual support policy before communicating formal SLA commitments.

| Priority | Initial response target | Update approach |
|---|---|---|
| P1 – Critical | 15–30 minutes | Every 30–60 minutes |
| P2 – High | Within 1 hour | Every 2–4 business hours |
| P3 – Normal | Within 4 business hours | At least once daily |
| P4 – Low | Within 1 business day | Based on progress and agreed timelines |

### Escalation approach

- P1 incidents affecting production or multiple customers should be escalated immediately to the appropriate on-call team.
- Repeated failures, possible data inconsistencies, and external integration issues should be shared with engineering with supporting evidence.
- Customer updates should continue even when the issue is waiting for another team.
- Escalation does not mean the support ticket is no longer owned by L1. I would continue tracking it until there is a confirmed resolution or handover.

## 9. General L1 Support Approach

For similar incidents, I would follow this process:

1. Understand the issue and confirm the customer's actual impact.
2. Check the affected customer, invoice, import ID and timestamps.
3. Review available application logs and relevant status information.
4. Reproduce the issue in a safe environment when possible.
5. Separate confirmed findings from possible explanations.
6. Avoid risky actions such as duplicate submissions or unverified retries.
7. Escalate with clear evidence and the checks already performed.
8. Keep the customer informed until the issue is resolved.
9. Confirm the final status with the customer and document the resolution.

## 10. Evidence and Limitations

The investigations above are based on the ticket information and the available lab evidence. Local test results should not be treated as proof of production behaviour.

The API, Al Noor and Sahara investigations include technical evidence from the available lab files. The Oasis Builders and Ghaf sections describe the initial assessment and recommended L1 checks based on the ticket information.

I did not make code changes or claim that any production issue was resolved. Any suggested code or retry-handling changes should be reviewed and tested by the engineering team before being applied.

## 11. Conclusion

This exercise helped me practise approaching support issues in a structured way. I focused on understanding customer impact, checking evidence, communicating clearly and knowing when an issue needs engineering involvement.

As an L1 Product Support Engineer, I would aim to resolve issues within my access and knowledge, avoid actions that could affect customer data, and provide clear updates while working with the relevant teams.