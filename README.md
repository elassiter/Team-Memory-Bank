# Team Memory Bank

A dual MCP (Model Context Protocol) memory system for GitHub Copilot that separates **team-wide architectural patterns** from **project-specific context**.

## Overview

This repository implements a structured approach to managing development knowledge across your organization using two distinct memory banks:

### 🏢 Team Memory (Global)
**Purpose**: Centralized repository for organization-wide patterns, standards, and architectural decisions

**Contains**:
- **Architecture Decisions** (`architecture-decisions/`) - ADRs documenting major technical choices
- **Global .NET Patterns** (`dotnet-global/`) - Reusable patterns like Clean Architecture, CQRS, Result<T>
- **Team Standards** (`team-standards/`) - Coding conventions, EF Core guidelines, SQL Server practices

**Access Pattern**: Read-only by default (ask before modifying to avoid unintended impacts)

**When to Use**:
- Establishing consistent patterns across all projects
- Documenting architectural decisions that affect multiple teams
- Defining company-wide coding standards

### 📁 Project Memory (Local)
**Purpose**: Project-specific context, active work tracking, and implementation details

**Structure**: Follows the LouisDesca convention with 6 core files:
- `00-project-brief.md` - Project identity and goals
- `01-product-context.md` - Features, user stories, business requirements
- `02-active-context.md` - Current work, immediate next steps, active session state
- `03-system-patterns.md` - Project-specific technical patterns and architecture
- `04-tech-context.md` - Technology stack, configuration, setup instructions
- `05-progress.md` - Completed work, metrics, blockers, sprint status

**Access Pattern**: Read and write freely throughout development

**When to Use**:
- Tracking active development work
- Documenting project-specific patterns
- Managing feature implementation and progress

## Why Two Memory Banks?

| Aspect | Team Memory | Project Memory |
|--------|-------------|----------------|
| **Scope** | Organization-wide | Single project |
| **Lifespan** | Permanent, evolves slowly | Active during project lifecycle |
| **Update Frequency** | Infrequent (architectural changes) | Frequent (daily/hourly) |
| **Examples** | CQRS pattern, Clean Architecture | Current sprint tasks, API endpoints |
| **Impact** | Affects all projects | Isolated to one project |

---

## Configuration for VS Code

### Prerequisites

#### Required Software
- **VS Code** with GitHub Copilot installed
- **Node.js and npm** (for npx)
- **Git** for cloning the repository

#### Check if Node.js and npm are Installed

Open PowerShell and run:

```powershell
node --version
npm --version
```

If both commands return version numbers (e.g., `v20.10.0` and `10.2.3`), you're ready to proceed.

#### Install Node.js and npm (if needed)

If the commands above fail or are not found:

