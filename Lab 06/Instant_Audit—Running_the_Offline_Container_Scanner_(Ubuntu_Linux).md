# Instant Audit — Running the Offline Container Scanner (Ubuntu Linux)

**Target OS:** Ubuntu Linux 22.04 / 24.04 LTS

**Target IDE:** Cursor IDE (v3.1.2+)

**Audit Tool:** AI Security Labs Containerized Scanner (`coding-agent-security`)

**Audience:** Developers, DevOps Engineers, and Security Beginners

---

## 1. Overview

In traditional code editors, terminal commands run only when you type them. In Cursor IDE, the built-in AI Agent can write, edit, and propose running terminal commands automatically.

If you open an untrusted project or if the AI encounters a malicious prompt (prompt injection), the agent can be tricked into running harmful commands on your computer.

This lab teaches you how to audit your Cursor setup on Ubuntu using a safe, offline, throwaway container scanner. You will:

1. Check what Cursor's default settings look like (**Track A: Baseline Audit**).
2. Create a test project with common risky settings (**Track B: Controlled Audit**).
3. Run the container scanner to detect the risks.
4. Fix the vulnerabilities and verify that the audit passes.

<!-- ```
┌────────────────────────────────────────────────────────────────────────┐
│                          LAB WORKFLOW OVERVIEW                         │
├────────────────────────────────────────────────────────────────────────┤
│ 1. Check Ubuntu Prerequisites (Docker & Git)                           │
│ 2. Track A: Run Baseline Audit on a Fresh / Empty Folder               │
│ 3. Track B: Create a Test Project with Risky Settings (Simulated)      │
│ 4. Run Container Scanner & Observe FAIL Findings                       │
│ 5. Apply Security Fixes (Remediation)                                  │
│ 6. Re-run Scanner & Confirm All Checks PASS                            │
└────────────────────────────────────────────────────────────────────────┘
``` -->
![Security Scanning Workflow](/SecurityScanningWorkflow.drawio.svg)

---

## 2. Learning Objectives

By completing this lab, you will be able to:

- Understand where Cursor stores its configuration files on Ubuntu (`~/.cursor/` and `~/.config/Cursor/`).
- Understand why Cursor's default setting for **Workspace Trust** needs to be turned on.
- Run the official offline Docker container scanner with zero network access (`-network none`).
- Understand `PASS`, `WARN`, and `FAIL` results in the audit report.
- Identify risky developer mistakes: allowlisting terminal interpreters (`python`, `bash`), unpinned MCP servers, and exposed `.env` files.
- Fix insecure configurations using clean, production-ready templates.

---

## 3. Prerequisites

You only need a standard Ubuntu machine with Docker and Git.

### System Requirements

- **Operating System:** Ubuntu 22.04 LTS or 24.04 LTS (x86_64 or ARM64).
- **RAM:** 4 GB minimum.
- **Storage:** 2 GB free disk space.
- **Privileges:** Standard user with permission to run Docker.

### Verify Prerequisites on Ubuntu

Run these commands in your Ubuntu terminal:

```bash
# 1. Verify Git is installed
git --version

# 2. Verify Docker is running
docker --version

# 3. Verify Cursor
cursor --version
```
- **Expected Output:**
![check all version](/SS/check%20all%20version.png)
- **What to do if Docker gives a permission error:**
If you see `permission denied while trying to connect to the Docker daemon`, add your user to the `docker` group:
    
    ```bash
    sudo usermod -aG docker $USER
    newgrp docker
    ```
    

---

## 4. Key Security Concepts (Explained Simply)

### 1. User Settings vs. Project Settings

- **User-Level (`~/.cursor/` and `~/.config/Cursor/`):** Applies to all projects on your computer.
- **Project-Level (`<project-folder>/.cursor/`):** Bundled inside a specific repository.
- **The Dangerous Merger Rule:** In Cursor, project settings **add to** your user settings. If an untrusted project includes `.cursor/permissions.json` with dangerous commands, Cursor adds them to your active session!

### 2. Workspace Trust

- **Default in Cursor:** **Disabled by default.**
- **What this means:** When you open any folder (even one downloaded from an unknown person), Cursor trusts it immediately. Any bundled hooks or MCP tools will run without asking for permission.
- **The Fix:** Turn on Workspace Trust so Cursor asks: *"Do you trust this folder?"* before enabling agent automations.

### 3. Terminal Allowlist (`terminalAllowlist`)

- You can configure commands that the agent can run automatically without prompting you.
- **Safe:** Read-only commands like `git status`, `git diff`, `ls`, `pwd`.
- **Dangerous:** Interpreters and shells like `python`, `node`, `bash`, `sh`, `rm`, `curl`. If `python` is allowlisted, the agent can execute any Python code without your approval.

