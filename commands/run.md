---
allowed-tools: mcp__p4mcp__query_changelists, mcp__p4mcp__query_files, mcp__p4mcp__query_reviews, mcp__p4mcp__modify_reviews
description: Code review a Perforce changelist
---

Provide a code review for the given Perforce changelist.

`$ARGUMENTS` is the changelist number to review, optionally followed by `--comment`.
Example: `/p4-code-review:run 2095` or `/p4-code-review:run 2095 --comment`.

**Server name is a placeholder — resolve it first.** Throughout this document, tools are
written as `mcp__p4mcp__<tool>`. The `p4mcp` segment is a PLACEHOLDER for
whatever name the user registered their Perforce MCP server under (common alternatives:
`p4mcp`, `p4`, `helix`). Do NOT assume it is literally `p4mcp`.

Before doing anything else, resolve the real prefix:
- Look at the MCP tools actually available in this session and find the server that exposes
  the tools `query_changelists`, `query_files`, `query_reviews`, and `modify_reviews`
  (their fully-qualified names look like `mcp__<server>__query_changelists`, etc.).
- Take the `<server>` segment from those names — that is the real prefix.
- If exactly one server exposes these tools, use it. If none do, stop and tell the user no
  Perforce MCP server is connected. If more than one does, ask the user which to use.
- Substitute this resolved prefix everywhere `p4mcp` appears below, and pass the
  resolved prefix explicitly to every subagent you launch so they call the right tools.
  (The `allowed-tools` list in this command's frontmatter is pre-approval only, keyed to the
  default `p4mcp` name; if your server has a different name the tools still work, they
  will just prompt for permission — resolving the prefix at runtime is what makes the command
  functionally correct.)

**Agent assumptions (applies to all agents and subagents):**
- All tools are functional and will work without error. Do not test tools or make exploratory calls. Make sure this is clear to every subagent that is launched.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

**Diffing a changelist (share this with every reviewing subagent):**
- Get the changelist metadata with `mcp__p4mcp__query_changelists` (action `get`). This returns the file list, each file's `action` (add/edit/delete/branch/integrate) and its `rev`.
- For a **submitted, edited** file, get the diff with `mcp__p4mcp__query_files` (action `diff`) comparing `//path#<rev-1>` against `//path#<rev>`.
- For an **added** file, review the full content with `mcp__p4mcp__query_files` (action `content`) at `//path#<rev>` — there is no prior revision.
- The `diff` output lists the FIRST file (`#rev-1`) on the left (`<` = removed) and the SECOND file (`#rev`) on the right (`>` = added). A diff shows only changed lines, not surrounding context — when a finding depends on a surviving (unchanged) line, confirm it against the full `content` of the `#<rev>` file before flagging.

To do this, follow these steps precisely:

1. Launch a haiku agent to check if any of the following are true:
   - The changelist does not exist, or is empty (no files).
   - The changelist is already committed and its associated Swarm review is `approved`, `rejected`, or `archived` (a finished review that no longer needs input). Find the associated review with `mcp__p4mcp__query_reviews` (action `list`, `keywords` = the changelist number, `keywords_fields` = `["changes"]`), then read its state via action `get`.
   - The changelist does not need code review (e.g. automated/bot submit, generated data, a trivial change that is obviously correct).
   - Claude has already commented on the associated Swarm review (check `mcp__p4mcp__query_reviews` action `comments` for comments authored by the current user / Claude).

   If any condition is true, stop and do not proceed.

   Note: Still review changelists that were themselves authored by Claude/bots when they contain real code.

2. Launch a haiku agent to return a list of depot paths (not their contents) for all relevant CLAUDE.md files including:
   - The root CLAUDE.md file, if it exists (e.g. `//depot/CLAUDE.md` or the depot/stream root of the changed files).
   - Any CLAUDE.md files in depot directories that contain files modified by the changelist, and in their parent directories up to the depot root.

   Use `mcp__p4mcp__query_files` (action `search`, `pattern` = `CLAUDE.md`) scoped to the relevant depot paths to locate them.

3. Launch a sonnet agent to view the changelist and return a summary of the changes. It should use `mcp__p4mcp__query_changelists` (action `get`) for the description and file list, and `mcp__p4mcp__query_files` (action `diff`/`content`) per the diffing guidance above.

