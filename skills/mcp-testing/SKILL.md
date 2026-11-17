---
name: mcp-testing
description: Use when checking, debugging or writing tests for an MCP server, local or remote, whoever built it. Use for requests such as "does my MCP server work", "why does this tool call fail", "connect to this MCP URL" or "make sure these tools keep working". Not for using Postman's own MCP tools on a workspace; that is postman-mcp-server.
---

# MCP Testing

`postman mcp` is a scriptable MCP client in the CLI, not only in the app. Each
command connects to one server, does one operation and disconnects;
`postman mcp -h` has the targets, credential order and exit codes. If the
command is missing, update the CLI rather than conclude Postman cannot do this.

One `postman mcp tools call` answers whether a tool works. Save the call as an
MCP request (`postman collection request add --type mcp`) when the check should
outlive the session: `collection run` repeats it locally and in CI with the same
transport, auth and scripts, so what you proved by hand is what CI checks.

Take expectations from the tool's contract: its `inputSchema`, `outputSchema`
and description, which are what a calling agent relies on. A test copied from
the output the tool just returned only proves the server agrees with itself.

## Results that mislead

- **Exit 2 from `tools call`**: the CLI rejected the arguments against
  `inputSchema` and sent nothing. To test how the server handles bad input,
  save the call as a request message, which is sent as written.
- **Exit 4 from `inspect`**: one list method failed, often an optional one such
  as `resources/templates/list`. The rest of the server may be fine.
- **Sampling and elicitation** are declined, because a one-shot client cannot
  answer them. A tool that needs them fails here, not in an agent's client.
- **The stdio command is split at every space**, so no argument can contain
  one, and a script given by absolute path cannot take arguments; use a
  relative path.
- **Each MCP request is a new session**, and a stdio request a new server
  process, so server state does not carry from one request to the next. Values
  do: a script stores what one call returned with
  `pm.collectionVariables.set()`, and a later request uses it as `{{name}}` in
  its arguments. That is how a run chains steps, such as create then complete,
  into one flow that CI repeats unattended. The chain also needs a server that
  keeps its state across sessions; a stdio server restarted for every request
  loses it, so test that flow in the server's own tests instead.

## Credentials

`postman mcp login` opens a browser for the user and keeps the sign-in on this
machine. CI has neither, so CI for a server that needs auth takes a token from
the CI secret store, set as the request's auth through a variable
(see **variables**).

## Verification

Keep three results apart: the server answered, the tool succeeded (no
`isError`), and the assertions passed. Exit 1 covers both an `isError` result
and a failed test.
