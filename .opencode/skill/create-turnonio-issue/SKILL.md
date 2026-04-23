---
name: create-turnonio-issue
description: Create a GitHub issue in Turnonio/Issue-Tracker, link it to the Turnonio Roadmap project (org project #1), and set its issue type. REQUIRED inputs - title, body, and issue type (one of Feature, Task, Bug). Optional - Status (default Backlog), Iteration (default "Not scheduled"). Triggers - "create issue", "new ticket", "log a feature/bug/task", any issue-creation request for the Turnonio issue tracker.
---

# Create Turnonio Issue Skill

End-to-end workflow to create a GitHub issue in `Turnonio/Issue-Tracker`, add it to the "Turnonio Roadmap" project (org project #1), and set its issue type + initial project fields.

## Required inputs

- **title** (string) - concise, descriptive issue title. If missing, ASK the user.
- **body** (string) - issue description / context. If missing, ASK the user. Accept Markdown.
- **issueType** (enum): `Feature` | `Task` | `Bug`
  - If the user did not specify one, ASK before proceeding. Do not guess.

## Optional inputs (with defaults)

- **status** (enum) - default: `Backlog`. Options: `Backlog`, `Planned`, `On hold`, `In Progress`, `Done`
- **iteration** (string) - default: `Not scheduled`. Other current options: `Q2 - 2026`, `Q3 - 2026`, `Q4 - 2026`

## Hardcoded constants

These values are stable for this repo and can be used directly. Only re-query if a mutation returns "not found".

```
REPO_OWNER      = Turnonio
REPO_NAME       = Issue-Tracker
PROJECT_NUMBER  = 1
PROJECT_ID      = PVT_kwDOA9clQc4A0S4z     # Turnonio Roadmap

# Project fields
STATUS_FIELD_ID     = PVTSSF_lADOA9clQc4A0S4zzgp81cw
ITERATION_FIELD_ID  = PVTIF_lADOA9clQc4A0S4zzgp9pjE

# Status options
STATUS_BACKLOG       = f75ad846
STATUS_PLANNED       = 3133b182
STATUS_ON_HOLD       = f1f951f0
STATUS_IN_PROGRESS   = 47fc9ee4
STATUS_DONE          = 98236657

# Iteration IDs (refetch if "Not scheduled" ID stops working - iteration IDs rotate over time)
ITERATION_NOT_SCHEDULED = 1c080095
ITERATION_Q2_2026       = 87682184
ITERATION_Q3_2026       = eeefc020
ITERATION_Q4_2026       = 31a583b4

# Issue type IDs (org-level, stable)
ISSUE_TYPE_TASK    = IT_kwDOA9clQc4Au0l_
ISSUE_TYPE_BUG     = IT_kwDOA9clQc4Au0mD
ISSUE_TYPE_FEATURE = IT_kwDOA9clQc4Au0mG
```

## Auth requirement

`gh auth status` must show scopes including `repo` and `project`. `read:org` is NOT required (we avoid `gh project list` by using the stored project node ID).

## Execution steps

Run steps in order. After each step, verify success before continuing.

### Step 1 - Create the issue

```bash
gh issue create \
  --repo Turnonio/Issue-Tracker \
  --title "<TITLE>" \
  --body "<BODY>"
```

Capture the returned URL and extract the issue number (e.g. `.../issues/94` → `94`).

### Step 2 - Get the issue node ID

```bash
gh api graphql -f query='
  query($num:Int!){
    repository(owner:"Turnonio",name:"Issue-Tracker"){
      issue(number:$num){ id }
    }
  }' -F num=<ISSUE_NUMBER>
```

Save as `ISSUE_NODE_ID`.

### Step 3 - Add issue to project

```bash
gh api graphql -f query='
  mutation($projectId:ID!,$contentId:ID!){
    addProjectV2ItemById(input:{projectId:$projectId,contentId:$contentId}){
      item{ id }
    }
  }' -F projectId=PVT_kwDOA9clQc4A0S4z -F contentId=<ISSUE_NODE_ID>
```

Save returned `item.id` as `ITEM_ID`.

### Step 4 - Set Status (default: Backlog)

```bash
gh api graphql -f query='
  mutation($p:ID!,$i:ID!,$f:ID!,$o:String!){
    updateProjectV2ItemFieldValue(input:{
      projectId:$p,itemId:$i,fieldId:$f,
      value:{singleSelectOptionId:$o}
    }){ projectV2Item{ id } }
  }' \
  -F p=PVT_kwDOA9clQc4A0S4z \
  -F i=<ITEM_ID> \
  -F f=PVTSSF_lADOA9clQc4A0S4zzgp81cw \
  -F o=<STATUS_OPTION_ID>
```

### Step 5 - Set Iteration (default: Not scheduled)

```bash
gh api graphql -f query='
  mutation($p:ID!,$i:ID!,$f:ID!,$it:String!){
    updateProjectV2ItemFieldValue(input:{
      projectId:$p,itemId:$i,fieldId:$f,
      value:{iterationId:$it}
    }){ projectV2Item{ id } }
  }' \
  -F p=PVT_kwDOA9clQc4A0S4z \
  -F i=<ITEM_ID> \
  -F f=PVTIF_lADOA9clQc4A0S4zzgp9pjE \
  -F it=<ITERATION_ID>
```

> If this returns a "not found" error, iteration IDs have rotated. Refetch them:
> ```bash
> gh api graphql -f query='query{node(id:"PVT_kwDOA9clQc4A0S4z"){... on ProjectV2{field(name:"Iteration"){... on ProjectV2IterationField{configuration{iterations{id title}}}}}}}'
> ```

### Step 6 - Set Issue Type (REQUIRED)

```bash
gh api graphql -f query='
  mutation($issueId:ID!,$typeId:ID!){
    updateIssueIssueType(input:{issueId:$issueId,issueTypeId:$typeId}){
      issue{ number issueType{ name } }
    }
  }' -F issueId=<ISSUE_NODE_ID> -F typeId=<ISSUE_TYPE_ID>
```

Use the ID that matches the input:
- `Feature` → `IT_kwDOA9clQc4Au0mG`
- `Task`    → `IT_kwDOA9clQc4Au0l_`
- `Bug`     → `IT_kwDOA9clQc4Au0mD`

## Final report

After all steps succeed, report to the user:

- Issue URL
- Issue number
- Issue type set
- Project: Turnonio Roadmap
- Status + Iteration set

## Error recovery

- **"not found" on project/field/option ID** → IDs rotated. Re-query fields via:
  ```bash
  gh api graphql -f query='query{node(id:"PVT_kwDOA9clQc4A0S4z"){... on ProjectV2{fields(first:50){nodes{... on ProjectV2SingleSelectField{id name options{id name}} ... on ProjectV2IterationField{id name configuration{iterations{id title}}}}}}}}'
  ```
- **Issue types query forbidden (missing `read:org`)** → use the hardcoded IDs above, or fetch via repo:
  ```bash
  gh api graphql -H "GraphQL-Features: issue_types" -f query='query{repository(owner:"Turnonio",name:"Issue-Tracker"){issueTypes(first:20){nodes{id name}}}}'
  ```
- **Issue created but later step fails** → do NOT recreate. Resume from the failed step using the existing issue/item IDs.

## Anti-patterns

- Do NOT use `gh issue create --project ...` — it requires `read:org` and fails with "not found" on org projects for this token.
- Do NOT skip the issue type step — it is a required input for this skill.
- Do NOT delete and recreate the issue if a project mutation fails — fix and resume.