4. Launch 4 agents in parallel to independently review the changes. Each agent should return the list of issues, where each issue includes a description, the depot file + line it occurs on, and the reason it was flagged (e.g. "CLAUDE.md adherence", "bug"). The agents should do the following:

   Agents 1 + 2: CLAUDE.md compliance sonnet agents
   Audit changes for CLAUDE.md compliance in parallel. Note: When evaluating CLAUDE.md compliance for a file, you should only consider CLAUDE.md files that share a depot path with the file or are in a parent directory.

   Agent 3: Opus bug agent (parallel subagent with agent 4)
   Scan for obvious bugs. Focus only on the diff itself without reading extra context. Flag only significant bugs; ignore nitpicks and likely false positives. Do not flag issues that you cannot validate without looking at context outside of the changelist diff.

   Agent 4: Opus bug agent (parallel subagent with agent 3)
   Look for problems that exist in the introduced code. This could be security issues, incorrect logic, etc. Only look for issues that fall within the changed code.

   **CRITICAL: We only want HIGH SIGNAL issues.** Flag issues where:
   - The code will fail to compile or parse (syntax errors, type errors, missing imports, unresolved references)
   - The code will definitely produce wrong results regardless of inputs (clear logic errors)
   - Clear, unambiguous CLAUDE.md violations where you can quote the exact rule being broken

   Do NOT flag:
   - Code style or quality concerns
   - Potential issues that depend on specific inputs or state
   - Subjective suggestions or improvements

   If you are not certain an issue is real, do not flag it. False positives erode trust and waste reviewer time.

   In addition to the above, each subagent should be told the changelist description (and the associated Swarm review title/description if one exists). This will help provide context regarding the author's intent.

5. For each issue found in the previous step by agents 3 and 4, launch parallel subagents to validate the issue. These subagents should get the changelist description along with a description of the issue. The agent's job is to review the issue to validate that the stated issue is truly an issue with high confidence. For example, if an issue such as "variable is not defined" was flagged, the subagent's job would be to validate that is actually true in the code — including reading the full `content` of the `#<rev>` file, not just the diff, so a surviving line is not missed. Another example would be CLAUDE.md issues. The agent should validate that the CLAUDE.md rule that was violated is scoped for this file and is actually violated. Use Opus subagents for bugs and logic issues, and sonnet agents for CLAUDE.md violations.

6. Filter out any issues that were not validated in step 5. This step will give us our list of high signal issues for our review.

7. Output a summary of the review findings to the terminal:
   - If issues were found, list each issue with a brief description and its `//depot/path#rev` location.
   - If no issues were found, state: "No issues found. Checked for bugs and CLAUDE.md compliance."

   If `--comment` argument was NOT provided, stop here. Do not post any Swarm comments.

   Before posting anything, resolve the Swarm review for this changelist with `mcp__p4mcp__query_reviews` (action `list`, `keywords` = the changelist number, `keywords_fields` = `["changes"]`). If NO review exists, do not create one — print the findings to the terminal, tell the user no Swarm review is associated with this changelist, and stop.

   If `--comment` argument IS provided, a review exists, and NO issues were found, post a summary comment using `mcp__p4mcp__modify_reviews` (action `add_comment`, `review_id` = the review, `body` = the summary) and stop.

   If `--comment` argument IS provided, a review exists, and issues were found, continue to step 8.

8. Create a list of all comments that you plan on leaving. This is only for you to make sure you are comfortable with the comments. Do not post this list anywhere.

9. Post inline comments for each issue using `mcp__p4mcp__modify_reviews` (action `add_comment`). For each comment set:
   - `review_id`: the resolved review ID
   - `body`: a brief description of the issue and the suggested fix
   - `comment_file_path`: the depot path of the file (e.g. `//depot/main/foo.cpp`)
   - `comment_right_line`: the line in the new revision the comment applies to (use `comment_left_line` instead when commenting on a removed line)
   - `comment_version`: the latest review version (get it from `mcp__p4mcp__query_reviews` action `get`)

   Guidance for the body:
   - For small, self-contained fixes, include the corrected code snippet inline in the body.
   - For larger fixes (6+ lines, structural changes, or changes spanning multiple locations), describe the issue and suggested fix without a full snippet.
   - Only include a suggested snippet UNLESS applying it fixes the issue entirely. If follow up steps are required, describe them instead.

   **IMPORTANT: Only post ONE comment per unique issue. Do not post duplicate comments.**

Use this list when evaluating issues in Steps 4 and 5 (these are false positives, do NOT flag):

- Pre-existing issues (present before this changelist)
- Something that appears to be a bug but is actually correct
- Pedantic nitpicks that a senior engineer would not flag
- Issues that a linter will catch (do not run the linter to verify)
- General code quality concerns (e.g., lack of test coverage, general security issues) unless explicitly required in CLAUDE.md
- Issues mentioned in CLAUDE.md but explicitly silenced in the code (e.g., via a lint ignore comment)

Notes:

- Use the `p4mcp` tools to interact with Perforce and Swarm (fetch changelists, diffs, reviews, comments). Do not use web fetch or the `p4` shell CLI.
- Create a todo list before starting.
- Cite each issue with its `//depot/path#rev` and line number in the comment body so the reviewer can locate it precisely.
- If no issues are found and `--comment` argument is provided (and a review exists), post a comment with the following format:

---

## Code review

No issues found. Checked for bugs and CLAUDE.md compliance.

---