1. **Download Node.js**: Visit [https://nodejs.org](https://nodejs.org)
2. **Choose LTS version**: Download the "LTS" (Long Term Support) version for Windows
3. **Run the installer**: 
   - Double-click the downloaded `.msi` file
   - Follow the installation wizard (accept defaults)
   - Ensure "Add to PATH" is checked
4. **Verify installation**: Close and reopen PowerShell, then run:
   ```powershell
   node --version
   npm --version
   ```

**Note**: npm is automatically installed with Node.js.

### Initial Setup

#### Step 1: Clone Team Memory Bank Repository

```powershell
git clone <repository-url> c:\repos\team-memory-bank
```

Or if the repository doesn't exist yet:

```powershell
mkdir c:\repos\team-memory-bank
cd c:\repos\team-memory-bank
git init
```

#### Step 2: Configure Global Team Memory MCP Server

1. Open VS Code Command Palette: **Ctrl+Shift+P**
2. Type and select: **`MCP: Open User Configuration`**
3. This opens your global MCP configuration file (applies to all VS Code windows)
4. Add the following server configuration to the `"servers"` section:

```json
{
  "servers": {
    "team-memory": {
      "command": "npx",
      "args": ["-y", "@allpepper/memory-bank-mcp@latest"],
      "env": {
        "MEMORY_BANK_ROOT": "C:\\repos\\team-memory-bank"
      },
      "disabled": false,
      "autoApprove": [
        "memory_bank_read",
        "list_projects",
        "list_project_files",
        "memory_bank_write",
        "memory_bank_update"
      ]
    }
  }
}
```

**Important**: Use double backslashes (`\\`) in Windows paths within JSON files.

#### Step 3: Configure Project Memory MCP Server

For **each project** you work on:

1. Navigate to your project root directory
2. Create the MCP configuration directory and file:
   ```powershell
   mkdir .vscode
   ```
   
3. Open VS Code Command Palette: **Ctrl+Shift+P**
4. Type and select: **`MCP: Open Workspace Folder MCP Configuration`**
5. This creates/opens `.vscode/mcp.json` in your project root
6. Add the following configuration:

```json
{
  "servers": {
    "project-memory": {
      "command": "npx",
      "args": ["-y", "@allpepper/memory-bank-mcp@latest"],
      "env": {
        "MEMORY_BANK_ROOT": "${workspaceFolder}\\.memory-bank"
      },
      "disabled": false,
      "autoApprove": [
        "memory_bank_read",
        "list_projects",
        "list_project_files",
        "memory_bank_write",
        "memory_bank_update"
      ]
    }
  }
}
```

**Note**: The `${workspaceFolder}` variable automatically resolves to your project root.

#### Step 4: Add GitHub Copilot Instructions

In your project repository, create or update `.github/copilot-instructions.md`:

```powershell
mkdir .github
```

Then create `.github/copilot-instructions.md` with the complete dual MCP memory system instructions. *(See the complete template in the appendix below)*

#### Step 5: Restart VS Code

1. Close **all** VS Code windows
2. Restart VS Code
3. Open your project

---

## Verification & Testing

### Check MCP Server Status

1. Open VS Code Command Palette: **Ctrl+Shift+P**
2. Type: **`MCP: List Connected Servers`**
3. You should see:
   - `team-memory` (connected)
   - `project-memory` (connected for workspace)

### Test Team Memory Access

Open GitHub Copilot Chat and try:

```
@workspace Can you list the projects in team-memory?
```

Or:

```
@workspace Read the dotnet-global system patterns from team-memory
```

### Test Project Memory Access

In your project, ask Copilot:

```
@workspace Can you list the project memory files?
```

Or:

```
@workspace Read the project brief from project-memory
```

### View MCP Logs (If Issues Occur)

1. **Ctrl+Shift+P** → **`MCP: Show MCP Logs`**
2. Check for connection errors or missing dependencies
3. Common issues:
   - Node.js not in PATH
   - Invalid JSON syntax in configuration
   - Incorrect path separators (use `\\` on Windows)

---

## Usage Workflow

### At Session Start

GitHub Copilot will automatically:
1. Read global patterns from `team-memory`
2. Read all project memory files from `project-memory`
3. Confirm understanding of current context

### During Development

Copilot will:
- Apply team patterns from global memory
- Update `02-active-context.md` after each subtask
- Track progress in project memory

### At Session End

Copilot will:
- Save current state to `02-active-context.md`
- Update completed work in `05-progress.md`
- Document new patterns in `03-system-patterns.md`

---

## Directory Structure

### Team Memory Bank (This Repository)

```
c:\repos\team-memory-bank\
├── README.md (this file)
├── architecture-decisions/
│   └── adr-001-clean-architecture.md
├── dotnet-global/
│   └── systemPatterns.md
└── team-standards/
    ├── coding-conventions.md
    └── efcore-sqlserver.md
```

### Project Memory (In Each Project)

```
your-project\
├── .vscode/
│   └── mcp.json (project memory configuration)
├── .github/
│   └── copilot-instructions.md (workflow instructions)
└── .memory-bank/
    └── project/
        ├── 00-project-brief.md
        ├── 01-product-context.md
        ├── 02-active-context.md
        ├── 03-system-patterns.md
        ├── 04-tech-context.md
        └── 05-progress.md
```

---

## Troubleshooting

### MCP Server Won't Connect

1. Verify Node.js is installed: `node --version`
2. Check JSON syntax in configuration files (use a JSON validator)
3. Ensure paths use double backslashes on Windows: `C:\\repos\\...`
4. Check MCP logs: **Ctrl+Shift+P** → **`MCP: Show MCP Logs`**

### "Memory Bank Root Not Found" Error

1. Verify the path exists: `dir C:\repos\team-memory-bank`
2. Check environment variable in MCP config
3. Ensure no typos in path names

### Auto-Approve Not Working

The `autoApprove` array in your MCP configuration should include all operations you want to happen automatically. If you're being prompted for every operation, verify the array is correctly formatted.

### Changes Not Persisting

1. Ensure MCP server has write permissions to the directory
2. Check if files are read-only
3. Verify Copilot is using `memory_bank_update()` not just reading

---

## Best Practices

### Team Memory
- ✅ Document patterns used across multiple projects
- ✅ Keep ADRs up to date with rationale
- ✅ Review changes with team before committing
- ❌ Don't add project-specific implementation details
- ❌ Don't frequently modify (impacts all projects)

### Project Memory
- ✅ Update `02-active-context.md` after each meaningful change
- ✅ Keep `05-progress.md` current for tracking
- ✅ Document project-specific patterns in `03-system-patterns.md`
- ❌ Don't duplicate global patterns (reference them instead)
- ❌ Don't let active context grow stale

---

## Appendix: Complete Copilot Instructions Template

Place this in `.github/copilot-instructions.md` in each project:

```markdown
# GitHub Copilot Instructions - Dual MCP Memory System

## Memory Architecture

You have TWO MCP memory bank servers:

### 1. Team Memory (Global Patterns)
- **MCP Server**: `team-memory`
- **Location**: `C:\repos\team-memory-bank\`
- **Access**: Read-only (ask before writing)
- **Projects**: 
  - `dotnet-global` - Universal .NET patterns
  - `team-standards` - Company coding standards
  - `architecture-decisions` - ADRs

### 2. Project Memory (This Project)
- **MCP Server**: `project-memory`
- **Location**: `.memory-bank\project\` in this repository
- **Access**: Read and write freely
- **Structure**: LouisDesca convention (6 core files)

## File Structure

### Project Memory Files (in `.memory-bank\project\`):
- `00-project-brief.md` - Project identity and goals
- `01-product-context.md` - Features, user stories, business context
- `02-active-context.md` - **Current work** (update frequently)
- `03-system-patterns.md` - Technical architecture and patterns
- `04-tech-context.md` - Tech stack, setup, configuration
- `05-progress.md` - Completed work, metrics, blockers

## Workflow Protocol

### Session Start (ALWAYS):

**Step 1: Read Team Patterns**
```
Use team-memory MCP server:
- memory_bank_read("dotnet-global/systemPatterns.md")
- Get universal .NET patterns (Clean Arch, CQRS, Result<T>, etc.)
```

**Step 2: Read Project Memory**
```
Use project-memory MCP server to read ALL files in order:
1. memory_bank_read("project/00-project-brief.md")
2. memory_bank_read("project/01-product-context.md")
3. memory_bank_read("project/02-active-context.md")
4. memory_bank_read("project/03-system-patterns.md")
5. memory_bank_read("project/04-tech-context.md")
6. memory_bank_read("project/05-progress.md")
```

**Step 3: Confirm Understanding**
```
Summarize:
- Team patterns that apply
- This project's purpose
- Current active task
- What to do next
```

### During Development:

**As You Work:**
- Apply team patterns from `team-memory`
- Follow project patterns from `03-system-patterns.md`
- Update `02-active-context.md` after each subtask:
```
  Use project-memory.memory_bank_update(
    "project/02-active-context.md",
    <updated content>
  )
```

**When Completing Features:**
- Update `02-active-context.md` (move to "Recent Changes")
- Update `05-progress.md` (mark complete, update metrics)

**When Discovering Patterns:**
- Add to `03-system-patterns.md` if project-specific
- Ask about adding to `team-memory` if applicable globally

### Session End:

**Always Update:**
```
Use project-memory.memory_bank_update() to save:
- Current state in 02-active-context.md
- Any completed work in 05-progress.md
- New patterns in 03-system-patterns.md
```

## MCP Commands Reference

### Team Memory (Read-Only)
```
team-memory.memory_bank_read("dotnet-global/systemPatterns.md")
team-memory.memory_bank_read("team-standards/coding-conventions.md")
team-memory.list_projects()
team-memory.list_project_files("dotnet-global")

# Write operations - ASK FIRST
team-memory.memory_bank_write("dotnet-global/new-pattern.md", content)
```

### Project Memory (Read/Write)
```
project-memory.memory_bank_read("project/02-active-context.md")
project-memory.memory_bank_update("project/02-active-context.md", updated_content)
project-memory.memory_bank_write("project/new-file.md", content)
project-memory.list_projects()
project-memory.list_project_files("project")
```

## Update Patterns

### 02-active-context.md (Update After Each Task)
**When**: After completing subtasks, hitting blockers, changing focus
**What**: Current session work, immediate next steps, recent changes, challenges

### 05-progress.md (Update Weekly/After Features)
**When**: Completing features, end of sprint, hitting milestones
**What**: Move tasks to completed, update metrics, add/remove blockers

### 03-system-patterns.md (Update When Discovering Patterns)
**When**: Creating reusable patterns specific to this project
**What**: Code examples, usage guidance, rationale, alternatives considered

## Important Rules

1. **ALWAYS** read both MCP servers at session start
2. **UPDATE** project-memory after every meaningful change
3. **ASK** before writing to team-memory (impacts all projects)
4. **KEEP** 02-active-context.md current (your working memory)
5. **USE** MCP tools, not direct file editing
6. **SEPARATE** global patterns (team) from project-specific (project)

## Quick Reference

| Need | Command |
|------|---------|
| Start session | Read team-memory + all project-memory files |
| During work | Update project-memory/02-active-context.md |
| Complete feature | Update 02-active-context.md + 05-progress.md |
| New pattern | Add to 03-system-patterns.md (or ask about team-memory) |
| Stuck/blocked | Note in 02-active-context.md challenges |
```

---

## Contributing

When adding to team memory:
1. Ensure the pattern applies to multiple projects
2. Document rationale and alternatives considered
3. Include code examples where applicable
4. Get team review before committing

---


## Support

For issues or questions:
- Check troubleshooting section above
- Review MCP logs
- Consult with your development team
