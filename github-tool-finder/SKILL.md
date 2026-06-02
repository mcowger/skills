---
name: github-tool-finder
description: How to discover open-source tools and repositories on GitHub. Use this skill whenever the user asks to find tools, repos, libraries, or projects on GitHub — even if they don't explicitly say "GitHub." This includes requests like "find me tools for X", "what repos exist for Y", "show me alternatives to Z", or any exploration of the open-source ecosystem. Make sure to use this skill for any GitHub discovery task, including finding TUI tools, CLI utilities, libraries, frameworks, or any software project. Do NOT use general web search for GitHub discovery — use the GitHub-specific tooling described below.
---

# GitHub Tool Finder

This skill helps you discover open-source tools and repositories on GitHub efficiently and comprehensively. Your goal is to return **30-50 relevant results**, not just the first few obvious matches. Be expansive and err on the side of inclusion.

## Tool Priority Order

When searching GitHub, use available tooling in this exact priority order:

1. **Direct GitHub tools** - If you have access to GitHub-specific MCP tools or API integrations, use these first
2. **GitHub skills** - If you cannot use direct GitHub tools, check for any installed skills that specialize in GitHub operations
3. **gh CLI** - If neither tools nor skills are available, use the GitHub CLI (`gh`) for all search operations

**NEVER use general web search** (Tavily, Google, etc.) for GitHub discovery. GitHub has its own rich search infrastructure — use it directly.

## Search Strategy

### Initial Broad Search

Start with broad searches to cast a wide net. You want 30-50 results, so don't stop at the first page:

- Search by **keywords** in repo name/description
- Search by **topics** relevant to your query
- Search by **language** if relevant
- Combine multiple search terms using boolean operators
- Request large result sets to get comprehensive coverage

### Iterative Refinement

If initial searches don't yield enough results:

1. **Try synonym variations** - "tui" vs "terminal ui" vs "text user interface"
2. **Search related topics** - If looking for "vibe coding" tools, also search "pair programming", "ai coding", "developer tools"
3. **Explore awesome lists** - Search for curated "awesome" collections in the relevant category
4. **Check "related repositories"** - Once you find a good match, explore similar repos

### Constraint Verification

When a user specifies constraints (e.g., "must support worktrees", "needs macOS compatibility", "MIT license only"):

1. **Check the repo metadata first** - Use your available GitHub tooling to fetch license, language, stars
2. **Read the README** - If constraint isn't clear from metadata, fetch and read the README
3. **Check releases** - For activity/version info, query the releases endpoint
4. **Check recent activity** - For activity verification, check the last commit/push timestamp

**Do not guess** about whether a repo meets constraints. If it's unclear from the surface info, research it. If still unclear after checking README/releases, note it as "unclear" in your output rather than making assumptions.

## Output Format

Return results as a markdown table with these exact columns:

| Tool Name | GitHub Link | Star Count | Fork Count | Last Update | Description |
|-----------|-------------|------------|------------|-------------|-------------|
| ratatui   | https://github.com/ratatui-org/ratatui | 12.5k | 890 | 2 weeks ago | TUI framework for Rust with comprehensive widget library and active maintenance |

**Column details:**
- **Tool Name**: The repository name (e.g., `ratatui`, not `ratatui-org/ratatui`)
- **GitHub Link**: Full URL to the repository
- **Star Count**: Use human-readable format (e.g., `12.5k`, `1.2M`, `890`)
- **Fork Count**: Same format as stars
- **Last Update**: Relative time (e.g., "2 weeks ago", "3 days ago", "1 month ago")
- **Description**: 1-2 sentences summarizing what the tool does and why it matches the request

## Ranking Results

Sort results by **match quality**, which combines:

1. **Alignment with request** - How well does the repo match the user's stated need?
2. **Recency** - More recently updated repos rank higher (active maintenance matters)
3. **Stars** - Higher star count indicates community validation

A repo with 50k stars but last updated 3 years ago should rank below a repo with 5k stars updated last week, assuming both match the request equally well.

## Quality Thresholds

While you should be expansive, maintain basic quality:

- **Minimum stars**: Generally 50+ stars (exceptions for niche/new tools)
- **Activity**: Updated within the last 1-2 years (exceptions for "finished" tools)
- **Relevance**: Must genuinely match the request — don't pad with irrelevant results

## Example Workflow

Here's how a typical search might flow:

1. **Parse the request** - Identify key terms, constraints, and domain
2. **Initial broad search** - Search repos by keywords with high result limit
3. **Topic-based search** - Search by relevant topics
4. **Synonym searches** - Repeat with alternative terminology
5. **Constraint verification** - For each candidate, verify it meets stated constraints
6. **Deduplicate** - Remove forks/duplicates, keep the canonical repo
7. **Rank and format** - Sort by match quality, format as table

## Tips for Better Results

- **Use high result limits** - Request large result sets to get comprehensive coverage
- **Multiple queries are better than one** - Run 5-10 different searches to build up 30-50 results
- **Check "awesome" lists** - They're goldmines for curated tool collections
- **Look at "used by"** - Popular repos often have "used by" lists showing dependent projects
- **Explore organizations** - Some orgs specialize in certain tool categories

## When You Hit Limits

If GitHub search limits prevent getting 30-50 results:

1. Try broader search terms
2. Search multiple related topics
3. Look at awesome lists and curated collections
4. Explore organizations known for the tool category
5. Check "related repositories" from known good matches

It's better to return 25 highly relevant results than 50 padded with weak matches — but make a genuine effort to be comprehensive.
