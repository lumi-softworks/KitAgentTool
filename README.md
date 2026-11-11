# KitAgentTool

📖 English | [中文版](README_zh.md)

Reusable Agent tool protocol and runtime models. Defines LLM tool calls, argument parsing, permission risk, cancellation context, and localized schemas, shared by host apps and plugins.

## Package

- Manifest package name: `KitAgentTool` (matches the repository name)
- Product / Target: `KitAgentTool`
- Platform: macOS 14+, iOS 17+
- Swift tools: 6.0

## Main Types

| Type | Description |
|------|-------------|
| `SuperAgentTool` | Agent tool protocol (name, description, schema, risk, execution) |
| `LocalizedAgentTool` | Wraps a tool's description and input schema per language |
| `ProjectConfigDirectory` | Project-level config conventions (writes `.lumi/`, reads `.agent/` for compatibility) |
| `ToolCall` | A tool call issued by the model (including local authorization state) |
| `ToolArgument` | Tool argument values (string, number, boolean, etc.) |
| `ToolExecutionContext` | Per-call cancellation context, forwardable to underlying resources |
| `CommandRiskLevel` | Command risk level (logic layer; UI extensions live in LumiUI) |
| `ToolCallAuthorizationState` | Local authorization state of a tool call |
| `ToolContextProviding` | Host-provided tool construction context (e.g. language preference) |
| `ToolError` / `ToolExecutionError` | Tool execution error types |
| `ImageAttachment` / `ToolImageResultCodec` | Image attachment encoding and decoding in tool results |

## Dependencies and Integration

```swift
dependencies: [
    .package(url: "https://github.com/lumi-softworks/KitAgentTool.git", from: "1.0.0"),
],
targets: [
    .target(
        name: "YourTarget",
        dependencies: [
            .product(name: "KitAgentTool", package: "KitAgentTool"),
        ],
    ),
]
```

## Usage

### Implementing a tool

```swift
import KitAgentTool

struct ReadFileTool: SuperAgentTool {
    let name = "read_file"

    func description(for language: LanguagePreference) -> String {
        "Read the contents of a file"
    }

    func inputSchema(for language: LanguagePreference) -> [String: Any] {
        [
            "type": "object",
            "properties": ["path": ["type": "string"]],
            "required": ["path"],
        ]
    }

    func permissionRiskLevel(arguments: [String: ToolArgument]) -> CommandRiskLevel {
        .low
    }

    func execute(arguments: [String: ToolArgument]) async throws -> String {
        guard case let .string(path)? = arguments["path"] else {
            throw ToolError.invalidArguments("path is required")
        }
        return try String(contentsOfFile: path, encoding: .utf8)
    }
}
```

### Project-level config directory

Project-level configuration (rules / skills / commands) is written under the project's `.lumi/` directory; reads stay compatible with the historical `.agent/` directory. Plugins should reuse `ProjectConfigDirectory` instead of hardcoding directory names.

```swift
// Write target: <project>/.lumi/rules
let target = ProjectConfigDirectory.writeURL(projectPath: path, component: "rules")

// Read candidates: <project>/.lumi/rules (preferred), <project>/.agent/rules (fallback)
let candidates = ProjectConfigDirectory.readURLs(projectPath: path, component: "rules")

// Locate a single entry's actual location
if let url = ProjectConfigDirectory.locate(
    projectPath: path,
    component: "skills",
    name: "my-skill"
) {
    // url may be .lumi/skills/my-skill, or .agent/skills/my-skill
}
```

Entries with the same name resolve to `.lumi`; updating or deleting entries in the historical directory happens **in place**, with no implicit migration.

### Execution with cancellation

```swift
func execute(arguments: [String: ToolArgument], context: ToolExecutionContext) async throws -> String {
    let processId = context.onCancel { terminateProcess() }
    defer { context.removeCancellationHandler(processId) }

    try context.checkCancellation()
    // ... run work ...
    return result
}
```

## Host integration

- Tool registration, permission prompts, and conversation persistence stay in the host app (e.g. Lumi Core / plugins).
- Reusable protocols, models, risk, and cancellation semantics live in this package.
- Use `KitAgentTool.ToolCall` to disambiguate from the LLM API's `ToolCall` type.

## Testing

```sh
swift test
```
