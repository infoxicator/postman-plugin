---
name: mcp-testing
description: Use when exploring, debugging, or testing an MCP (Model Context Protocol) server from the terminal, whether the user built it or not. Inspects capabilities, calls tools, reads resources, gets prompts, signs in to OAuth servers, and saves MCP calls with pm.test assertions that run in collection runs and CI. Not for working on Postman workspaces through Postman's own MCP tools; that is postman-mcp-server. Use for requests such as "check my MCP server's tools work".
---

# MCP Testing

`postman mcp` is an MCP client. Each command connects to one server, does one
operation, prints the result and disconnects. It does not start, install or
configure servers, and it never edits project files. Needs Postman CLI 1.88.0
or later; `postman mcp login` and MCP requests in `collection run` need 1.89.0.

## Pick a target

Every command takes exactly one target:

| Server | Target |
| --- | --- |
| Remote, over HTTP | `--url <url>` (streamable HTTP, falling back to legacy SSE) |
| Local, over stdio | `--allow-stdio -- <command> <args...>` |
| A saved MCP request | `--request <path-or-id>` |

Starting a local process always needs `--allow-stdio`; without it the command
exits 8. Give the server command after `--`, or as one `--command` string,
which is split at every space. No argument can contain a space, and a script
given by absolute path cannot take arguments; use a path relative to the
current directory. When a stdio server exits before the handshake, run its
command directly to see why it does not start.

## Explore and call

| Question | Command |
| --- | --- |
| What does the server offer? | `postman mcp inspect` |
| Which tools, and their schemas? | `postman mcp tools list` |
| Does a tool work? | `postman mcp tools call <tool> --arg key=value` |
| Resources | `postman mcp resources list`, `resources templates list`, `resources read <uri>` |
| Prompts | `postman mcp prompts list`, `prompts get <name> --arg key=value` |

`--arg` values are typed by the tool's `inputSchema`; use `--args` for a JSON
object, `@file` or `-` for stdin. Arguments are checked against the schema
before sending, and a failed check exits 2 without contacting the tool. To
send a deliberately invalid call, save it as a request message (below).

Add `--json` to parse a result, and select fields with `jq`, for example
`.structuredContent` or `.content`; `_meta` can carry large server icons. A
server's sampling and elicitation requests are declined, because a one-shot
client cannot answer them.

## Credentials

The first match wins: `-H "Authorization: ..."`, the saved request's auth, auth
it inherits from its folder or collection, then the sign-in stored by
`postman mcp login --url <url>`. Pass tokens from an environment variable
(`-H "Authorization: Bearer $TOKEN"`) or a secret variable (see **variables**),
never as a literal.

Without a terminal, a server that needs an OAuth sign-in makes the command exit
3 with the commands that fix it. `postman mcp login --url <url>` opens a browser
for the user to sign in, and every later command reuses that sign-in.
`postman mcp credentials` lists stored sign-ins without showing tokens.

## Save calls as tests

Save an MCP call as a request when it should be rerun, asserted on or shared.
See **collection-schema-v3** for the `mcp-request` file format.

```bash
postman collection new "My Server"    # when the collection does not exist yet
postman collection request add "Sum" --collection "My Server" --type mcp
postman collection request update "Sum" --collection "My Server" \
  --command "node server.js --stdio" \
  --body '{"method":"tools/call","params":{"name":"add","arguments":{"a":2,"b":3}}}' \
  --scripts 'afterResponse:pm.test("sum is 5", () => pm.expect(pm.response.result.structuredContent.sum).to.eql(5));'
```

Use `--url` instead of `--command` for an HTTP server; the file records it as
`transport: sse`. Script events are `beforeConnect`, `afterConnect`,
`afterResponse` and `afterDisconnect`. A collection or folder definition
holds the same hooks as `mcp:<event>`, and they run for every MCP request
under it.

- `afterConnect` sees what the server lists: `pm.response.tools`,
  `resources`, `resourceTemplates`, `prompts`, `server` and `capabilities`.
  Put assertions on those lists here.
- `afterResponse` sees `pm.response.result`, and runs only for a message that
  calls `tools/call`, `resources/read` or `prompts/get`. On a list message
  such as `tools/list` it never runs, and its tests are silently missing from
  the assertion count.

Assert on `structuredContent` only when the tool declares an `outputSchema`
(see `tools list --json`); otherwise assert on `content`, for example
`pm.response.result.content[0].text`. Run `postman collection lint` on the
collection after editing it, and check the run's assertion count matches the
tests you wrote.

Each MCP request opens its own session, and a stdio request starts its own
server process, so no state carries from one request to the next. Write each
request against the server's starting state. A flow that needs state from an
earlier call, such as create then complete, needs a server that keeps state
across sessions, or a test outside the collection.

- `postman mcp tools call <tool> --request <path>` sends the saved arguments
  and runs the scripts; `--arg` or `--args` override the arguments.
- `postman collection run <collection-path>` runs every MCP request with the
  rest of the collection. Add `--allow-stdio` when any of them starts a local
  server. See **api-testing** for run options and **ci-integration** for CI,
  including which reports a run can write.

Use command-specific `-h` for options.

## Verification

| Exit | Meaning |
| --- | --- |
| 0 | Success |
| 1 | Failure, including a tool result with `isError` or a failed test |
| 2 | Usage error, including arguments that fail the `inputSchema` check |
| 3 | The server rejected or requires credentials (HTTP 401/403) |
| 4 | `inspect` could not list everything |
| 5 | Cancelled |
| 8 | Refused: stdio without `--allow-stdio`, or no browser could open to sign in |

Report the target, transport and negotiated protocol version, the operations
run, test counts and the exit code. Distinguish a server that answered from a
tool that succeeded and from assertions that passed: a tool can return a
result with `isError`, and a returned result can still fail a test.
