# Specification: Team Tab in Tower Right Panel

## Metadata
- **ID**: spec-587
- **Status**: draft
- **Created**: 2026-03-06
- **GitHub Issue**: #587

## Clarifying Questions Asked

1. **What data should appear per team member?** — Name, GitHub handle, role, assigned issues, open PRs, recent activity (PR merges, issue closes).
2. **Where do team member definitions live?** — New `codev/team/` directory with one `.md` file per member, using YAML frontmatter for structured data (name, github, role).
3. **Is this read-only or interactive?** — Read-only for v1 (GitHub data and messages). Future work includes richer inter-architect messaging.
4. **How fresh should the data be?** — Fetch-on-activation pattern (like Analytics tab), not polling. Manual refresh button for explicit re-fetch.
5. **Should it work without `codev/team/` files?** — Yes, show an empty state with instructions on how to add team members.
6. **What about team communication?** — A `codev/team/messages.md` append-only log allows team members to post timestamped messages. These are displayed in the Team tab.

## Problem Statement

The Tower dashboard currently has no visibility into team composition or what other team members are working on. In a multi-architect setup, each architect works in isolation with no shared view of assignments, activity, or availability. This makes coordination difficult and leads to duplicate work or blocked handoffs.

## Current State

- Tower has two main content tabs: **Work** (builders, PRs, backlog) and **Analytics** (metrics, charts)
- No concept of "team" exists in the dashboard or data model
- Team coordination happens outside the tool (Slack, meetings, etc.)
- No `codev/team/` directory convention exists

## Desired State

- A new **Team** tab in the Tower dashboard (main tab area alongside Work and Analytics)
- Each team member defined in a `codev/team/<github-handle>.md` file with YAML frontmatter
- The tab displays per-member: assigned issues, open PRs, and recent GitHub activity
- A `codev/team/messages.md` append-only message log displayed in the Team tab for team communication
- Empty state guides users to create team member files
- Foundation for richer inter-architect messaging in the future

## Stakeholders
- **Primary Users**: Architects using Tower to coordinate work
- **Secondary Users**: Team leads reviewing workload distribution
- **Technical Team**: Codev maintainers

## Success Criteria
- [ ] New `codev/team/` directory convention with documented file format
- [ ] Team tab appears in Tower dashboard tab bar alongside Work and Analytics (always visible, shows empty state when no team files)
- [ ] Tab loads team member files and displays parsed frontmatter (name, role, GitHub handle)
- [ ] Tab fetches and displays per-member GitHub data (assigned issues, open PRs, recent activity)
- [ ] `codev/team/messages.md` append-only message log parsed and displayed in Team tab
- [ ] Empty state shown when no team files exist, with guidance text
- [ ] Manual refresh button works
- [ ] Tab follows existing UI patterns (styling, layout, responsive behavior)
- [ ] All new code has test coverage >90% (server, hooks, and UI layers)
- [ ] No regression in existing Tower functionality

## Constraints

### Technical Constraints
- Must follow existing tab registration pattern in `useTabs.ts` (add `'team'` to type union)
- Must follow existing data fetching pattern (custom hook with fetch-on-activation for `/api/team` endpoint, like `useAnalytics`)
- Must use existing CSS variable theming system
- GitHub API calls must use the existing `gh` CLI or Octokit patterns already in the codebase
- Team member files use YAML frontmatter parsed with the same library used elsewhere (gray-matter or similar)

### Business Constraints
- Read-only in v1 — no editing team files or messages from the UI (messages are appended via git)
- No real-time presence or online status — just GitHub activity data and message log
- Richer inter-architect messaging (e.g., inline replies, notifications) is out of scope for v1

## Assumptions
- The `gh` CLI is available and authenticated in the environment (same assumption as existing GitHub integrations)
- Team member `.md` files are committed to the repo and available in the worktree
- GitHub handles in team files are valid and correspond to real GitHub users

## Solution Approaches

### Approach 1: File-Based Team Directory with GitHub Integration (Recommended)

**Description**: Each team member gets a `codev/team/<handle>.md` file with YAML frontmatter. The Tower backend reads these files, enriches with GitHub API data, and serves via a new `/api/team` endpoint. The frontend renders a new Team tab.

**Team member file format** (`codev/team/<github-handle>.md`):
```yaml
---
name: Waleed Khan
github: wkhan
role: architect
---

Optional freeform notes (not rendered in the dashboard UI — for repo readers only).
```

Required frontmatter fields: `name`, `github`. Optional: `role` (defaults to "member").

**Message log format** (`codev/team/messages.md`):
```markdown
# Team Messages

<!-- Append new messages below. Do not edit or delete existing entries. -->

---
**wkhan** | 2026-03-06 14:30 UTC
Starting work on the auth refactor. Will need the staging env by EOD.

---
**jdoe** | 2026-03-06 15:12 UTC
Staging env is ready. Credentials in 1Password.
```

Each message is a block separated by `---`, with a header line of `**<github-handle>** | <ISO-ish timestamp>` followed by the message body. Simple, human-readable, git-friendly.

**GitHub data semantics** (per member, scoped to current repo):
- **Assigned issues**: `assignee:<handle> is:issue is:open` in the current repo
- **Open PRs**: `author:<handle> is:pr is:open` in the current repo
- **Recent activity** (last 7 days): PRs merged (`author:<handle> is:pr is:merged merged:>YYYY-MM-DD`) and issues closed (`assignee:<handle> is:issue is:closed closed:>YYYY-MM-DD`). Commits are excluded (too noisy, available via PR context).

For performance, use a single batched GraphQL query via `gh api graphql` (see existing pattern in `src/lib/github.ts` `fetchOnItTimestamps`). All member queries run in a single request to stay within the 2s target.

