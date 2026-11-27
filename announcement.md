# Assignment Announcement — System Testing (Wallet-Lite API)

## Deadline
* **First Class after Mid Term Examination**
* **Report Submission: 7th Week (2nd class)**
* **Presentation: 8th Week (1st class)**
* **No extensions will be possible.**

## Presentation
* **Section B, C: TBA**
* **Section A, D: TBA**
    * Each team will give a short in-class presentation of the key defects they found.
    * For each defect, include a screenshot of the evidence (request/response, etc.) in the slides — **one slide per defect is enough**, though a scenario may need more.

## What you'll do

You will perform **system testing** on the **Wallet-Lite API** — a mobile-wallet system with customer, agent, merchant, and admin workflows. It supports wallet transfers, cash-in/out, merchant payments, payment requests, refunds, disputes, and KYC, with JWT authentication and PostgreSQL storage.

* **Before you start:** watch the class videos and tutorials first — they cover the concepts and workflow you'll need.
* **You do not need to run or install the project yourself.** The base URL and other access details are shared in the course email and tutorial.
* **Specifications:** Use the [API References](./api-references.md) and [Business Rules](./business-rules.md) for endpoint contracts and expected behavior.
* **Do not** copy API specifications into your report. Reference the documentation instead.

## Important context

The API may have intentional or unintentional defects. **Your job is to find them, reproduce them, and report them.** Do not fix the code.

Use whichever testing tools work for your team — Postman, REST-assured, custom scripts, or others.

> Quality over quantity. A handful of well-documented, reproducible defects is worth far more than a long list of shallow ones.

> [!CAUTION]
> ## Collaboration & Plagiarism Policy
>
> **This section is strictly enforced. Please read it.**
>
> * You may discuss general approaches with other teams, such as tool setup, test-plan structure, or how JWT works.
> * **Do NOT share** test cases, test data, or specific test ideas with anyone outside your team — not verbally, not in writing, and not in any form.
> * **Do not use AI tools**, the internet, or peers to generate your test cases or defect ideas. Your work must come from your own hands-on analysis of the system.
> * **Do not copy or paraphrase** anything from external sources. Rewriting someone else's test case in your own words is still plagiarism.
> * If your team is caught sharing or receiving specific testing content, **everyone involved will be penalized** — regardless of who started it.
>
> If you're unsure whether something is okay, it probably isn't.

## Deliverables

Grading will be split between **report quality** and **testing coverage/effectiveness** — both matter.

Submit one **Markdown report** and a Postman collection export. Your report must have these sections:

1. **Test Plan**
    * For each endpoint or behavior, document your input choices and expected response, including the HTTP status code.
    * Include a test specification for each function or endpoint.

2. **Test Cases**
    * Use a table or list with: **ID, Short Title, Pre-conditions, Steps, Expected Status Code, Actual Status Code, Verdict**.
    * The **ID** for each test case may be the **last 6 digits** of the response `_lab.requestId`.
    * Cover successful flows, invalid inputs, authentication and authorization, roles and ownership, transaction fees and limits, PIN behavior, idempotency, ledger balances, payment-request states, refunds and reversals, disputes, KYC, filtering, and edge cases.

3. **Defect Reports**
    * For each defect include: ID, Title, Severity, **Steps to Reproduce**, Expected, Actual, and Test Case No.
    * The **defect ID** may be the last 6 digits of the `_lab.requestId` from the response where the defect was observed.
    * Group related defects and flag cascading or duplicate issues where relevant.

4. **Individual Reflections** *(1 page total for the whole team)*
    * Include a short paragraph from each team member describing their personal contribution to the testing effort.

## Evidence

Export your Postman collection, including requests and responses, as a JSON file and submit it alongside your report. No separate screenshots or evidence folder is needed for the report itself.

## Bonus

Build something genuinely useful — a reusable test harness, a scenario runner, a fuzzer, or another tool beyond basic request-and-check scripts — and you may earn bonus credit. Include the repository link in your report.

## Submission

Name your archive exactly as specified in ELMS:
**`<StudentID1>_..._<StudentIDn>_SystemTesting.zip`**

Inside:
* `<StudentID1>_..._<StudentIDn>_SystemTestingReport.md`
* Postman collection export (`<StudentID1>_..._<StudentIDn>_SystemTesting.json`)

## A Few Last Things

* Every defect should be reproducible and backed by evidence.
* Keep your assigned `X-STQA-Key` private. Include it in requests as instructed, and never use another student's key.
* Every response includes a `_lab.requestId`; use it to identify the request in your report.
* When in doubt about expected technical behavior, check the [API References](./documentation/api-references.md) and [Business Rules](./documentation/business-rules.md) first.
