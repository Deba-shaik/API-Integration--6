# Topic 6 - Handling API Responses, Errors, and Rate Limits

**Course:** n8n Hands-On API Integration  
**Submission:** Implementation README and execution evidence  
**Test date:** 30 October 2026  
**Status:** Demonstrated exercises completed; see verification scope below.

## Objective
Implement resilient API integrations in n8n using HTTP request retries, controlled error bypass, page-offset pagination with a bounded loop, one-second request pacing, and a separate Global Error Handler workflow.

## Workflows

### A. API Error Handling
`Manual Trigger -> Call Flaky API`

- Retry scenario: GET `https://postman-echo.com/status/500`; expected HTTP 500, with automatic retry configured. The screenshots confirm the expected HTTP 500 response and automatic-retry indication, but not the exact retry count or delay.
- Continue-on-error scenario: GET `https://jsonplaceholder.typicode.com/invalid-route-9988`; expected HTTP 404. The error output received one item and the workflow continued.

### B. Pagination
`Manual Trigger -> Initialize Page -> Fetch User Page -> Check More Pages`

- True branch: `Increment Page -> Wait 1 Second -> Fetch User Page` (loop back).
- False branch: `Pagination Complete` (No Operation).
- Initialize Page sets numeric `currentPage = 1`.
- Fetch User Page: GET `https://reqres.in/api/users` with query parameter `page = {{ $json.currentPage }}`.
- Check More Pages: `{{ Number($json.page) }}` **is less than** `{{ Math.min(Number($json.total_pages), 5) }}`.
- Increment Page sets numeric `currentPage = {{ Number($json.page) + 1 }}`.
- Wait node resumes after a 1-second interval.
- Observed API response: `total = 12`, `per_page = 6`, `total_pages = 2`. Two Fetch User Page executions succeeded; the second page reached the False branch and Pagination Complete executed with one item.

**Safety bound:** The IF condition caps page traversal at 5 when the API returns valid `page` and `total_pages` metadata. This is a condition-based bound, not a separate node execution limit.

### C. Global Error Handler
`Error Trigger -> Format Error Alert`

- Fields: `failedWorkflowId = {{ $json.workflow.id }}`; `failedWorkflowName = {{ $json.workflow.name }}`; `failedNode = {{ $json.execution.error.nodeName || $json.execution.lastNodeExecuted || 'Unknown Node' }}`; `errorMessage = {{ $json.execution.error.message }}`.
- Published Global Error Handler was linked through Workflow Settings.
- Production test: `Schedule Trigger (every 1 minute) -> Force HTTP 500 Error` using GET `https://postman-echo.com/status/500` with Stop Workflow on error.
- Observed failure: execution ID `549127` in the test workflow. Global Error Handler received execution `549132` and produced the real workflow name `Test Global Error Handler`, failed node `Force HTTP 500 Error`, and message `The service was not able to process your request`.

## Test execution matrix

| ID | Test | Observed result | Outcome |
|---|---|---|---|
| T01 | HTTP 500 retry | HTTP 500 returned; retry enabled | Expected error verified; retry timing/count not independently evidenced |
| T02 | Continue on Fail (404) | Error Branch received 1 item | Pass |
| T03 | Initialize page | Numeric currentPage 1 | Pass |
| T04 | Dynamic API pagination | Two successful requests, pages 1 and 2 | Pass |
| T05 | Bounded loop exit | Page 2 routed to False; Pagination Complete received 1 item | Pass |
| T06 | Request pacing | Wait 1 Second node executed | Pass (configuration/execution; timing not measured) |
| T07 | Error workflow formatting | All four fields populated | Pass |
| T08 | Production failure capture | Real HTTP 500 failure routed to Global Error Handler | Pass |

## Submission contents
- `Topic_6_Handling_API_Responses_README.md`
- `Topic_6_Final_Assessment_Evidence.pdf`
- Exported n8n workflow JSON files (export separately from n8n; not included here)
- Optional screen-recorded demonstration video

## Operational cleanup
**Disable or unpublish the temporary one-minute `Test Global Error Handler` workflow after testing** to prevent continuous intentional failures. Retain the pagination and Global Error Handler workflows. Do not include API credentials or secrets in recordings or exports.

## Evidence limitations
This README and PDF use screenshots shared during the implementation conversation. They do not establish that a separate retry-count measurement, stopwatch timing, notification delivery, or exported JSON validation was performed.