### 4. How the Container Scanner Works Safely

- **Offline (`-network none`):** The container cannot connect to the internet. It cannot leak your code or credentials.
- **Read-Only (`:ro`):** Your files are mounted read-only. The scanner cannot modify or delete anything on your computer.
- **Throwaway (`-rm`):** The container is deleted immediately after the scan finishes.

---

## 5. Configuration Discovery: Where Do Files Live?

A common myth is that Cursor creates a `.cursor/` folder as soon as you open an empty project. **It does not.**

| Event | What Exists in `~/.cursor/` | What Exists in Project Folder |
| --- | --- | --- |
| **Fresh Install of Cursor** | Empty or non-existent | Empty project directory |
| **First Launch of Cursor** | Created, but no security files | Empty project directory |
| **Open an Empty Folder** | Unchanged | **No `.cursor/` directory is created** |
| **Add a Project Rule or MCP Tool** | Unchanged | `.cursor/` directory is created on disk |

> [!NOTE]
The absence of `.cursor/` on a fresh install does **not** mean your system is broken. It simply means Cursor is using its built-in defaults.
> 

---

## 6. Step-by-Step Practical Lab

Follow these steps directly on your Ubuntu machine.

<!-- ```
[Screenshot SS-01: Ubuntu Terminal & Docker Verification]
* Title: Docker Engine and Git Verification
* What must be visible: Output of git --version and docker info on Ubuntu
* Associated Step: Step 1
* Evidence: Establishes host readiness for offline container scanning
``` -->

### Step 1: Download the Scanner Toolkit

Clone the open-source audit repository from AI Security Labs into your home directory:

```bash
# Clone the repository
git clone <https://github.com/aisecuritylabs-org/coding-agent-security.git> ~/coding-agent-security

# Verify the audit files exist
ls -la ~/coding-agent-security/cursor/audit/
```

- **Explanation:** Downloads the Dockerfile, runner script, and audit checks.
- **Expected Output:** You should see `Dockerfile`, `audit.sh`, and `run.sh`.
![Security Audit Files](/SS/Security%20Audit%20files.png)

---

### Step 2: Track A — Baseline Audit (Fresh / Empty Environment)

First, audit a completely clean, empty folder to see what an unconfigured Cursor environment looks like.

```bash
# Create an empty test directory
mkdir -p ~/cursor-baseline-test
cd ~/cursor-baseline-test

# Run the audit script
bash ~/coding-agent-security/cursor/audit/run.sh
```

<!-- ```
[Screenshot SS-02: Track A Baseline Scan Terminal Output]
* Title: Track A Baseline Container Scan Output
* What must be visible: Console showing PASS items and WARN for Workspace Trust disabled
* Associated Step: Step 2
* Evidence: Documents default out-of-the-box security posture of fresh Cursor IDE
``` -->

![Baseline Audit](/SS/running_baseline_audit.png)

- **What happens:** The tool builds a tiny, secure Alpine container (if not already cached) and inspects your environment.
- **Interactive Prompt:** At the end, it will ask:
`Save a report with how-to-fix steps? [h]tml, [t]ext, [c]sv, [j]son, [n]o (default n):`
Press `n` and hit Enter (we will save a full report in Track B).
- **Baseline Observation:**
    - You will see **`WARN [1] Workspace Trust is off (the default)`**.
    - You will see **`0 FAIL`**.
    - **Takeaway:** An unconfigured Cursor setup has no critical failures, but Workspace Trust is off by default.

---

### Step 3: Track B — Create a Test Project with Common Risky Settings

Now create a test project that simulates common developer mistakes:

1. An unignored `.env` file containing secrets.
2. A `.cursor/permissions.json` file that allowlists `python`.
3. A `.cursor/mcp.json` file with an unpinned script.

```bash
# 1. Create a dedicated project directory
mkdir -p ~/cursor-risky-project
cd ~/cursor-risky-project

# 2. Initialize a git repo
git init

# 3. Create a dummy Python file
cat << 'EOF' > app.py
print("Hello from test app")
EOF

# 4. Create a mock .env file (DO NOT use real credentials!)
cat << 'EOF' > .env
DATABASE_URL=postgres://test_user:test_password@localhost:5432/test_db
API_SECRET_KEY=mock_secret_key_12345
EOF

# 5. Create .cursor folder
mkdir -p .cursor

# 6. Create a RISKY permissions.json (allowlists python and wildcard MCP)
cat << 'EOF' > .cursor/permissions.json
{
  "terminalAllowlist": [
    "python",
    "git status",
    "ls"
  ],
  "mcpAllowlist": [
    "*:*"
  ]
}
EOF

# 7. Create a RISKY mcp.json (unpinned local script)
cat << 'EOF' > .cursor/mcp.json
{
  "mcpServers": {
    "helper": {
      "command": "node",
      "args": ["./tools/helper.js"]
    }
  }
}
EOF
```
![Files of controlled audit](/SS/files%20on%20controlled%20audit%20project.png)


