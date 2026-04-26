# Issue Management Reference

Patterns for creating well-structured GitHub issues: templates, labels, projects, and milestones.

---

## Issue Templates

Store templates under `.github/ISSUE_TEMPLATE/`. GitHub picks them up automatically.

### Directory Layout

```
.github/
└── ISSUE_TEMPLATE/
    ├── config.yml          ← controls the template chooser
    ├── bug_report.yml
    ├── feature_request.yml
    ├── task.yml
    └── documentation.yml
```

### `config.yml` — Template Chooser

```yaml
blank_issues_enabled: false   # force template selection
contact_links:
  - name: 💬 Discussions
    url: https://github.com/ORG/REPO/discussions
    about: Ask questions or share ideas here first.
```

---

### Bug Report — `bug_report.yml`

```yaml
name: 🐛 Bug Report
description: Report a reproducible defect.
title: "fix: <short description>"
labels: ["bug", "needs-triage"]
projects: ["ORG/PROJECT_NUMBER"]   # optional default project
assignees: []
body:
  - type: markdown
    attributes:
      value: |
        Thanks for taking the time to report a bug. Fill in as much detail as possible.

  - type: textarea
    id: description
    attributes:
      label: Description
      description: A clear summary of what went wrong.
    validations:
      required: true

  - type: textarea
    id: steps
    attributes:
      label: Steps to Reproduce
      placeholder: |
        1. Go to …
        2. Click on …
        3. See error
    validations:
      required: true

  - type: textarea
    id: expected
    attributes:
      label: Expected Behaviour
    validations:
      required: true

  - type: textarea
    id: actual
    attributes:
      label: Actual Behaviour
    validations:
      required: true

  - type: textarea
    id: environment
    attributes:
      label: Environment
      placeholder: |
        - OS: macOS 14 / Ubuntu 22.04 / Windows 11
        - Node: 20.x
        - App version / commit SHA:
    validations:
      required: false

  - type: textarea
    id: logs
    attributes:
      label: Logs / Screenshots
      description: Paste relevant output. Use code blocks.
    validations:
      required: false

  - type: dropdown
    id: severity
    attributes:
      label: Severity
      options:
        - "🔴 Critical — data loss or security issue"
        - "🟠 High — major feature broken"
        - "🟡 Medium — partial degradation"
        - "🟢 Low — cosmetic or minor"
    validations:
      required: true
```

---

### Feature Request — `feature_request.yml`

```yaml
name: ✨ Feature Request
description: Propose a new feature or enhancement.
title: "feat: <short description>"
labels: ["enhancement", "needs-triage"]
body:
  - type: textarea
    id: problem
    attributes:
      label: Problem / Motivation
      description: What problem does this solve? Who is affected?
    validations:
      required: true

  - type: textarea
    id: solution
    attributes:
      label: Proposed Solution
    validations:
      required: true

  - type: textarea
    id: alternatives
    attributes:
      label: Alternatives Considered
    validations:
      required: false

  - type: textarea
    id: acceptance
    attributes:
      label: Acceptance Criteria
      placeholder: |
        - [ ] …
        - [ ] …
    validations:
      required: false
```

---

### Task — `task.yml`

```yaml
name: 📋 Task
description: General engineering or chore work.
title: "chore: <short description>"
labels: ["task"]
body:
  - type: textarea
    id: objective
    attributes:
      label: Objective
    validations:
      required: true

  - type: textarea
    id: subtasks
    attributes:
      label: Sub-tasks
      placeholder: |
        - [ ] …
        - [ ] …
    validations:
      required: false

  - type: textarea
    id: notes
    attributes:
      label: Notes / Context
    validations:
      required: false
```

---

### Documentation — `documentation.yml`

```yaml
name: 📝 Documentation
description: Request or report a documentation issue.
title: "docs: <short description>"
labels: ["documentation"]
body:
  - type: dropdown
    id: type
    attributes:
      label: Type
      options:
        - Missing documentation
        - Incorrect / outdated documentation
        - Improve clarity
    validations:
      required: true

  - type: input
    id: location
    attributes:
      label: File or URL
      placeholder: "docs/api.md or https://…"
    validations:
      required: false

  - type: textarea
    id: details
    attributes:
      label: Details
    validations:
      required: true
```

---

## Labels

### Creating Labels with `gh`

```bash
# Create a single label
gh label create "bug" --color "d73a4a" --description "Confirmed defect"

# Bulk-create from a JSON file (see schema below)
gh label create --from-file .github/labels.json
```

### Recommended Label Schema — `.github/labels.json`

```json
[
  { "name": "bug",              "color": "d73a4a", "description": "Confirmed defect" },
  { "name": "enhancement",      "color": "a2eeef", "description": "New feature or improvement" },
  { "name": "documentation",    "color": "0075ca", "description": "Docs additions or fixes" },
  { "name": "task",             "color": "e4e669", "description": "General engineering work" },
  { "name": "needs-triage",     "color": "ededed", "description": "Awaiting classification" },
  { "name": "in-progress",      "color": "fbca04", "description": "Actively being worked on" },
  { "name": "blocked",          "color": "e11d48", "description": "Cannot proceed; dependency exists" },
  { "name": "ready-for-review", "color": "0e8a16", "description": "Work done; awaiting review" },
  { "name": "wontfix",          "color": "ffffff", "description": "Intentionally not addressed" },
  { "name": "duplicate",        "color": "cfd3d7", "description": "Already reported elsewhere" },
  { "name": "good first issue", "color": "7057ff", "description": "Suitable for new contributors" },
  { "name": "help wanted",      "color": "008672", "description": "Extra attention needed" },
  { "name": "security",         "color": "b60205", "description": "Security-related concern" },
  { "name": "performance",      "color": "f9d0c4", "description": "Speed or resource usage" },
  { "name": "tech-debt",        "color": "bfd4f2", "description": "Refactor or clean-up work" },
  { "name": "breaking-change",  "color": "e99695", "description": "Introduces a breaking change" }
]
```

