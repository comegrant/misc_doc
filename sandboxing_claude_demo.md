# Sandboxing Claude Code

## Disclaimer
- This is my personal setup. I'm not sure it's the best.
- Not all config examples have been thoroughly tested.
- For legal reason, this is not security advice.
- You will have to customize this to your specific machine, your workflows and the tools you use, this might take a while.

## Layers of Claude settings (TODO)
- Global and project settings
- How do different levels of settings interact
- `settings.json` vs `settings.local.json`
- Only talking about user settings in `~/.claude/settings.json`, `~/.claude/settings.local.json` or whatever the windows equivalent is. 

## Goal
- Use Claude Code in a safer way
- Simplify security & permission management
- Make Claude Code more autonomous by reducing permission requests (for parallel work etc.)

## What's the risk?
- Destructive operations on your local machine
- Destructive operations on services through CLI, MCPs etc
- Prompt injection
- Data exfiltration

## Allow & Deny lists
- Default way to control Claude permissions.
- This is where stuff goes when you click "yes and don't ask again".
- Mix of bash commands, tools (Read, WebFetch) and MPC commands (mcp__*)
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
      "WebFetch(domain:docs.anthropic.com)",
    ],
    "deny": [
      "Bash(rm -rf /*:*)",
      "Read(.env)",
    ]
  },
```

## Why I don't like allow lists
- You probably have >1000 commands on your computer, impossible to make a thorough allow / deny list.
- If you click "yes, and don't ask again for ...", your settings get super messy and Claude Code eventually gets god powers on your machine.
- Claude is not aware (by default) of what Bash commands and Tools are allowed. Even if your permission allow Claude to perform an operation with specific commands / syntax, it might choose a different way to write the command that will trigger a permission check. e.g: You allow `Bash(cat:*)` but Claude uses `$ less file.txt`.
- Just too many permission checks.

## Why Deny lists aren't enough
- Some basic operations (e.g: reading a file) can be done with many different commands. Making a deny list really easy to bypass.
e.g: If you try to prevent Claude from reading a secret file by denying `Read(.env)`, Claude can still read the file with any of these commands:
    - `$ cat .env`
    - `$ less .env`
    - `$ nl .env`
    - `$ more .env`
    - `$ head -n 9999 .env`
    - `$ tail -n +1 .env`
    - `$ python -c "print(open('.env').read())"`
    - `$ cp .env out.txt && cat out.txt`
- A Deny list is more of a "Please don't do this" than an actual lock. Good to have but not foolproof.
- Still good to have to prevent stuff like force push to main, terraform destroy etc.
- Hackers probably don't care about messing up your git history or infrastructure. The risk is more leaked credentials etc.

## Deny & Allowlist (conclusion)
Don't trust regex to secure your machine?
*Insert Crowdstrike joke*

## The Alternative: Sandboxing
- Default sandbox built into Claude Code.
- You can make your custom sandbox setup using Docker / VMs etc. But more complicated.
- Check https://code.claude.com/docs/en/sandboxing for installing and turning on sandbox.

## What does Sandboxing actually let us do
- Filesystem isolation: Control what files & folders Claude has read & write access to
- Network isolation: Control which domains Claude has access to
- These are actually enforced, not regex-ing commands. They apply to all tools & commands.
- `"autoAllowBashIfSandboxed"` Auto-allow bash commands if Claude is running in the sandbox
- When a command is blocked by the Sandbox, Claude can retry with `dangerously-disable-sandbox`, which will prompt a permission request. I was worrying about Claude silently getting blocked by the sandbox, but it's been working well.

## What do we actually want to control?
- Destructive operations on your local machine
    -> What folders and files Claude has write access to
- Destructive operations on remote services through CLI, MCPs etc 
    -> What Claude can actually do on remote services (TODO: rephrase)
- Prompt injection
    -> Where Claude can receive data from
    -> Ingress control is slightly impractical, you want to connect Claude to stuff (Medium, Slack, Linear, Github etc.)
- Data exfiltration
    -> Where Claude can send data to
    -> what secrets and credentials Claude has access to.

## Preventing destructive operations on your local machine
Default Write permissions:
- Claude has write access in the current working directory and every sub-directory (i.e: wherever you're starting Claude Code from)

Recommendation:
- Use git, git is your undo button. Do not give Claude write access to folders which aren't version-controlled.
- Use a whitelist for write access
- Allow write access to the folder where you put your gitub repos, or working files
- Be careful which folder you start Claude Code in.
- You might have to give write access to folders used internally by tools (e.g: `"/Users/come.grant/.cache/gh"`)

```json
    "filesystem": {
      "allowWrite": [
        "/Users/come.grant/repos/cheffelo"
      ],
```

## Preventing Claude from reading secrets
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
        "/Users/come.grant/Documents/cheffelo/**/.env",
        "/Users/come.grant/Documents/cheffelo/**/.env.*",
        "/Users/come.grant/Documents/cheffelo/**/*.pem",
        "/Users/come.grant/Documents/cheffelo/**/*.key",
        "/Users/come.grant/Documents/cheffelo/**/*.p12",
        "/Users/come.grant/Documents/cheffelo/**/secrets/**",
        "/Users/come.grant/Documents/cheffelo/**/credentials.json"
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
Note: Put secrets `settings.local.json` which is git ignored by default. Can use project-level settings here.
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


## Almost there!
- Control what files Claude has write access to :white-checkmark:
- Limit what secrets stuff Claude has access to. :white-checkmark:
- Limit the permissions of Claude on CLI tools (Giving Claude a different user using env variables in your `settings.local.json` + deny list)

Next: Network Isolation
- Controlling what data Claude has access to (prompt injection, don't trust random people on the internet)
- Controlling where Claude can send data to (data exfiltration, aka: Prevent Claude from sending all your passwords to a random server out there)

## Network isolation
- List of allowed hosts (supports glob patterns `:*`)
- Unfortunately, can't have different lists from websites you trust to read from vs websites you trust to send data to.
My allowed domain:
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
- Tell Claude not use command patterns that always trigger a permissions check like compound `cd && git` commands or backslash-escaped whitespace (use `cat "My Notes.md"` instead of  `cat My\ Notes.md`)

- Guidance from your global CLAUDE.md: no compound cd && git commands, no
backslash-escaped spaces (quote instead), prefer gh CLI over GitHub MCP, etc.
  # ❌ PROMPTS — backslash-escaped whitespace trips the matcher
  #    even though `cat` is fully allowed
  cat My\ Notes.md



## Auto mode (VERIFY)
What stays enforced in auto mode

- Sandbox — filesystem and network restrictions remain active at the OS
level. Your read/write deny lists and allowed network hosts are not
bypassed.
- Deny rules — any explicitly denied tools or bash commands in your
settings are always respected, regardless of mode.
- Narrow allow rules — specific allowlist entries like Bash(npm test) carry
over into auto mode.

TODO: What's the rec between autoAllowBashIfSandboxed and Auto mode?
-> Default to autoAllowBash, maybe try turning on auto mode selectively

## Example config (clean up, go over and explain)


## Conclusion
- hard to make a foolproof system
- swiss cheese model of security.

