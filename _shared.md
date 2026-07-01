---
name: _shared
description: "Internal — not a command. Base execution rules, MCP policies, and verification protocols shared by all prd-* commands."
---

<!-- Internal shared rules. Referenced by all prd-* commands. Not a user-invokable command. -->

<shared_execution>
  <!-- Every prd-* command runs under these settings unless it adds extras inline. -->
  <follow_structure>strict</follow_structure>
  <treat_tags_as_semantic>true</treat_tags_as_semantic>
  <do_not_skip_phases>true</do_not_skip_phases>
  <do_not_assume>true</do_not_assume>
</shared_execution>

<shared_mcp_rules>

  <semble_first priority="critical">
    mcp__semble is MANDATORY for all codebase research — call mcp__semble__search before any
    mcp__octocode__localSearchCode or mcp__serena__find_symbol. If semble is unavailable or errors,
    fall back to mcp__octocode__localSearchCode, then mcp__serena__find_symbol. Never skip to Bash search.
  </semble_first>

  <unverified_protocol priority="critical">
    No file path, symbol name, API route, schema field, prop, or library API may appear in any output
    unless that exact string was returned by a tool call in this session. Unconfirmed names →
    label [UNVERIFIED: name]. Never guess, correct spelling, or substitute a similar-looking name.
    A made-up file:line citation is worse than none.
  </unverified_protocol>

  <mcp_fallback>
    If MCP tools are unavailable or return errors: state the gap explicitly, use the narrowest Bash
    fallback available, and record the limitation in the output. Do not silently proceed with
    unverified assumptions when MCP evidence is missing.
  </mcp_fallback>

</shared_mcp_rules>
