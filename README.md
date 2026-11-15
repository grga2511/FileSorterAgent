# FileSorterAgent

An interactive Kotlin application that connects to a separate MCP
server using the Koog framework.

It supports two modes:

- Direct execution of an MCP tool selected from a console menu.
- Natural-language tasks where a Gemini-powered agent selects
  and combines available MCP tools.

Despite the project name, the current application inspects files.
The connected file MCP server provides listing and reading tools,
not file-moving tools.

## Related project

This application requires a separately running MCP server:

https://github.com/grga2511/file-mcp-server

Start the MCP server before starting this application.
The agent connects to it over Streamable HTTP; it does not start
the server automatically.

## Requirements

- JDK 21
- IntelliJ IDEA, recommended
- A running, reachable file-mcp-server instance
- Internet access for dependency downloads and Gemini requests
- A valid Google Gemini API key for the configured model

The Gradle Wrapper is included.

The current startup code requires the Google API key and model
configuration even when only direct tool execution is used.
Direct tool execution itself does not call Gemini.

## Configuration

In IntelliJ IDEA, open:

**Run → Edit Configurations → the configuration used to launch
FileSorterAgent → Environment variables**

Configure:

| Variable | Required | Purpose |
|---|---|---|
| MCP_URL | Yes | Streamable HTTP endpoint of the MCP server |
| GOOGLE_API_KEY | Yes | Your Google Gemini API key |
| GEMINI_MODEL | Yes | Supported Gemini model |
| GEMINI_PROXY_URL | No | HTTP proxy for Gemini requests |
| AGENT_SYSTEM_PROMPT | No | Additional system instructions for the agent |

Example:

```text
MCP_URL=http://127.0.0.1:8080/mcp;GOOGLE_API_KEY=your-api-key;GEMINI_MODEL=gemini-2.5-flash
```

If your network requires a proxy, also configure:

```text
GEMINI_PROXY_URL=http://your-proxy-host:8080
```

The Gemini proxy applies to Gemini requests. The MCP connection
uses a separate HTTP client.

Never commit real API keys or run configurations containing them.

### Supported models

The current model selection code supports:

- `gemini-2.5-flash`
- `gemini-2.5-pro`

Model availability, quotas and possible charges depend on your
Google account.

To use another model or provider, update the model selection,
LLM client and dependencies as needed.

## Running

1. Clone the repository:

   ```powershell
   git clone https://github.com/grga2511/FileSorterAgent.git
   cd FileSorterAgent
   ```

2. Open the project in IntelliJ IDEA.
3. Select JDK 21 and synchronize Gradle.
4. Start the separate file-mcp-server application.
5. Configure the agent's environment variables.
6. Run `FileSorterAgent.main()`.

Use the Run console to enter selections and parameters.

If running through the Gradle `run` task, console input must be
forwarded in `build.gradle.kts`:

```kotlin
tasks.named<JavaExec>("run") {
    standardInput = System.`in`
}
```

Then run:

```powershell
.\gradlew.bat run
```

Environment variables must be available to the configuration or
terminal used to launch the application.

## Project root and directory selection

The agent obtains its project root from the process working directory:

```kotlin
val projectRoot = Path.of("")
    .toAbsolutePath()
    .normalize()
    .toString()
```

Keep the working directory set to the intended project root.

For each file operation, the agent supplies:

- `projectRoot`: the absolute root path
- `directory`: a relative directory inside that root
- `fileName`: required when reading a particular file

Example:

```json
{
  "projectRoot": "C:\\Apps\\FileSorterAgent",
  "directory": "Testing",
  "fileName": "settings.conf"
}
```

Use `.` as the directory to target the root itself.

Paths refer to the MCP server's filesystem. They work directly
when both applications run on the same machine. If the server runs
elsewhere, the supplied paths must exist on that server.

The server validates that the requested directory remains inside
the root supplied with the request.

## Console menu

The tool names, descriptions and required parameters are discovered
from the connected MCP server.

With the related file server, the menu contains:

```text
Available tools:
1. list_files
2. read_text_file
A. Ask the agent
0. Exit
```

The menu repeats until you select `0`. Closing console input or
stopping the application also ends the program.

### Direct tool execution

Select a numbered tool.

The application displays its required parameters, supplies
`projectRoot` automatically and requests the remaining values.

For `list_files`, enter a relative directory such as:

```text
Testing
```

For `read_text_file`, enter the directory and an exact file name
returned by the listing tool.

Direct execution prints the tool's text result without asking
Gemini to interpret it.

The current parameter input handles required string parameters,
matching the related file server's tools. Other parameter types
or optional parameter input would require additional handling.

### Ask the agent

Select `A`, enter the relative directory, then describe your task.

Example:

```text
Find the files containing application configuration.
Read their contents and return their names and configuration parameters.
```

The agent can list files, select candidates, read their contents
and produce an answer.

Other examples:

```text
List the files in this directory. Do not read their contents.
```

```text
Read notes.txt and summarize its contents in one sentence.
```

File-reading capabilities depend on the MCP server and its installed
parsers. Not every file format provides extractable text.

## Errors and time limits

The menu handles tool errors and exceptions by displaying a warning
and returning to the selection prompt.

Direct tool calls have a 30-second limit. Natural-language tasks
have a separate limit configured around `agent.run()`.

After a timeout, the application displays a warning and returns
to the menu. Multi-step tasks can require several model requests
and may take longer than direct tool calls.

Stopping the application still cancels execution.

## Implementation notes

- MCP tools are registered through `McpToolRegistryProvider`.
- Numbered selections invoke the selected MCP tool directly.
- The Ask the agent option uses `AIAgent.run()` with the same tool registry.
- Required parameters come from tool descriptors.
- Direct MCP execution currently uses Koog's internal `McpTool` API
  with an explicit opt-in. Dependency upgrades may require changes.

## Troubleshooting

- **Missing environment variable:** configure it in the exact run
  configuration being used.
- **MCP connection failed:** confirm that the server is running and
  that `MCP_URL` matches its host, port and endpoint.
- **Directory could not be opened:** check the printed project root,
  relative directory and permissions on the server machine.
- **Gemini connection timeout:** check network access and
  `GEMINI_PROXY_URL`.
- **Ask the agent times out:** try a smaller task and inspect the
  MCP server logs to see which tools were called.
- **Console input is unavailable:** check the Gradle `standardInput`
  configuration or launch through an interactive terminal.
- **Gradle cannot find the project after moving it:** update the
  Gradle project path and working directory in IntelliJ IDEA.
