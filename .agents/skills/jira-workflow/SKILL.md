---
name: jira-workflow
description: Pull issues from Jira via jira-cli, coordinate parent tickets and sub-tasks, and record verified implementation evidence.
metadata:
  internal: true
---

# Jira Workflow Skill

Use this skill when work is tracked by a Jira parent ticket used as <TICKET_NUMBER>. The captain will relay the ticket number.

## Firstmate Responsibilities

Firstmate owns the parent ticket from intake through closure. Everything starts with the parent Jira ticket number and follows this process.

1. Fetch the parent ticket and its acceptance criteria. Treat a missing or empty values as an intake blocker and report to the captain that execution cannot continute.
   - Use `jira issue view <TICKET_NUMBER>  --plain` for the rendered desciption and acceptance criteria.

2. Move the parent from `To Do` to `In Progress` upon successful intake using `jira issue move $KEY "In Progress"`

3. Build the crewmate instructions from the parent ticket summary and the retrieved acceptance criteria. Convert each acceptance criterion into an explicit verification item and require evidence for every item before the worker reports completion.

4. Ask the worker for an implementation breakdown before authorizing code changes. The worker may inspect the repository and propose logical sub-task boundaries, but it must not edit code, tests, documentation, or configuration at this stage.
   Explicitly instruct the worker that implementation is authorized only after the sub-tasks exist in Jira. The worker must not create Jira issues or use jira-cli.

5. Firstmate creates the proposed Jira sub-tasks from the worker implementation breakdown and records their acceptance criteria before authorizing implementation after getting report from the crewmate.
   - Use `jira issue create -P <TICKET_NUMBER> -t Subtask -s "TASK HERE" -b "DETAILS HERE" --no-input` to create the sub-tasks

6. All crewmate acceptance criteria evidence and relevant test results should be written to `/tmp/<TICKET NUMBER>` 

7. Spawn and supervise the authorized implementation work instructing the crewmate to provide an implementation summary report back to Firstmate after complete. The crewmate should provide links to evidence stashed in `/tmp/<TICKET NUMBER>`

8. Review the returned implementation, test results, PR or MR, and test evidence. Do not accept a worker's completion report as validation by itself. Compare its evidence against every acceptance criterion.

9. Move the parent from `In Progress` to `In Review` after implementation and the review is completed by Firstmate
   - Use `jira issue move $KEY "In Review"`

10. Makes comments on the parent ticket using `jira issue comment add <TICKET_NUMBER> "COMMENT HERE" --no-input` with the following bulleted list
   - the overarching work completed
   - the full PR or MR URL
   - a concise evidence matrix mapping every acceptance criterion to the relevant sub-task evidence
   - an explanation of how each cited item demonstrates that criterion is covered
   - limitations, exclusions, or remaining gaps

11. Attach all relevant evidence written from the crewmates directly to the parent ticket. Be mindful of what pieces of evidence satisy the acceptance criteria - don't just dump everything without thinking.
    - The installed Jira CLI has no attachment command. Upload logs, screenshots, and other evidence files through the Jira REST API
    - Use `POST <JIRA_SERVER>/rest/api/2/issue/<ISSUE-KEY>/attachments`
    - Use curl Basic authentication with `-u "$JIRA_LOGIN:$JIRA_API_TOKEN"`, plus `X-Atlassian-Token: no-check` and multipart field `file=@<PATH>`; `JIRA_API_TOKEN` is the raw token and must not be placed directly in an `Authorization: Basic` header.
    - Verify the response includes the attachment id, filename, and content URL before claiming the upload succeeded.

12. For now we will leave the closure of the parent ticket and the movement to `Done` by the captain

## Crewmate Responsibilities

The crewmate owns reporting on the work that needs to get done, the implementation of the work when authorized, and the testing required to achieve the acceptance criteria.

1. When receiving the implementation breakdown orders from Firstmate, inspect the repository and propose the logical implementation pieces

2. Do not create Jira issues, call jira-cli, or perform any code, test, documentation, or configuration edits while proposing the implementation breakdown

3. Wait for Firstmate to create the Jira sub-tasks and explicitly authorize implementation before conducting the implementation

4. After authorization conduct the implementation and run the required tests and checks

5. Record implementation details, test results, application verification, evidence, and limitations in the implementaion summary report for Firstmate
   - All crewmate acceptance criteria evidence and relevant test results (logs, screenshots, etc.) should be written to `/tmp/<TICKET NUMBER>` and linked in the the implementation summary report

## Failure Handling

If Jira issue creation, transition, commenting, or evidence upload fails, report the exact blocker and preserve the implementation evidence.
Never silently create an incomplete sub-task or report a transition as complete without verifying Jira accepted it.
