# Architecture Decision Records

## ADR-001: Pure Prompt Skill (No Code)

**Status**: Accepted
**Date**: 2026-02-16

**Context**: The original `skill-security-auditor` prototype used Python with 350+ regexes for pattern matching, made external Opus API calls, and had fragile JSON-from-markdown parsing. Two vendor skills (droid-tings/security-auditor, antigravity/security-auditor) also existed as separate installable packages.

**Decision**: Pure SKILL.md + reference files. No TypeScript, no Python, no scripts.

**Rationale**:
- Claude can read files, parse YAML frontmatter, identify threats, and reason about alignment natively
- Semantic understanding > regex pattern matching (detects novel obfuscation patterns regexes have never seen)
- External API calls are unnecessary when the skill runs inside Claude Code, which IS the model
- Eliminates dependency management, venvs, and build steps

**Trade-off**: Loses deterministic CI/CD scanning. Acceptable because [pors/skill-audit](https://github.com/pors/skill-audit) fills that niche with semgrep/shellcheck/trufflehog and SARIF output. The two tools are complementary: skill-audit for automated CI, this skill for semantic human review.

---

## ADR-002: Absorb Vendor Skills Into References

**Status**: Accepted
**Date**: 2026-02-16

**Context**: Two vendor skills audited application code for security issues:
- **droid-tings/security-auditor** (ovachiever): OWASP Top 10 patterns, dependency scanning
- **antigravity/security-auditor** (sickn33): Threat modeling (STRIDE/PASTA), DevSecOps

Both installed to `~/.agents/skills/security-auditor/`, causing name collisions with each other and with this skill.

**Decision**: Absorb their domain knowledge into reference files with attribution. No external skill dependencies. Single skill replaces all three.

**Rationale**:
- Neither vendor skill understood the agent skills format or intent-alignment analysis
- Both audited application code (SQL injection, XSS), not skill intent
- Their domain knowledge (OWASP patterns, threat modeling) is already in Claude's training data -- the reference files serve as activation/checklist, not new knowledge
- Name collisions made coexistence impractical
- One well-crafted prompt > three poorly integrated ones

**Attribution**: Both are credited in SKILL.md and in the reference files where their knowledge was absorbed.

---

## ADR-003: Sandboxed Subagent as Default Execution Architecture

**Status**: Accepted
**Date**: 2026-02-16

**Context**: The skill must read files from potentially malicious skills. Those files may contain prompt injection designed to hijack Claude's reasoning. In autonomous mode (increasingly common as users seek to escape the human-in-the-loop bottleneck), a hijacked agent could execute arbitrary commands without approval.

**Decision**: The entire audit runs inside a sandboxed subagent. The main agent never reads any file in the target skill directory.

**Architecture**:
```
Main Agent (unrestricted, never reads hostile content)
  1. Resolve target directory (clone if GitHub URL)
  2. Run dependency scans programmatically (npm audit, pip-audit)
  3. Spawn audit subagent (Explore type, model: opus)
     |
Subagent (Read/Glob/Grep only, inoculated)
  1. Read all skill files
  2. Run 5-step analysis with reference checklists
  3. Return structured report
     |
Main Agent
  4. Present report to user
```

**Rationale**:
- Hostile content never enters the main agent's context window, so the main agent cannot be hijacked
- Even if the subagent is fully hijacked by prompt injection, it cannot: write files, execute commands, access the network, or spawn further agents
- The worst case is a false audit report, not actual system damage
- False reports are detectable by the user reading them; unauthorized command execution is not
- Shifts the security burden from user judgment (fallible, especially in autonomous mode) to architectural constraints (deterministic)

**Alternatives considered**:
- *Direct reading with inoculation only*: Relies entirely on Claude's ~99% prompt injection resistance. The 1% failure case is catastrophic in autonomous mode.
- *Split reading/analysis between agents*: The main agent would need to trust the subagent's inventory without being able to verify it. Rejected because blind trust in an intermediary is worse than direct reading for a security tool.
- *Full subagent (entire audit in subagent)*: This is what we chose. The subagent does both reading AND analysis, so it can apply semantic understanding to raw content. The main agent's clean context ensures the report presentation is trustworthy.

**Trade-offs**:
- Subagent tool restrictions are soft (instructional), not hard (enforced). Mitigated by OS-level sandbox (Layer 3 hardening) for users who want deterministic guarantees.
- Subagent's return value enters main agent context and could theoretically carry injection. Mitigated by the fact that it's a structured report, not raw file content, and the main agent is primed to present it rather than follow instructions within it.

---

## ADR-004: Root Agent Treats Target Directory as Radioactive

**Status**: Accepted
**Date**: 2026-02-16

**Context**: During testing, the root agent attempted to unzip an archive file in the target skill directory in order to make its contents available for the subagent to inspect. This violated the isolation boundary -- interacting with hostile files (unzipping, extracting, executing) can trigger embedded payloads even without Claude reading the content.

**Decision**: The main agent is explicitly forbidden from reading, opening, unzipping, decompressing, extracting, or executing ANY file in the target skill directory. Its only permitted interactions are:
- Checking file existence via Glob (returns names, not content)
- Running `npm audit` / `pip-audit` in the directory (programmatic tools that don't inject content into Claude's context)
- Passing the directory path string to the subagent

**Rationale**:
- Archive files can contain path traversal exploits or trigger decompression bombs
- Even "peeking" at a file to decide whether to pass it to the subagent puts hostile content in the main agent's context
- The subagent should discover and flag archive files as findings ("[ARCHIVE -- not extracted]") rather than having them extracted
- Clean separation: main agent handles infrastructure, subagent handles all content inspection

---

## ADR-005: Defense-in-Depth Hardening Layers

**Status**: Accepted
**Date**: 2026-02-16

**Context**: No single defense against prompt injection is sufficient. Claude's ~1% attack success rate (Anthropic's benchmark) is state-of-the-art but not zero. For a security tool that reads adversarial content by design, layered defenses are essential.