---

### Step 4: Run the Audit on the Risky Project

Execute the scanner against your simulated project:

```bash
# Make sure you are in the test project directory
cd ~/cursor-risky-project

# Run the scanner
bash ~/coding-agent-security/cursor/audit/run.sh
```

![Controlled Audit](/SS/running%20controlled%20audit_1.png)

<!-- ```
[Screenshot SS-04: Track B Audit Output with FAILs]
* Title: Track B Security Audit Output Displaying FAIL Findings
* What must be visible: Console showing FAIL for terminalAllowlist (python) and FAIL for mcpAllowlist (*:*)
* Associated Step: Step 4
* Evidence: Proves scanner accurately detects project-level misconfigurations
``` -->

- **Interactive Prompt:**
When prompted:
`Save a report with how-to-fix steps? [h]tml, [t]ext, [c]sv, [j]son, [n]o (default n):`
Press `h` and hit Enter to generate an HTML report.
- **Check the Exit Code:**
The exit code is **`1`** because the scanner found critical `FAIL` items!
    
    ```bash
    echo $?
    ```
    

---

### Step 5: Understand Some Example Why These Findings Failed

| Finding | Why It Failed | The Real-World Risk |
| --- | --- | --- |
| **`FAIL: terminalAllowlist: python`** | `python` is in the allowlist. | An attacker can inject a prompt telling the agent to run `python -c "evil code"`. The agent runs it immediately without asking you! |
| **`FAIL: mcpAllowlist: *:*`** | `*:*` is a wildcard allowlist. | Gives every tool on every MCP server permission to run automatically. |
| **`FAIL: .env is not ignored`** | `.env` is not in `.gitignore`. | Cursor will index `.env`, include secrets in LLM context prompts, or commit them to GitHub. |
| **`WARN: Workspace Trust is off`** | Trust is disabled in settings. | Any folder you open is trusted automatically. |

---

### Step 6: Fix the Issues (Remediation)

Now apply the proper security fixes to your project and user settings.

#### Fix 1: Add `.env` to `.gitignore`

```bash
cat << 'EOF' > .gitignore
.env
.env.*
*.key
EOF

# Verify Git ignores .env now
git status
```

#### Fix 2: Clean up `.cursor/permissions.json`

Remove `python` and wildcard `*:*`. Only allow safe, read-only commands:

```bash
cat << 'EOF' > .cursor/permissions.json
{
  "terminalAllowlist": [
    "git status",
    "git diff",
    "git log",
    "ls",
    "pwd"
  ],
  "mcpAllowlist": [],
  "autoRun": {
    "allow_instructions": [],
    "block_instructions": [
      "Anything that reads .env files, ~/.ssh, ~/.aws or other credential files should go through approval.",
      "Any command that deletes files, pushes to git, or deploys should go through approval."
    ]
  }
}
EOF
```

#### Fix 3: Clean up `.cursor/mcp.json`

Remove unpinned scripts or restrict them:

```bash
cat << 'EOF' > .cursor/mcp.json
{
  "mcpServers": {}
}
EOF
```

#### Fix 4: Enable Workspace Trust in User Settings

On Ubuntu, Cursor stores user settings in `~/.config/Cursor/User/settings.json`.

```bash
# Ensure the directory exists
mkdir -p ~/.config/Cursor/User

# Enable Workspace Trust (creates or appends to settings.json)
python3 -c "
import json, os
p = os.path.expanduser('~/.config/Cursor/User/settings.json')
data = {}
if os.path.exists(p):
    try:
        with open(p, 'r') as f: data = json.load(f)
    except: data = {}
data['security.workspace.trust.enabled'] = True
with open(p, 'w') as f: json.dump(data, f, indent=2)
print('Workspace Trust enabled in:', p)
"
```

#### Fix 5: Set Up a Safe User-Level Sandbox

Create `~/.cursor/sandbox.json` to lock down network and filesystem access:

```bash
mkdir -p ~/.cursor
cat << 'EOF' > ~/.cursor/sandbox.json
{
  "type": "workspace_readwrite",
  "networkPolicy": {
    "default": "deny",
    "allow": [
      "registry.npmjs.org",
      "pypi.org",
      "files.pythonhosted.org"
    ]
  },
  "disableTmpWrite": false
}
EOF
```

---

### Step 7: Re-Run the Audit and Verify PASS

Re-run the audit against your remediated project:

```bash
cd ~/cursor-risky-project
bash ~/coding-agent-security/cursor/audit/run.sh
```
![Controlled audit 2](/SS/running%20controlled%20audit_2.png)