**Pros**:
- Simple, version-controlled team definition
- Follows existing codev conventions (YAML frontmatter in markdown)
- Easy to add/remove team members
- Git history tracks team changes

**Cons**:
- Requires manual file creation per member
- GitHub API rate limits could affect large teams

**Estimated Complexity**: Medium
**Risk Level**: Low

### Approach 2: GitHub-Only Discovery

**Description**: Discover team members automatically from GitHub repo collaborators/contributors. No local files needed.

**Pros**:
- Zero configuration
- Always up to date

**Cons**:
- No control over who appears (all contributors shown)
- No place for custom metadata (role, notes)
- Dependent entirely on GitHub API availability
- Can't represent team members who haven't contributed yet

**Estimated Complexity**: Low
**Risk Level**: Medium (less control, API dependency)

### Recommended Approach

**Approach 1** — File-based team directory. It's explicit, version-controlled, and provides a foundation for future features (messaging, role-based views). The file format is trivially simple and consistent with how codev already works.

## Open Questions

### Critical (Blocks Progress)
- [x] File format for team member definitions — **Resolved**: YAML frontmatter in `.md` files
- [x] Message log format — **Resolved**: Append-only `codev/team/messages.md` with `---`-separated entries

### Important (Affects Design)
- [x] Should the tab be persistent or lazy? — **Resolved**: Fetch-on-activation (like Analytics), not polling. Only fetch when tab becomes active.
- [x] What GitHub activity counts as "recent"? — **Resolved**: Last 7 days of PRs merged and issues closed (no commits)
- [x] What "assigned issues" and "open PRs" mean exactly — **Resolved**: Assignee for issues, author for PRs, scoped to current repo only
- [x] Should the markdown body of team files be rendered? — **Resolved**: No. Body is for repo readers only; dashboard shows frontmatter data + GitHub activity

### Nice-to-Know (Optimization)
- [ ] Should team data be cached server-side to reduce GitHub API calls? — Defer to implementation; start with batched GraphQL, add caching if needed

## Performance Requirements
- **Tab Load Time**: <2s for teams of up to 10 members (achieved via single batched GraphQL query)
- **GitHub API**: Single batched GraphQL request for all members; graceful degradation if API unavailable
- **Data Fetching**: Fetch-on-activation only (not polling). Re-fetch on manual refresh button click.

## Security Considerations
- No new authentication — uses existing `gh` CLI auth
- Team files contain only public GitHub handles and names — no secrets
- API endpoint is local-only (Tower runs on localhost)
- Markdown body of team files is NOT rendered in the UI — no XSS risk from freeform content
- Message log entries are displayed as plain text (not rendered as HTML/markdown) to prevent injection
- GitHub handles passed to `gh` CLI are sanitized (alphanumeric + hyphens only) to prevent shell injection

## Test Scenarios

### Functional Tests
1. Team tab always appears in TabBar (shows empty state when no team files exist)
2. Each team member card shows name, role, GitHub handle from frontmatter
3. Assigned issues (`assignee:`) and open PRs (`author:`) display correctly per member
4. Recent activity section shows last 7 days of merged PRs and closed issues
5. Refresh button triggers data re-fetch
6. Malformed team files are skipped with warning (not crash)
7. Team files with valid YAML but missing `github` field are skipped with warning
8. Duplicate GitHub handles across files: first file wins, duplicate skipped with warning
9. Invalid GitHub handle (no matching user) shows member card with "GitHub user not found" note
10. Messages from `codev/team/messages.md` display in reverse chronological order
11. Malformed message entries are skipped (not crash)
12. Empty messages file or missing file shows "No messages yet" state

### Non-Functional Tests
1. Tab renders within 2s for 10 team members (batched GraphQL)
2. Tab gracefully handles GitHub API failures (shows member cards from files, error banner for GitHub data)
3. Tab gracefully handles unauthenticated `gh` CLI (error message, no crash)
4. Tab follows responsive layout patterns for mobile view

## Dependencies
- **External Services**: GitHub API (via `gh` CLI or Octokit)
- **Internal Systems**: Tower server (new `/api/team` endpoint), Dashboard React app
- **Libraries**: gray-matter (or existing YAML frontmatter parser), existing React component patterns

## Risks and Mitigation
| Risk | Probability | Impact | Mitigation Strategy |
|------|------------|--------|-------------------|
| GitHub API rate limiting | Medium | Medium | Cache responses, only fetch when tab active, batch requests |
| Team files with invalid YAML | Low | Low | Skip malformed files, log warning, show partial results |
| Large teams (>20 members) | Low | Medium | Paginate or virtualize the member list |

## Expert Consultation

**Date**: 2026-03-06
**Models Consulted**: Gemini, Codex (GPT), Claude

**Key feedback incorporated**:
- **Gemini**: Clarified GitHub search semantics (assignee vs author vs reviewer). Added batched GraphQL recommendation for performance. Clarified markdown body is not rendered.
- **Codex**: Tightened data scope definitions. Added edge cases for missing/duplicate/invalid GitHub fields. Clarified tab visibility (always show, empty state). Added XSS note for message rendering.
- **Claude**: Fixed polling vs fetch-on-activation inconsistency. Clarified "right panel" terminology (these are main dashboard tabs). Added test scenarios for unauthenticated `gh` CLI and invalid handles. Noted batched GraphQL for performance target.

## Notes

The `codev/team/` directory and file format should also be added to the `codev-skeleton/` template so new projects adopting codev get the convention. However, the skeleton update is a small follow-up and not core to this spec.

The `codev/team/messages.md` append-only log is the foundation for inter-architect communication. v1 is read-only in the dashboard — messages are appended by team members directly (via git commits or `af send`-style tooling in the future). The team member file format is intentionally extensible — additional frontmatter fields can be added later without breaking existing files.