### Sync Labels Across Repos

```bash
# Copy labels from one repo to another (requires gh CLI + jq)
gh label list --repo ORG/SOURCE_REPO --json name,color,description \
  | jq 'map(del(.id))' \
  | gh label create --repo ORG/TARGET_REPO --from-file /dev/stdin
```

### Label Hygiene Rules

- Every issue must have **at least one type label** (`bug`, `enhancement`, `task`, `documentation`).
- Apply `needs-triage` on creation; replace it once classified.
- Do not stack conflicting state labels (`in-progress` + `blocked` is fine; `in-progress` + `wontfix` is not).
- Use `breaking-change` on any issue whose resolution will require a major version bump.

---

## Projects (GitHub Projects v2)

### Link an Issue to a Project on Creation

```bash
# Create issue and immediately add to a project
ISSUE_URL=$(gh issue create \
  --title "fix: crash on empty cart" \
  --body "Reproduces with zero items at checkout." \
  --label "bug,needs-triage" \
  --assignee "@me" \
  --repo ORG/REPO)

gh project item-add PROJECT_NUMBER \
  --owner ORG \
  --url "$ISSUE_URL"
```

### Set a Custom Field Value

```bash
# Get the item ID after adding to the project
ITEM_ID=$(gh project item-list PROJECT_NUMBER --owner ORG --format json \
  | jq -r '.items[] | select(.content.url == "'"$ISSUE_URL"'") | .id')

# Set a single-select field, e.g. "Priority" = "High"
gh project item-edit \
  --project-id PROJECT_NUMBER \
  --id "$ITEM_ID" \
  --field-id FIELD_ID \
  --single-select-option-id OPTION_ID
```

> **Tip:** Retrieve `FIELD_ID` and `OPTION_ID` once with:
> ```bash
> gh project field-list PROJECT_NUMBER --owner ORG --format json
> ```

### Automate via Issue Template

Add `projects` to the template front matter to automatically link on issue creation (requires GitHub Actions or project automation rules; native template support is org-level only):

```yaml
# In bug_report.yml
projects: ["ORG/5"]   # project number 5
```

### Workflow: Triage → In Progress → Done

| Column / Status | Label to apply       | Trigger                        |
| --------------- | -------------------- | ------------------------------ |
| Triage          | `needs-triage`       | Issue opened                   |
| Backlog         | *(type label only)*  | After triage, not yet started  |
| In Progress     | `in-progress`        | Branch created / PR opened     |
| Blocked         | `blocked`            | Dependency identified          |
| In Review       | `ready-for-review`   | PR ready                       |
| Done            | *(label removed)*    | Issue closed via PR merge      |

---

## Milestones

### Create a Milestone

```bash
gh api repos/ORG/REPO/milestones \
  --method POST \
  --field title="v1.2.0" \
  --field description="Scope: payment flow rewrite and accessibility pass." \
  --field due_on="2025-09-30T00:00:00Z"
```

### Assign an Issue to a Milestone on Creation

```bash
# Get milestone number first
MILESTONE=$(gh api repos/ORG/REPO/milestones \
  --jq '.[] | select(.title == "v1.2.0") | .number')

gh issue create \
  --title "feat: add Apple Pay support" \
  --label "enhancement" \
  --milestone "$MILESTONE"
```

### Milestone Hygiene Rules

- One milestone per planned release version (e.g., `v1.2.0`, `v2.0.0-beta`).
- Set a realistic `due_on`; leave blank only for unscheduled / icebox milestones.
- Close a milestone only when **all issues are closed or explicitly moved out**.
- Do not reopen a closed milestone; create a patch milestone (e.g., `v1.2.1`) instead.
- Review milestone progress weekly:

```bash
gh api repos/ORG/REPO/milestones --jq \
  '.[] | {title, open_issues, closed_issues, due_on, state}'
```

### Close a Milestone

```bash
MILESTONE_NUM=$(gh api repos/ORG/REPO/milestones \
  --jq '.[] | select(.title == "v1.2.0") | .number')

gh api repos/ORG/REPO/milestones/$MILESTONE_NUM \
  --method PATCH \
  --field state="closed"
```

---

## Full `gh issue create` Reference

```bash
gh issue create \
  --title "fix: <description>" \          # follows Conventional Commits style
  --body-file .github/ISSUE_TEMPLATE/bug_report.md \
  --label "bug,needs-triage" \            # comma-separated, must exist first
  --assignee "username" \                 # or @me
  --milestone "v1.2.0" \                  # milestone title or number
  --project "ORG/PROJECT_NUMBER" \        # optional; links to Projects v2
  --repo ORG/REPO
```

### Quick Inline Issue (no template)

```bash
gh issue create \
  --title "chore: update Node to 22 LTS" \
  --body "- [ ] Update .nvmrc\n- [ ] Update Dockerfile\n- [ ] Update CI matrix" \
  --label "task" \
  --milestone "v1.3.0"
```

---

## Checklist: Well-Formed Issue

- [ ] Title follows `<type>: <imperative description>` (Conventional Commits style)
- [ ] At least one **type label** assigned (`bug`, `enhancement`, `task`, `documentation`)
- [ ] `needs-triage` applied if not yet classified
- [ ] Milestone set (or explicitly left blank for icebox items)
- [ ] Linked to the active project board
- [ ] Assignee set (or left blank if un-owned)
- [ ] Acceptance criteria or sub-tasks listed when scope is non-trivial