<!-- ```
[Screenshot SS-05: Re-Audit Terminal Output with 0 FAILs]
* Title: Re-Audit Terminal Output Showing Passing Status
* What must be visible: Console showing 0 FAILs and exit code 0
* Associated Step: Step 7
* Evidence: Confirms successful remediation of all security findings
``` -->

- **Interactive Prompt:** Press `n` or choose a report format.
- **Check the Exit Code:**
    
    ```bash
    echo $?
    ```
    
- **Expected Result:**
    - **0 FAIL findings.**
    - Exit code is **`0`**!
    - Your Cursor workspace on Ubuntu is now securely configured.

---

<!-- ### Step 8: View Your Generated HTML Audit Report

The scanner saves all generated reports in `~/cursor-audit-reports/` in your Ubuntu home directory (never inside your project).

```bash
# List your generated reports
ls -la ~/cursor-audit-reports/

# Open the latest HTML report in your default Ubuntu browser (e.g., Firefox or Chrome)
xdg-open ~/cursor-audit-reports/cursor-audit-*.html 2>/dev/null || echo "Open the file manually in your browser."
```

```
[Screenshot SS-06: HTML Audit Report Rendered in Browser]
* Title: Rendered Interactive HTML Audit Report Artifact
* What must be visible: Audit report displayed in browser showing clean passing checks
* Associated Step: Step 8
* Evidence: Verifies compliance-ready audit documentation
``` -->

### Step 8: View Your Generated Text Audit Reports

The scanner saves all generated reports in `~/cursor-audit-reports/` in your Ubuntu home directory (never inside your project).

```bash
# List all generated audit reports
ls -lah ~/cursor-audit-reports/
```

This command displays all generated audit report files, including their names, sizes, and last modification dates.

---

## 7. Security Verification Checklist (Ubuntu)

Use this simple reference table to verify your settings anytime:

| Check | What to Look For | Secure Setting | Insecure Setting |
| --- | --- | --- | --- |
| **Workspace Trust** | `~/.config/Cursor/User/settings.json` | `"security.workspace.trust.enabled": true` | `false` or missing |
| **Terminal Allowlist** | `.cursor/permissions.json` | Only read-only commands (`git status`, `ls`) | Interpreters (`python`, `bash`, `node`) |
| **MCP Allowlist** | `.cursor/permissions.json` | `mcpAllowlist: []` (empty or specific tools) | Wildcards (`*:*`) |
| **Environment Secrets** | `.gitignore` | `.env` is listed and ignored | `.env` tracked by Git |
| **Sandbox Policy** | `~/.cursor/sandbox.json` | `"type": "workspace_readwrite"`, `default: "deny"` | `"type": "insecure_none"` |
| **CLI Config** | `~/.cursor/cli-config.json` | Explicit deny: `Read(**/.env*)` | `.env` reads permitted |

---

## 8. What the Scanner Can and Cannot Do

To maintain a professional security posture, remember what automated scanning does and does not cover:

### What the Scanner DOES Check:

- On-disk configuration files (`sandbox.json`, `permissions.json`, `mcp.json`).
- Whether `.env` is ignored by Git.
- Whether shell startup files (`~/.bashrc`) contain dangerous bypass aliases (`-yolo`, `-force`).
- Whether instruction rules files (`.cursorrules`) contain hidden Unicode trickery.

### What the Scanner CANNOT Check:

- **The active GUI Run Mode:** The scanner cannot see which dropdown item you clicked in the running Cursor window (e.g., "Run Everything" vs. "Auto-review").
- **Your behavior:** If Cursor prompts you: *"Run this suspicious command?"* and you click "Approve", the scanner cannot stop you.
- **Clean Exit Code (0) does not mean 100% immune:** An exit code of `0` means your configuration has no known vulnerabilities. Always inspect unfamiliar code before trusting a repository!

---

## 9. Cleaning Up After the Lab

Once you have finished the lab, you can safely remove the temporary test folders:

```bash
# Remove temporary test folders
rm -rf ~/cursor-baseline-test
rm -rf ~/cursor-risky-project

# Keep your report artifacts in ~/cursor-audit-reports for reference!
```

---

## 10. References & Further Reading

- **Official Cursor Documentation:** docs.cursor.com
- **Cursor Sandbox Reference:** docs.cursor.com/reference/sandbox
- **Cursor Permissions Reference:** docs.cursor.com/reference/permissions
- **AI Security Labs Cursor Audit Tool:** aisecuritylabs.org/resources/cursor-audit/
- **AI Security Labs Coding Agent Security Repo:** github.com/aisecuritylabs-org/coding-agent-security
- **OWASP Top 10 for LLM Applications (2025):** owasp.org