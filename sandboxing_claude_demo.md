# Sandboxing Claude Code

## Disclaimer
- This is my personal setup. I'm not sure it's the best.
- Not all config examples have been thoroughly tested.
- For legal reasons, this is not security advice.
- You will have to customize this to your specific machine, your workflows and the tools you use, this might take a while.

## Claude Code settings files
Claude Code merges settings from several files. Precedence, highest first:
1. Enterprise-managed settings (not covered here)
2. CLI args
3. Project local: `<project>/.claude/settings.local.json` (git-ignored by default)
4. Project shared: `<project>/.claude/settings.json` (committed)
5. User global: `~/.claude/settings.json` and `~/.claude/settings.local.json`

Rule of thumb:
- `settings.json`: shareable config. Allow/deny lists, hooks, sandbox rules.
- `settings.local.json`: personal or sensitive. Env vars with secrets, experimental tweaks. Git-ignored by default.
- Project-level: team conventions for that repo.
- User-level: your defaults across every project.

This talk focuses on user-level settings in `~/.claude/settings.json` and `~/.claude/settings.local.json` (or the Windows equivalent).

## What's the risk?
- Destructive operations on your local machine
- Destructive operations on services through CLI, MCPs etc
- Prompt injection
- Data exfiltration

## Goal
- Use Claude Code in a safer way
- Simplify security & permission management
- Make Claude Code more autonomous by reducing permission requests (for parallel work etc.)

## Allow & Deny lists
- Default way to control Claude permissions.
- This is where stuff goes when you click "yes and don't ask again".
- Mix of bash commands, tools (Read, WebFetch) and MCP commands (mcp__*)
- Example of deny & allow lists (yours is probably much bigger).
```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",
      "Bash(git status:*)",
      "Bash(ls:*)",
      "Bash(find:*)",
      "Bash(cat:*)",
      "Bash(grep:*)",
      "WebFetch(domain:docs.anthropic.com)"
    ],
    "deny": [
      "Bash(rm -rf /*:*)",
      "Read(.env)",
    ]
  },
```

## Why I don't like Deny & Allow list for Bash

**Allow lists are too annoying**
- You probably have 1000+ commands on your machine. Impossible to build a thorough list.
- Clicking "yes, don't ask again" grows your settings into de-facto god-mode over time.
- Claude doesn't know what's allow-listed. It might pick a different form of the same operation and trigger a prompt anyway (you allow `Bash(cat:*)`, Claude uses `less file.txt`).

**Deny lists aren't secure:**
- Too many ways to write effectively the same command.
- Denying `Read(.env)` doesn't stop:
    - `$ cat .env`
    - `$ less .env`
    - `$ head -n 9999 .env`
    - `$ python -c "print(open('.env').read())"`
    - `$ cp .env out.txt && cat out.txt`
- Still worth using for things like force-push to main, `terraform destroy`
- Deny list is more of a "please don't do that", than a proper lock.
- Hackers don't care about messing up your git history. Real risk is credential / data exfiltration.

**Takeaway:** don't trust regex to secure your machine.
*Insert Crowdstrike joke*

## The Alternative: Sandboxing
- Default sandbox built into Claude Code.
- You can make your custom sandbox setup using Docker / VMs etc. But more complicated.
- Check https://code.claude.com/docs/en/sandboxing for installing and turning on sandbox.

## What does Sandboxing actually let us do
- **Filesystem isolation**: control what files & folders Claude has read & write access to.
- **Network isolation**: control which domains Claude has access to. i.e: Which websites Claude can read & send data to.
- These are enforced at the OS level, not by pattern matching commands. They apply to all tools & commands.
- When a command is blocked by the sandbox, Claude can retry with `dangerously-disable-sandbox`, which triggers a permission prompt. Don't worry about it getting silently stuck.

## `autoAllowBashIfSandboxed`
- This is the whole point.
- If you manage security at the sandbox level, you can auto-allow any bash command.
- Probably eliminates >90% of permission requests. 

## What do we actually want to control?
- Destructive operations on your local machine
    -> What folders and files Claude has write access to
