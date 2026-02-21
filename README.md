# 📘 GitHub Repos Manager MCP — Complete Project Documentation

---

## Table of Contents

1. [What Is This Project?](#1-what-is-this-project)
2. [Problems This Project Solves](#2-problems-this-project-solves)
3. [Technologies Used](#3-technologies-used)
4. [Architecture & Codebase Overview](#4-architecture--codebase-overview)
5. [Project Directory Structure](#5-project-directory-structure)
6. [Core Components Deep Dive](#6-core-components-deep-dive)
7. [Complete Tool Reference (89 Tools)](#7-complete-tool-reference-89-tools)
8. [Configuration & Environment](#8-configuration--environment)
9. [Data Flow & Request Lifecycle](#9-data-flow--request-lifecycle)
10. [Security Model](#10-security-model)
11. [Caching Strategy](#11-caching-strategy)
12. [Error Handling Philosophy](#12-error-handling-philosophy)
13. [Testing Strategy](#13-testing-strategy)
14. [Setup & Installation Workflow](#14-setup--installation-workflow)
15. [Common Workflows & Use Cases](#15-common-workflows--use-cases)
16. [Troubleshooting Guide](#16-troubleshooting-guide)
17. [Future Roadmap (Placeholders in Code)](#17-future-roadmap-placeholders-in-code)

---

## 1. What Is This Project?

**GitHub Repos Manager MCP** is a **Node.js server** that implements the **Model Context Protocol (MCP)** to give AI assistants (like Claude Desktop, Cursor, Windsurf, Roo Code, Cline, etc.) the ability to **read, write, and manage GitHub repositories** entirely through natural language.

In plain terms: you talk to your AI assistant and say *"Create an issue titled 'Login Bug' in my repo"*, and this MCP server translates that into real GitHub API calls using your personal access token — no Docker, no CLI tools, no complex setup.

### Key Value Proposition

| Aspect | This Project | Typical Alternatives |
|---|---|---|
| **Setup** | One token, one command | Docker containers, CLI tools, config files |
| **Dependencies** | 2 npm packages | Heavy CLI toolchains (`gh`, Docker Engine) |
| **Tool Count** | 89 specialized tools | Usually 10-20 basic operations |
| **Transport** | Direct HTTP to GitHub API | Often wraps `gh` CLI (slower) |
| **Configuration** | Fine-grained (allow/deny tools, repos) | All-or-nothing access |

---

## 2. Problems This Project Solves

### 🔴 Problem 1: AI Assistants Can't Touch GitHub
AI coding assistants (Claude, GPT, etc.) operate in a sandbox. They can write code, but they **cannot** create issues, open pull requests, manage branches, or upload files on GitHub. This project **bridges that gap** by exposing 89 GitHub operations as MCP tools.

### 🔴 Problem 2: Complex Setup for GitHub Automation
Most GitHub automation tools require Docker, `gh` CLI, or heavyweight SDKs. This project needs **only a GitHub token** and **Node.js** — zero other dependencies.

### 🔴 Problem 3: No Granular Control over AI-GitHub Access
When you give an AI access to GitHub, you typically give it full access. This server lets you:
- **Allow/deny specific tools** (e.g., block `delete_file` but allow `create_issue`)
- **Restrict to specific repos** (e.g., only `myorg/frontend`, nothing else)
- **Set default repos** to avoid repeating `owner/repo` in every command

### 🔴 Problem 4: Rate Limit Headaches
GitHub's API has rate limits (5,000/hour). This project includes:
- Built-in rate limit detection and user-friendly error messages
- An in-memory **caching layer** (1-hour TTL) for analytics data
- Direct HTTP requests (no wasted calls through CLI wrappers)

### 🔴 Problem 5: Fragmented GitHub Workflows
Instead of switching between GitHub's web UI, CLI, and API docs, you can do **everything** through a single conversational interface:
- Create a branch → commit a file → open a PR → request review → merge — all in one chat session.

---

## 3. Technologies Used

### Core Runtime
| Technology | Version | Purpose |
|---|---|---|
| **Node.js** | ≥ 18.0.0 | Server runtime (uses native `fetch`, `crypto`) |
| **CommonJS** (`.cjs`) | — | Module system (for broad compatibility) |

### npm Dependencies (only 2!)
| Package | Version | Purpose |
|---|---|---|
| **@modelcontextprotocol/sdk** | ^0.4.0 | MCP protocol implementation — handles `Server`, `StdioServerTransport`, request schemas |
| **node-cache** | ^5.1.2 | In-memory caching for analytics data (avoids GitHub API rate limit hits) |

### External APIs
| API | Usage |
|---|---|
| **GitHub REST API v3** | All 89 tools — repos, issues, PRs, branches, files, search, orgs, webhooks, secrets, workflows, etc. |
| **GitHub GraphQL API v4** | Used by `GitHubAPIService.makeGraphQLRequest()` for complex queries |

### Protocol
| Protocol | Purpose |
|---|---|
| **Model Context Protocol (MCP)** | Standardized way for AI clients to discover and invoke server-side tools over **stdio** transport |

### Design Patterns Used
- **Service Layer Pattern** — `GitHubAPIService` is the single point of contact with the GitHub API
- **Handler Pattern** — Each domain (issues, PRs, repos, etc.) has its own handler module
- **Formatter Pattern** — Response formatting is separated from business logic
- **Configuration-Driven Tool Registration** — All 89 tools are defined declaratively in `tools-config.cjs`

---

## 4. Architecture & Codebase Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                     MCP Client (Claude, Cursor, etc.)           │
│  User says: "Create an issue titled 'Bug' in my-org/my-repo"   │
└──────────────────────────┬──────────────────────────────────────┘
                           │ stdio (JSON-RPC)
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│                    server.cjs (Entry Point)                      │
│  • Parses CLI args (--default-owner, --allowed-tools, etc.)     │
│  • Imports MCP SDK modules                                      │
│  • Instantiates GitHubMCPServer                                 │
└──────────────────────────┬──────────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────────┐
│              src/index.cjs — GitHubMCPServer Class              │
│                                                                 │
│  constructor()       → Initializes API service, loads config,   │
│                        sets up allowed/disabled tools,           │
│                        creates MCP Server instance               │
│                                                                 │
│  setupToolHandlers() → Registers ListTools & CallTool handlers  │
│                        89 tools dispatched via switch statement  │
│                                                                 │
│  run()               → Connects stdio transport, authenticates, │
│                        starts listening                          │
└──────────┬──────────────────────────────────────┬───────────────┘
           │                                      │
     ┌─────▼─────┐                         ┌─────▼──────┐
     │  Handlers  │                         │  Services   │
     │ (17 files) │                         │ (3 files)   │
     └─────┬─────┘                         └─────┬──────┘
           │                                      │
     ┌─────▼─────┐                         ┌─────▼──────┐
     │ Formatters │                         │ Cache Svc  │
     │ (8 files)  │                         │ File Svc   │
     └────────────┘                         └────────────┘
                           │
                           ▼
              ┌────────────────────────┐
              │   GitHub REST/GraphQL  │
              │        API v3/v4       │
              └────────────────────────┘
```

### Layer Responsibilities

| Layer | Files | Responsibility |
|---|---|---|
| **Entry Point** | `server.cjs` | CLI arg parsing, SDK import, server bootstrap |
| **Core Server** | `src/index.cjs` (1,091 lines) | `GitHubMCPServer` class — tool registration, request routing, default repo logic |
| **Handlers** | `src/handlers/*.cjs` (17 files) | Business logic for each tool domain |
| **Services** | `src/services/*.cjs` (3 files) | GitHub API calls, file I/O, caching |
| **Formatters** | `src/formatters/*.cjs` (8 files) | Transform raw API responses into readable text |
| **Utilities** | `src/utils/*.cjs` (7 files) | Error handling, validation, tool config, response formatting |

---

## 5. Project Directory Structure

```
github-repos-manager-mcp/
│
├── server.cjs                      # Entry point — bootstraps the MCP server
├── package.json                    # Project metadata, 2 dependencies
├── package-lock.json               # Dependency lock file
├── .gitignore                      # Standard Node.js ignores
├── README.md                       # Original project documentation
├── TEST-CLIENT-GUIDE.md            # Guide for testing with MCP clients
├── TEST-RESULTS-REPORT.MD          # Test results documentation
│
├── test-all-tools.js               # Comprehensive test suite (89 tools)
├── test-branch-tools.js            # Branch-specific tests
├── test-tools.js                   # General tool tests
├── test-workflow-direct.js         # Workflow handler tests
├── test-workflow-tools.js          # Workflow integration tests
│
└── src/
    ├── index.cjs                   # GitHubMCPServer class (1,091 lines)
    │
    ├── handlers/                   # Business logic — 17 handler modules
    │   ├── repository.cjs          #   list_repos, get_repo_info, search_repos, get_repo_contents, list_repo_collaborators
    │   ├── issues.cjs              #   list_issues, create_issue, edit_issue, get_issue_details, lock/unlock, assignees
    │   ├── comments.cjs            #   list_issue_comments, create/edit/delete_issue_comment
    │   ├── pull-requests.cjs       #   list_prs
    │   ├── enhanced-pull-requests.cjs # create/edit PR, get_pr_details, reviews, files
    │   ├── branches-commits.cjs    #   list_branches, create_branch, list_commits, get_commit_details, compare_commits
    │   ├── users.cjs               #   get_user_info
    │   ├── labels.cjs              #   list_repo_labels, create/edit/delete_label
    │   ├── milestones.cjs          #   list/create/edit/delete_milestone
    │   ├── file-management.cjs     #   create/update/upload/delete_file
    │   ├── security.cjs            #   deploy_keys, webhooks, secrets
    │   ├── workflows.cjs           #   GitHub Actions — list/trigger/cancel workflows
    │   ├── analytics.cjs           #   repo_stats, topics, languages, stargazers, watchers, forks, traffic
    │   ├── search.cjs              #   search issues, commits, code, users, topics
    │   ├── organizations.cjs       #   org repos, members, info, teams, team management
    │   ├── projects.cjs            #   Classic projects — list/create/manage
    │   └── advanced-features.cjs   #   Code quality, dashboards, reporting, notifications, releases, dependencies
    │
    ├── services/                   # External integrations — 3 service modules
    │   ├── github-api.cjs          #   GitHubAPIService class (1,086 lines, 60+ methods)
    │   ├── cache-service.cjs       #   In-memory cache (node-cache) with 1hr TTL
    │   └── file-service.cjs        #   File read/encode/upload helpers
    │
    ├── formatters/                 # Output formatting — 8 formatter modules
    │   ├── repository.cjs          #   Format repo listings, info, contents, collaborators
    │   ├── issues.cjs              #   Format issue lists, details, state changes
    │   ├── comments.cjs            #   Format comment lists and modifications
    │   ├── pull-requests.cjs       #   Format PR listings
    │   ├── branches-commits.cjs    #   Format branch lists, commit history, diffs
    │   ├── labels.cjs              #   Format label operations
    │   ├── milestones.cjs          #   Format milestone operations
    │   └── users.cjs               #   Format user profile data
    │
    └── utils/                      # Shared utilities — 7 utility modules
        ├── tools-config.cjs        #   ALL 89 tool definitions (2,852 lines) — names, descriptions, input schemas
        ├── error-handler.cjs       #   MCPError class, safeAsync/safeSync wrappers, error categorization
        ├── response-formatter.cjs  #   Convert handler responses to MCP-compatible format
        ├── shared-utils.cjs        #   getOwnerRepo, validation, logging helpers
        ├── validators.cjs          #   Webhook signature validation, permission checking
        ├── formatters.cjs          #   Traffic and language data formatters
        └── markdown.cjs            #   Markdown formatting utilities
```

---

## 6. Core Components Deep Dive

### 6.1 `server.cjs` — Entry Point (68 lines)

The entry point is minimal by design. It:
1. Parses CLI arguments: `--default-owner`, `--default-repo`, `--disabled-tools`, `--allowed-tools`, `--allowed-repos`
2. Dynamically imports the MCP SDK modules (`Server`, `StdioServerTransport`, schemas)
3. Instantiates `GitHubMCPServer` with config and SDK modules
4. Calls `server.run()`

### 6.2 `src/index.cjs` — GitHubMCPServer Class (1,091 lines)

The heart of the project. Key methods:

| Method | Lines | Purpose |
|---|---|---|
| `constructor(config, sdkModules)` | 27–130 | Reads `GH_TOKEN`, sets up defaults, initializes `GitHubAPIService`, configures tool allow/deny lists, creates MCP Server |
| `setDefaultRepo(owner, repo)` | 132–167 | Validates and sets the default `owner/repo` for all subsequent tool calls |
| `getHandlerArgs(args)` | 169–211 | Resolves `owner` and `repo` from args, falling back to the default repo. Also handles allowed-repo validation |
| `setupToolHandlers()` | 213–986 | Registers the `ListTools` handler (filters by allow/deny lists) and the `CallTool` handler (massive switch statement dispatching to 89 tool handlers) |
| `run()` | 988–1061 | Connects `StdioServerTransport`, tests GitHub authentication, logs startup |

### 6.3 `src/services/github-api.cjs` — GitHubAPIService (1,086 lines)

A comprehensive wrapper around the GitHub API with **60+ methods**. Key capabilities:

- **`makeGitHubRequest(endpoint, options)`** — Core HTTP method using native `fetch` with auth headers
- **`makeGraphQLRequest(query, variables)`** — GraphQL support for complex queries
- **Repository**: create/update/delete files, get contents, traffic, stats
- **Pull Requests**: create, update, get details, list reviews, create reviews, list files
- **Security**: deploy keys, webhooks, secrets (CRUD for all)
- **Workflows**: list, trigger, cancel, download artifacts
- **Analytics**: traffic views/clones/referrers, contributor stats
- **Search**: issues, commits, code, users, topics
- **Organizations**: repos, members, teams, team repo management
- **Notifications**: list, mark read, manage threads
- **Releases**: create, update, delete, list assets
- **Dependencies**: dependency graph, Dependabot alerts

### 6.4 `src/services/cache-service.cjs` — Caching Layer (84 lines)

Uses **node-cache** with:
- **Default TTL**: 3,600 seconds (1 hour)
- **Check Period**: 600 seconds (10 minutes)
- **Key Format**: `repo:{owner}:{repo}:{dataType}`
- Used primarily by the **analytics handler** to cache traffic, languages, and stats data

### 6.5 `src/services/file-service.cjs` — File Operations (104 lines)

Provides helpers for:
- `createFile` / `updateFile` — text file CRUD via GitHub Contents API
- `uploadBinaryFile` — reads local file, base64 encodes, checks for existing file (SHA), creates or updates
- `deleteFile` — removes file via GitHub Contents API connected with commit message

### 6.6 `src/utils/tools-config.cjs` — Tool Definitions (2,852 lines)

The largest file in the project. Contains the **declarative definition** of all 89 tools including:
- `name` — unique string identifier
- `description` — human-readable explanation
- `inputSchema` — JSON Schema defining all parameters, types, enums, defaults, and required fields

### 6.7 `src/utils/error-handler.cjs` — Error System (332 lines)

Provides a robust error handling framework:
- **`MCPError`** class — extends `Error` with `type`, `originalError`, `context`, `timestamp`
- **Error types**: `VALIDATION`, `AUTHENTICATION`, `PERMISSION`, `API_ERROR`, `NETWORK_ERROR`, `RATE_LIMIT`, `NOT_FOUND`, `INTERNAL`, `CONFIGURATION`
- **`safeAsync(fn)`** — wraps async functions to never throw (returns error responses instead)
- **`wrapHandler(handler)`** — wraps tool handlers for crash-proof operation
- **`categorizeError(error)`** — auto-classifies errors based on message patterns
- **`createUserFriendlyMessage()`** — converts raw errors into helpful user-facing messages

---

## 7. Complete Tool Reference (89 Tools)

### 📁 Repository Management (6 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_repos` | `repository.cjs` | List user repositories with pagination, visibility filter, sorting |
| `get_repo_info` | `repository.cjs` | Get comprehensive repository information and stats |
| `search_repos` | `repository.cjs` | Search GitHub repositories with sorting options |
| `get_repo_contents` | `repository.cjs` | Browse files and directories (supports branch/commit refs) |
| `set_default_repo` | `index.cjs` (server-level) | Set a default owner/repo for all subsequent commands |
| `list_repo_collaborators` | `repository.cjs` | List collaborators with permission-based filtering |

### 🎫 Issue Management (8 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_issues` | `issues.cjs` | List issues with state filtering (open/closed/all) |
| `create_issue` | `issues.cjs` | Create issues with labels, assignees, and image uploads |
| `edit_issue` | `issues.cjs` | Edit title, body, state, labels, assignees, add images |
| `get_issue_details` | `issues.cjs` | Get complete issue information with all metadata |
| `lock_issue` | `issues.cjs` | Lock an issue (off-topic, too heated, resolved, spam) |
| `unlock_issue` | `issues.cjs` | Unlock a previously locked issue |
| `add_assignees_to_issue` | `issues.cjs` | Add one or more assignees to an issue |
| `remove_assignees_from_issue` | `issues.cjs` | Remove assignees from an issue |

### 💬 Issue Comments (4 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_issue_comments` | `comments.cjs` | List all comments with optional timestamp filtering |
| `create_issue_comment` | `comments.cjs` | Add a new comment to an issue |
| `edit_issue_comment` | `comments.cjs` | Edit an existing comment by ID |
| `delete_issue_comment` | `comments.cjs` | Delete a comment by ID |

### 🔄 Pull Request Management (7 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_prs` | `pull-requests.cjs` | List PRs with state filtering and pagination |
| `create_pull_request` | `enhanced-pull-requests.cjs` | Create a new PR with head/base branches |
| `edit_pull_request` | `enhanced-pull-requests.cjs` | Update PR title, body, state, base branch |
| `get_pr_details` | `enhanced-pull-requests.cjs` | Get comprehensive PR information |
| `list_pr_reviews` | `enhanced-pull-requests.cjs` | List all reviews on a PR |
| `create_pr_review` | `enhanced-pull-requests.cjs` | Submit a review (APPROVE, REQUEST_CHANGES, COMMENT) |
| `list_pr_files` | `enhanced-pull-requests.cjs` | List files changed in a PR with diff stats |

### 🌿 Branch & Commit Management (5 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_branches` | `branches-commits.cjs` | List branches with protection status |
| `create_branch` | `branches-commits.cjs` | Create a new branch from existing branch/commit |
| `list_commits` | `branches-commits.cjs` | List commits with author/date/branch filtering |
| `get_commit_details` | `branches-commits.cjs` | Get detailed commit info including file changes |
| `compare_commits` | `branches-commits.cjs` | Compare two commits/branches/tags |

### 📄 File & Content Management (4 tools)
| Tool | Handler File | Description |
|---|---|---|
| `create_file` | `file-management.cjs` | Create a new file in the repo with commit message |
| `update_file` | `file-management.cjs` | Update an existing file (requires SHA) |
| `upload_file` | `file-management.cjs` | Upload a local file (binary supported) |
| `delete_file` | `file-management.cjs` | Delete a file from the repository |

### 👤 User & Collaboration (2 tools)
| Tool | Handler File | Description |
|---|---|---|
| `get_user_info` | `users.cjs` | Get GitHub user profile (self or any user) |
| *(collaborators listed under repos)* | | |

### 🏷️ Labels Management (4 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_repo_labels` | `labels.cjs` | List all labels with colors and descriptions |
| `create_label` | `labels.cjs` | Create a label with custom color |
| `edit_label` | `labels.cjs` | Modify label name, color, or description |
| `delete_label` | `labels.cjs` | Delete a label |

### 🎯 Milestones Management (4 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_milestones` | `milestones.cjs` | List milestones with state/sort filtering |
| `create_milestone` | `milestones.cjs` | Create a milestone with due date |
| `edit_milestone` | `milestones.cjs` | Update milestone details |
| `delete_milestone` | `milestones.cjs` | Delete a milestone |

### 🔒 Security & Access (9 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_deploy_keys` | `security.cjs` | List all deploy keys |
| `create_deploy_key` | `security.cjs` | Add a new SSH deploy key |
| `delete_deploy_key` | `security.cjs` | Remove a deploy key |
| `list_webhooks` | `security.cjs` | List configured webhooks |
| `create_webhook` | `security.cjs` | Create a new webhook for events |
| `edit_webhook` | `security.cjs` | Modify webhook config/events |
| `delete_webhook` | `security.cjs` | Remove a webhook |
| `list_secrets` | `security.cjs` | List repository secret names |
| `update_secret` | `security.cjs` | Create/update an encrypted secret |

### ⚡ GitHub Actions & Workflows (6 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_workflows` | `workflows.cjs` | List all GitHub Actions workflows |
| `list_workflow_runs` | `workflows.cjs` | List workflow runs with filtering |
| `get_workflow_run_details` | `workflows.cjs` | Get detailed run information |
| `trigger_workflow` | `workflows.cjs` | Manually trigger a workflow dispatch |
| `download_workflow_artifacts` | `workflows.cjs` | Download artifacts from a run |
| `cancel_workflow_run` | `workflows.cjs` | Cancel a running workflow |

### 📊 Repository Analytics (8 tools)
| Tool | Handler File | Description |
|---|---|---|
| `get_repo_stats` | `analytics.cjs` | Comprehensive stats including contributor activity |
| `list_repo_topics` | `analytics.cjs` | List repository topics/tags |
| `update_repo_topics` | `analytics.cjs` | Update repository topics |
| `get_repo_languages` | `analytics.cjs` | Get programming languages with byte counts |
| `list_stargazers` | `analytics.cjs` | List users who starred the repo |
| `list_watchers` | `analytics.cjs` | List users watching the repo |
| `list_forks` | `analytics.cjs` | List all forks with sorting |
| `get_repo_traffic` | `analytics.cjs` | Traffic data — views, clones, referrers (admin access required) |

### 🔍 Advanced Search (5 tools)
| Tool | Handler File | Description |
|---|---|---|
| `search_issues` | `search.cjs` | Search issues/PRs across GitHub |
| `search_commits` | `search.cjs` | Search commits across repos |
| `search_code` | `search.cjs` | Search for code across GitHub |
| `search_users` | `search.cjs` | Search for users and organizations |
| `search_topics` | `search.cjs` | Search for repository topics |

### 🏢 Organization Management (6 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_org_repos` | `organizations.cjs` | List all repos in an organization |
| `list_org_members` | `organizations.cjs` | List org members with role filtering |
| `get_org_info` | `organizations.cjs` | Get detailed organization information |
| `list_org_teams` | `organizations.cjs` | List all teams in an organization |
| `get_team_members` | `organizations.cjs` | List members of a specific team |
| `manage_team_repos` | `organizations.cjs` | Add/remove team repo access |

### 📋 Projects (6 tools)
| Tool | Handler File | Description |
|---|---|---|
| `list_repo_projects` | `projects.cjs` | List classic projects |
| `create_project` | `projects.cjs` | Create a new project |
| `list_project_columns` | `projects.cjs` | List columns in a project |
| `list_project_cards` | `projects.cjs` | List cards in a column |
| `create_project_card` | `projects.cjs` | Create a card in a column |
| `move_project_card` | `projects.cjs` | Move a card between columns |

### 🚀 Advanced Features (6 tools)
| Tool | Handler File | Description |
|---|---|---|
| `code_quality_checks` | `advanced-features.cjs` | Code scanning results from GitHub's built-in tools |
| `custom_dashboards` | `advanced-features.cjs` | Aggregated repo metrics dashboard |
| `automated_reporting` | `advanced-features.cjs` | Generate repository activity reports |
| `notification_management` | `advanced-features.cjs` | Manage GitHub notifications |
| `release_management` | `advanced-features.cjs` | Create/update/delete releases |
| `dependency_analysis` | `advanced-features.cjs` | Dependency scanning and analysis |

---

## 8. Configuration & Environment

### Environment Variables

| Variable | Required | Description |
|---|---|---|
| `GH_TOKEN` | ✅ | GitHub Personal Access Token |
| `GH_DEFAULT_OWNER` | ❌ | Default repository owner |
| `GH_DEFAULT_REPO` | ❌ | Default repository name |
| `GH_ALLOWED_REPOS` | ❌ | Comma-separated list of allowed repos (e.g., `owner1/repo1,owner2`) |
| `GH_ALLOWED_TOOLS` | ❌ | Comma-separated list of allowed tool names |
| `GH_DISABLED_TOOLS` | ❌ | Comma-separated list of disabled tool names |

### CLI Arguments

| Argument | Example |
|---|---|
| `--default-owner` | `--default-owner microsoft` |
| `--default-repo` | `--default-repo vscode` |
| `--disabled-tools` | `--disabled-tools "delete_file,delete_label"` |
| `--allowed-tools` | `--allowed-tools "list_issues,create_issue"` |
| `--allowed-repos` | `--allowed-repos "myorg/frontend,myorg/backend"` |

### Configuration Priority (highest → lowest)
1. **CLI arguments** override everything
2. **Environment variables** are the default
3. **Runtime tool calls** (`set_default_repo`) can override at any time

### MCP Client Configuration (Claude Desktop)

```json
{
  "mcpServers": {
    "github-repos-manager": {
      "command": "npx",
      "args": ["-y", "github-repos-manager-mcp"],
      "env": {
        "GH_TOKEN": "ghp_YOUR_TOKEN",
        "GH_DEFAULT_OWNER": "your-org",
        "GH_DEFAULT_REPO": "your-repo",
        "GH_ALLOWED_REPOS": "your-org",
        "GH_DISABLED_TOOLS": "delete_file,delete_label"
      }
    }
  }
}
```

---

## 9. Data Flow & Request Lifecycle

Here's what happens when a user asks their AI: *"List open issues in microsoft/vscode"*

```
1. AI Client sends MCP CallTool request:
   { tool: "list_issues", args: { owner: "microsoft", repo: "vscode", state: "open" } }

2. server.cjs receives via StdioServerTransport (JSON-RPC over stdin/stdout)

3. GitHubMCPServer.setupToolHandlers() → CallTool handler fires

4. Tool name "list_issues" matched in switch statement

5. getHandlerArgs(args) resolves owner/repo:
   - Uses provided args.owner/args.repo
   - Falls back to defaultRepo if not provided
   - Validates against allowedRepos if configured
   
6. Handler dispatched: issueHandlerFunctions.listIssues(resolvedArgs, this.apiService)

7. issues.cjs → listIssues():
   - Validates parameters (state, per_page)
   - Calls apiService.makeGitHubRequest("/repos/microsoft/vscode/issues?state=open&per_page=10")

8. GitHubAPIService.makeGitHubRequest():
   - Constructs URL: https://api.github.com/repos/microsoft/vscode/issues?state=open&per_page=10
   - Adds Authorization header with token
   - Sends fetch() request
   - Parses JSON response

9. Response flows back through:
   - issueFormatters.formatListIssuesOutput() → human-readable text
   - formatHandlerResponse() → MCP-compatible { content: [{ type: "text", text: "..." }] }
   
10. MCP Server sends response back to AI Client via stdout
```

---

## 10. Security Model

### Authentication
- Uses **GitHub Personal Access Token** (PAT) with either classic or fine-grained scopes
- Token is passed via environment variable `GH_TOKEN` — **never stored on disk or in config files**
- Token is validated on startup via `apiService.testAuthentication()`

### Access Control Layers

```
Layer 1: GitHub Token Scopes
  └── Controls what the token can do on GitHub (repo, read:org, user, etc.)

Layer 2: Allowed Repositories (GH_ALLOWED_REPOS)
  └── Restricts which repos the server can operate on (validated per request)

Layer 3: Tool Allow/Deny Lists (GH_ALLOWED_TOOLS / GH_DISABLED_TOOLS)
  └── Controls which of the 89 tools are available to the AI client

Layer 4: Webhook Signature Validation (validators.cjs)
  └── HMAC-SHA256 verification for incoming webhook payloads
```

### Security Best Practices Enforced
- Token is trimmed and validated at startup
- Never logged or exposed in error messages
- Allowed-repos check happens **before** any API call
- `process.stderr` used for error logging (doesn't interfere with MCP stdout)

---

## 11. Caching Strategy

```
┌───────────────────────────────┐
│     cache-service.cjs         │
│                               │
│  Engine:   node-cache         │
│  TTL:      3,600s (1 hour)    │
│  Check:    600s (10 minutes)  │
│                               │
│  Key Format:                  │
│  repo:{owner}:{repo}:{type}   │
│                               │
│  Cached Data:                 │
│  • Traffic (views, clones)    │
│  • Programming languages      │
│  • Repository statistics       │
│  • Stargazers / watchers       │
└───────────────────────────────┘
```

The cache is used exclusively by the **analytics handler** to avoid burning through the 5,000 requests/hour rate limit on frequently-queried data.

---

## 12. Error Handling Philosophy

The project follows a **"never crash"** philosophy:

1. **Global handlers** — `uncaughtException` and `unhandledRejection` are caught at the process level
2. **Safe wrappers** — `safeAsync()` and `safeSync()` wrap any function to return error objects instead of throwing
3. **Error categorization** — Every error is auto-classified into one of 9 types (VALIDATION, AUTH, PERMISSION, API, NETWORK, RATE_LIMIT, NOT_FOUND, INTERNAL, CONFIG)
4. **User-friendly messages** — Raw GitHub API errors are translated into actionable advice:
   - `401` → *"Authentication failed. Please check your GitHub token."*
   - `403` → *"Permission denied. Your token may not have the required scope."*
   - `404` → *"Resource not found. Please verify the owner, repo, and resource exist."*
   - `422` → *"The request was invalid. Check your parameters."*
   - `429` → *"Rate limit exceeded. Wait for the reset window."*

---

## 13. Testing Strategy

The project includes **5 test files** with a focus on structural and integration testing:

| File | Lines | Tests |
|---|---|---|
| `test-all-tools.js` | 490 | Comprehensive test of all 89 tools — validates definitions, handler methods |
| `test-branch-tools.js` | ~70 | Branch-specific handler tests |
| `test-tools.js` | ~70 | General tool validation |
| `test-workflow-direct.js` | ~80 | Direct workflow handler tests |
| `test-workflow-tools.js` | ~120 | Workflow integration tests |

### Test Phases in `test-all-tools.js`

1. **Phase 1: Tool Definitions** — Verifies every tool has a `name`, `description`, and `inputSchema` in `tools-config.cjs`
2. **Phase 2: Handler Methods** — Verifies every tool maps to an existing handler function in the correct module

### Running Tests

```bash
# Structural tests (no GitHub token needed — uses dummy token)
node test-all-tools.js

# Branch-specific tests
node test-branch-tools.js

# Workflow tests
node test-workflow-tools.js
```

---

## 14. Setup & Installation Workflow

### Option A: Zero-Install (npx)

```bash
# One command — downloads and runs the latest version
npx -y github-repos-manager-mcp
```

### Option B: Local Development

```bash
# 1. Clone the repository
git clone https://github.com/kurdin/github-repos-manager.git
cd github-repos-manager

# 2. Install dependencies (only 2 packages)
npm install

# 3. Set your GitHub token
export GH_TOKEN="ghp_YOUR_TOKEN_HERE"    # macOS/Linux
set GH_TOKEN=ghp_YOUR_TOKEN_HERE         # Windows CMD
$env:GH_TOKEN = "ghp_YOUR_TOKEN_HERE"    # Windows PowerShell

# 4. Start the server
node server.cjs

# 5. (Optional) Set default repo
node server.cjs --default-owner microsoft --default-repo vscode
```

### Option C: MCP Client Configuration (Recommended)

Add to your MCP client's config file (e.g., Claude Desktop's `claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "github-repos-manager": {
      "command": "npx",
      "args": ["-y", "github-repos-manager-mcp"],
      "env": {
        "GH_TOKEN": "ghp_YOUR_TOKEN_HERE"
      }
    }
  }
}
```

**Config file locations:**
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

---

## 15. Common Workflows & Use Cases

### 🔄 Complete Feature Development Workflow
```
1. "Set default repository to my-org/my-app"
2. "Create a new branch called feature/dark-mode from develop"
3. "Create a file src/theme/dark.css with content '...' and commit message 'Add dark mode theme'"
4. "Create a pull request from feature/dark-mode to develop with title 'Add dark mode theme'"
5. "List the files changed in PR #42"
6. "Create a review on PR #42 with event APPROVE and body 'LGTM!'"
```

### 🐛 Bug Triage Workflow
```
1. "List all open issues with the bug label in my-org/my-app"
2. "Get details for issue #23"
3. "Add assignee 'dev-alice' to issue #23"
4. "Create a comment on issue #23: 'Investigating — likely related to the auth refactor'"
5. "Create a label 'priority-high' with color ff0000"
6. "Edit issue #23 to add label priority-high"
```

### 📊 Project Health Check Workflow
```
1. "Get repository stats for my-org/my-app"
2. "Get programming languages used in my-org/my-app"
3. "Get traffic data for my-org/my-app"
4. "List all stargazers for my-org/my-app"
5. "Run code quality checks for my-org/my-app"
6. "Generate an automated report for my-org/my-app"
```

### 🏢 Organization Management Workflow
```
1. "List all repositories in my-org"
2. "List all teams in my-org"
3. "Get members of the team 'backend-team' in my-org"
4. "Add repository access for 'backend-team' to my-org/new-service with push permission"
```

### 🔒 Security Audit Workflow
```
1. "List all deploy keys for my-org/my-app"
2. "List all webhooks for my-org/my-app"
3. "List all secrets for my-org/my-app"
4. "Run dependency analysis for my-org/my-app"
```

---

## 16. Troubleshooting Guide

| Symptom | Likely Cause | Fix |
|---|---|---|
| *"GH_TOKEN not set"* | Token not in environment | Add `GH_TOKEN` to MCP client env block |
| *"Authentication failed"* | Token expired or revoked | Generate a new PAT on GitHub |
| *"Permission denied (403)"* | Token missing required scopes | Add `repo`, `user:read`, `read:org` scopes |
| *"Resource not found (404)"* | Repo doesn't exist or is private | Verify repo name; ensure token has `repo` scope for private repos |
| *"Rate limit exceeded (429)"* | > 5,000 API calls/hour | Wait for reset; use caching; reduce query frequency |
| Server doesn't start | Node.js too old | Upgrade to Node.js ≥ 18 (`node --version`) |
| Tools not appearing in client | Tool filtered by allow/deny list | Check `GH_ALLOWED_TOOLS` / `GH_DISABLED_TOOLS` |
| Default repo not working | Env vars misspelled | Verify `GH_DEFAULT_OWNER` and `GH_DEFAULT_REPO` exactly |
| Image upload failing | File not found, wrong format, or too large | Ensure file exists; use PNG/JPG/GIF/WebP; check size limits |
| Windows: *"npx not found"* | Shell issue | Use `npx.cmd` instead of `npx` in MCP config |

---

## 17. Future Roadmap (Placeholders in Code)

The codebase contains several **placeholder features** that are partially implemented or stubbed out for future development:

| Feature | Current Status | Location |
|---|---|---|
| **Permission checking** | Returns `true` always (placeholder) | `validators.cjs` → `checkPermissions()` |
| **Code quality checks** | Uses GitHub's code scanning API (real implementation) | `advanced-features.cjs` |
| **Custom dashboards** | Real implementation aggregating metrics | `advanced-features.cjs` |
| **Automated reporting** | Real implementation generating reports | `advanced-features.cjs` |
| **Notification management** | Real implementation via GitHub API | `advanced-features.cjs` |
| **Release management** | Full CRUD implementation | `advanced-features.cjs` |
| **Dependency analysis** | Uses GitHub dependency graph API | `advanced-features.cjs` |
| **GraphQL queries** | Infrastructure exists, lightly used | `github-api.cjs` |

---

## Summary

**GitHub Repos Manager MCP** is a thoughtfully architected Node.js server that bridges AI assistants with the full power of the GitHub API. With 89 tools, fine-grained access control, built-in caching, robust error handling, and only 2 npm dependencies, it represents a lean yet powerful approach to GitHub automation that prioritizes simplicity, security, and developer experience.

---