**Decision**: Six defense layers, each independent, each surviving different failure modes:

| Layer | Type | Survives Hijack? | Implementation |
|-------|------|------------------|----------------|
| Subagent isolation | Architectural | Yes | Main agent never reads hostile content |
| Tool restrictions | Instructional | Soft | Subagent told to use only Read/Glob/Grep |
| Inoculation | Prompt-based | Probabilistic (~99%) | Explicit warning at start of subagent prompt |
| Lasso hooks | Pattern-based | Yes (external) | PostToolUse hook scans for 50+ injection patterns |
| Haiku classifier | LLM-based | Yes (separate model) | PostToolUse prompt hook classifies content |
| OS sandbox | Kernel-level | Yes (enforced by OS) | bubblewrap (Linux) / Seatbelt (macOS) |

**Rationale**:
- Layers 1 and 6 are deterministic -- they work regardless of whether prompt injection succeeds
- Layers 2 and 3 are probabilistic -- they reduce attack success rate but can be overcome
- Layers 4 and 5 are detection-based -- they alert on known patterns and suspicious content
- Only Layer 1 (subagent isolation) is mandatory in the skill. Layers 2-3 are built into the skill's instructions. Layers 4-6 are recommended hardening the user can opt into.

**Key insight**: The fundamental unsolved problem is that prompt injection cannot be eliminated at the model level. Defense must be architectural (limit what can happen if injection succeeds), not just model-level (hope injection doesn't succeed). The subagent pattern is the most important layer because it's the only one that provides a hard guarantee about blast radius.

---

## ADR-006: Require Opus for Audit Subagent

**Status**: Accepted
**Date**: 2026-02-16

**Context**: Security analysis requires detecting subtle capability-intent misalignment, recognizing novel obfuscation patterns, and resisting prompt injection in hostile content. These tasks are disproportionately affected by model capability.

**Decision**: The skill instructs the main agent to spawn the audit subagent with `model: "opus"` regardless of what model the main agent is running on. If Opus is unavailable, warn the user.

**Rationale**:
- Intent-alignment analysis is nuanced reasoning, not pattern matching -- it benefits significantly from the most capable model
- Prompt injection resistance correlates with model capability (~1% ASR for Opus vs higher for less capable models)
- The audit subagent is a single, bounded task -- the cost of one Opus subagent call is modest compared to the security value
- The Task tool's `model` parameter makes this trivially implementable
- Users running Sonnet or Haiku as their default model still get Opus-quality security analysis

**Trade-off**: Higher per-audit cost. Acceptable -- skills are high-leverage, high-risk; the investment in thorough analysis is justified.

---

## ADR-007: Intent-Aligned Security (Not Least-Privilege)

**Status**: Accepted
**Date**: 2026-02-16

**Context**: Traditional security applies the Principle of Least Privilege -- minimize permissions to the minimum necessary. In the agent skills ecosystem, this would mean rejecting any skill with broad capabilities.

**Decision**: Apply intent-aligned security instead. The security question is not "does this skill have dangerous capabilities?" but "do its capabilities align with its stated intent?"

**Rationale**:
- Agent skills legitimately need powerful capabilities. A deployment tool needs cloud credentials. A cleanup tool needs destructive operations. A security auditor needs broad file access. Rejecting all powerful skills defeats the purpose of having skills.
- The real threat is misalignment: capabilities that don't match stated intent indicate hidden behavior, deception, or malice.
- A skill that says "I delete node_modules" and actually deletes node_modules is honest and should be approved (with safeguards). A skill that says "I format poetry" but reads SSH keys is deceptive and should be rejected.
- This framing enables a four-tier verdict (APPROVE / APPROVE WITH SAFEGUARDS / NEEDS REVIEW / REJECT) that acknowledges legitimate power while catching deception.

**Inspiration**: This concept appeared in the original prototype's design. The redesign preserves it as the core philosophy while discarding the over-engineered implementation.

---

## ADR-008: Reference Files as Activation Checklists

**Status**: Accepted
**Date**: 2026-02-16

**Context**: The original prototype maintained a YAML database of 350+ regex patterns for threat detection. Two vendor skills contributed additional pattern databases (OWASP, STRIDE/PASTA).

**Decision**: Replace regex databases with concise reference files (threat-categories.md, intent-alignment-guide.md) that serve as checklists and activation reminders, not detection engines.

**Rationale**:
- Claude's semantic understanding is the detection engine. Tell it WHAT to look for, not HOW to search.
- A checklist of 11 threat categories with examples serves as a reminder -- "did you check for data exfiltration? credential access? obfuscation?" -- that activates knowledge Claude already has.
- The intent-alignment guide maps skill categories to expected capability profiles, providing calibration ("a formatter shouldn't need network access") without rigid rules.
- Reference files are loaded via "MANDATORY -- READ ENTIRE FILE" triggers during Step 4 (alignment analysis), not at skill load time, following progressive disclosure.
- Total reference content is ~230 lines across two files -- well under the context budget for a security-critical analysis.

**Trade-off**: Non-deterministic. The same skill audited twice might produce slightly different reports. Acceptable because the goal is human review, not CI/CD gating (which is skill-audit's niche).