- Destructive operations on remote services through CLI, MCPs etc 
    -> What Claude can actually do on remote services (TODO: rephrase)
- Prompt injection
    -> Where Claude can receive data from
- Data exfiltration
    -> Where Claude can send data to
    -> what secrets and credentials Claude has access to.

## Preventing destructive operations on your local machine
Default Write permissions:
- Claude has write access in the current working directory and every sub-directory (i.e: wherever you're starting Claude Code from)

Recommendation:
- Use git, git is your undo button. Be careful with giving Claude write access to folders which aren't version-controlled.
- Use a whitelist for write access.
- Pay attention to which folder you start Claude in. Add a blacklist if you're worried about accidentally starting Claude in a folder you don't want it edit.
- Allow write access to the folder where you put your gitub repos, or working files
- You might have to give write access to folders used internally by tools (e.g: `"/Users/come.grant/.cache/gh"`)

```json
    "filesystem": {
      "allowWrite": [
        "/Users/come.grant/repos/cheffelo"
      ],
```

## Read acccess: Preventing Claude from reading secrets
- Use a blacklist for read access
- Deny read from files containing secrets and sensitive info.
```json
    "filesystem": {
      "allowWrite": [
        "/Users/come.grant/repos/cheffelo"
      ],
      "denyRead": [
        "/Users/come.grant/.ssh/",
        "/Users/come.grant/.databrickscfg",
        "/Users/come.grant/.zshrc",
        "/Users/come.grant/.bash_history",
        "/Users/come.grant/repos/cheffelo/**/.env",
        "/Users/come.grant/repos/cheffelo/**/.env.*",
        "/Users/come.grant/repos/cheffelo/**/*.pem",
        "/Users/come.grant/repos/cheffelo/**/*.key",
        "/Users/come.grant/repos/cheffelo/**/*.p12",
        "/Users/come.grant/repos/cheffelo/**/secrets/**",
        "/Users/come.grant/repos/cheffelo/**/credentials.json"
      ]
    }
  }
```

## Keeping CLI tools working
- All these files containing secrets are here for a reason, they are used by CLI tools
- Denying read to config files containing secrets used by CLI tools might break them (e.g: the Databricks CLI needs read access to `"/Users/come.grant/.databrickscfg"`)

As an alternative:
- Use MCPs (more on why MCPs are generally more secure later)
- Use the `"env"` block in your config. This passes environment variables specific to Claude Code.
- Almost all CLI tools use CLI flags > env vars > config files.
- So in this example, Claude uses the Databricks CLI with his specific env var, while my terminal uses the settings in `"/Users/come.grant/.databrickscfg"`
Note: Put secrets `settings.local.json` which is git ignored by default. Can use project-level settings if the tools are project-specific.
```json
"env": {
    "DATABRICKS_HOST": "https://xxxxxxxx.azuredatabricks.net",
    "DATABRICKS_CLIENT_ID": "sp-client-id-here",
    "DATABRICKS_CLIENT_SECRET": "the-actual-secret"
},
```

- This also lets you create a different user for Claude Code to use without affecting how you use the CLI.
- Claude can still access secrets set as environment variables in your shell. Not foolproof.
- There are better way to do secret management, but they get complicated.


## Part 1 done, now for the network
- Control what files Claude has write access to ✅
- Limit what secrets Claude has access to ✅
- Limit Claude's permissions on CLI tools (different user via env vars in `settings.local.json` + deny list) ✅

Next: Network Isolation
- Controlling what data Claude has access to (prompt injection, don't trust random people on the internet)
- Controlling where Claude can send data to (data exfiltration, aka: Prevent Claude from sending all your passwords to a random server out there)

## Network isolation
- List of allowed hosts (supports glob patterns `:*`).
- Applies to Claude's own network calls (Bash, WebFetch). **MCP tools bypass this** (more on that in the next slide).
- Unfortunately, can't have different lists for "trust to read from" vs "trust to send data to". One list for both.

My allowed domains:
```json
"network": {
    "allowedDomains": [
    "pypi.org",
    "files.pythonhosted.org",
    "api.linear.app",
    "hub.getdbt.com",
    "docs.databricks.com",
    "code.claude.com",
    "adb-xxxxxxxxx.azuredatabricks.net",
    "adb-xxxxxxxxx.azuredatabricks.net",
    "adb-xxxxxxxxx.azuredatabricks.net",
    ]
},
```
Not too happy about having pypi.org in there, but I haven't found a way around it yet.

## None of this applies to MCP tools
(and why that's a good thing)
- MCPs have a smaller list of tools, easier to have a complete deny / approve list.
- MCP tools run outside of the Network sandbox
- Good because you can use MCPs instead of allowing too broad hosts (github.com, api.github.com...)
- Most MCPs can run with different auth
- For example, I use the GH MCP with a fine-grained access token that only has required privileges to the `cheffelo/sous-chef` repo.


## Some extra stuff in CLAUDE.md
Tell Claude to avoid command patterns that always trigger a permission check, even when the underlying command is allow-listed:
- **Compound commands** like `cd foo && git status`. Do `cd foo` then `git status` on separate lines.
- **Backslash-escaped whitespace**. Use `cat "My Notes.md"` instead of `cat My\ Notes.md`. The matcher doesn't unescape.
- Generally: keep commands simple enough that the allowlist can recognize them.



## Auto mode
New feature, haven't tried it yet.
Idk how it compares to autoAllowBash

## Example config:
You'll have to customize this, but here's an example config just to show what goes where.

`~/.claude/settings.json`:
```json
{
  "permissions": {
    "allow": [
      "mcp__context7__*",
      "Bash(git status:*)",
      "Bash(ls:*)",
      "WebFetch(domain:docs.anthropic.com)"
    ],
    "deny": [
      "Bash(rm -rf /*:*)",
      "Bash(git push --force:*)",
      "Bash(terraform destroy:*)"
    ],
    "autoAllowBashIfSandboxed": true
  },
  "sandbox": {
    "filesystem": {
      "allowWrite": [
        "/Users/come.grant/repos/cheffelo"
      ],
      "denyRead": [
        "/Users/come.grant/.ssh/",
        "/Users/come.grant/.databrickscfg",
        "/Users/come.grant/.zshrc",
        "/Users/come.grant/.bash_history",
        "/Users/come.grant/repos/cheffelo/**/.env",
        "/Users/come.grant/repos/cheffelo/**/.env.*",
        "/Users/come.grant/repos/cheffelo/**/*.pem",
        "/Users/come.grant/repos/cheffelo/**/*.key",
        "/Users/come.grant/repos/cheffelo/**/secrets/**",
        "/Users/come.grant/repos/cheffelo/**/credentials.json"
      ]
    },
    "network": {
      "allowedDomains": [
        "pypi.org",
        "files.pythonhosted.org",
        "api.linear.app",
        "hub.getdbt.com",
        "docs.databricks.com",
        "code.claude.com",
        "adb-xxxxxxxxx.azuredatabricks.net"
      ]
    }
  }
}
```

`~/.claude/settings.local.json` (git-ignored, secrets go here):
```json
{
  "env": {
    "DATABRICKS_HOST": "https://xxxxxxxx.azuredatabricks.net",
    "DATABRICKS_CLIENT_ID": "sp-client-id-here",
    "DATABRICKS_CLIENT_SECRET": "the-actual-secret"
  }
}
```

- `permissions.allow` / `permissions.deny`: allow & deny lists
- `permissions.autoAllowBashIfSandboxed`
- `sandbox.filesystem.allowWrite`: stops destructive local ops
- `sandbox.filesystem.denyRead`: stops secret reads
- `sandbox.network.allowedDomains`: stops data exfil / prompt injection from random hosts
- `env` (in `settings.local.json`): Claude-specific credentials, different from your shell user


## Conclusion
- No single layer is foolproof. **Swiss cheese model** of security: stack imperfect layers, hope the holes don't line up.
- Iterate. It might take a while to get a good setup. Ask Claude to help you.

