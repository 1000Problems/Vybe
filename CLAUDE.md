# VybePM iOS — Voice Client for VybePM

You are rebuilding the Vybe iOS app as a voice-first client for the VybePM-v2 web API. The user speaks a task, the app parses it with Claude, and POSTs it to VybePM's existing API. That's the core loop.

**Do NOT change VybePM-v2 or its API. Only this iOS app adapts.**

## What to Delete

Delete everything in the `VybePM/` directory except `Assets.xcassets/`. Start fresh with the files listed below. Keep the `.xcodeproj` — update its file references to match the new source files.

## Architecture

```
User speaks → SpeechService transcribes → Claude parses into structured task →
App shows confirmation → User taps Send → POST to VybePM-v2 API → Done
```

Three tabs:
1. **Record** — voice input (default tab, auto-starts recording)
2. **Tasks** — read-only task list pulled from VybePM API
3. **Settings** — API key, base URL config

## VybePM-v2 API Reference

Base URL: `https://vybepm-v2.vercel.app`
Auth: `X-API-Key` header (stored in Keychain, entered in Settings)

### Endpoints Used

```
GET /api/projects
→ Returns: [{ id, name, display_name, color, pending_count, in_progress_count, total_count, ... }]

GET /api/projects/{slug}/tasks?status=pending
→ Returns: [{ id, title, description, task_type, priority, status, assignee, created_at, ... }]

POST /api/projects/{slug}/tasks
→ Body: { "title": string, "description": string?, "task_type": string, "priority": int, "assignee": string }
→ Returns: created task object
→ Valid task_type: "dev", "design", "animation", "content", "deploy", "report", "other"
→ Valid priority: 1 (critical), 2 (high), 3 (medium), 4 (low)
→ Valid assignee: "angel", "cowork", "claude-code"

PATCH /api/tasks/{id}
→ Body: { "status": string }
→ State machine: pending → in_progress → review → checked_in → deployed → done
```

## File Structure

```
VybePM/
  VybePMApp.swift              — App entry point
  ContentView.swift            — TabView with Record, Tasks, Settings

  Models/
    VybePMProject.swift        — Codable struct matching GET /api/projects response
    VybePMTask.swift           — Codable struct matching task objects
    ParsedTask.swift           — Intermediate struct from Claude parsing

  Services/
    SpeechService.swift        — Port from KitchenInventory (continuous listening + session chaining)
    VybePMAPIService.swift     — HTTP client for VybePM-v2 API
    TaskParserService.swift    — Claude API call that turns transcription into structured task
    KeychainHelper.swift       — Store API keys securely
    HapticsHelper.swift        — Haptic feedback

  Views/
    Record/
      RecordTabView.swift      — Main voice input screen (auto-start, transcript, confirm)
      TaskConfirmView.swift    — Shows parsed task for review before sending
    Tasks/
      TasksTabView.swift       — Project picker + task list from API
      TaskRowView.swift        — Single task row
    Settings/
      SettingsView.swift       — API key input, base URL, about
```

## Models

### VybePMProject.swift
```swift
struct VybePMProject: Codable, Identifiable {
    let id: Int
    let name: String            // slug used in API paths
    let display_name: String
    let color: String?
    let pending_count: Int
    let in_progress_count: Int
    let total_count: Int
}
```

### VybePMTask.swift
```swift
struct VybePMTask: Codable, Identifiable {
    let id: Int
    let title: String
    let description: String?
    let task_type: String
    let priority: Int
    let status: String
    let assignee: String
    let created_at: String
}
```

### ParsedTask.swift
```swift
struct ParsedTask: Identifiable {
    let id = UUID()
    var projectSlug: String     // which project this goes to
    var title: String           // the task title
    var description: String?    // optional longer description
    var taskType: String        // dev, design, animation, etc.
    var priority: Int           // 1-4
    var assignee: String        // angel, cowork, claude-code
}
```

## Services

### SpeechService.swift

Port directly from `/Users/angel/1000Problems/KitchenInventory/KitchenInventory LLM/KitchenInventory LLM/Services/SpeechService.swift`.

Copy it verbatim. The only change: remove the `KitchenError` references and replace with a local `VybePMError` enum:

```swift
enum VybePMError: LocalizedError {
    case speechUnavailable
    case microphoneDenied
    case networkError(String)
    case apiError(Int, String)
    case parseError

    var errorDescription: String? {
        switch self {
        case .speechUnavailable: return "Speech recognition is not available"
        case .microphoneDenied: return "Microphone access is required"
        case .networkError(let msg): return "Network error: \(msg)"
        case .apiError(let code, let msg): return "API error \(code): \(msg)"
        case .parseError: return "Failed to parse response"
        }
    }
}
```

### VybePMAPIService.swift

Simple HTTP client. All methods take the API key and base URL from Keychain/UserDefaults.

```swift
final class VybePMAPIService {
    private var baseURL: String { UserDefaults.standard.string(forKey: "vybepm_base_url") ?? "https://vybepm-v2.vercel.app" }
    private var apiKey: String? { KeychainHelper.retrieve(.vybepmAPIKey) }

    func fetchProjects() async throws -> [VybePMProject]
    func fetchTasks(projectSlug: String, status: String?) async throws -> [VybePMTask]
    func createTask(projectSlug: String, task: ParsedTask) async throws -> VybePMTask
    func updateTaskStatus(taskId: Int, status: String) async throws -> VybePMTask
}
```

Every request adds:
```
X-API-Key: {apiKey}
Content-Type: application/json
```

`createTask` maps ParsedTask to the POST body:
```json
{
  "title": "{task.title}",
  "description": "{task.description}",
  "task_type": "{task.taskType}",
  "priority": {task.priority},
  "assignee": "{task.assignee}"
}
```

### TaskParserService.swift

Uses the Claude API (same pattern as KitchenInventory's ClaudeAPIService, but simpler — no tool loop, just a single message→response).

Takes the voice transcript + list of project names, returns a `ParsedTask`.

System prompt:
```
You are a task parser for VybePM. The user will dictate a task via voice.
Extract: project (from the list provided), title, description, task_type, priority, assignee.

Available projects: {project_names_list}
Valid task_type values: dev, design, animation, content, deploy, report, other
Valid assignee values: angel, cowork, claude-code
Priority: 1=critical, 2=high (default), 3=medium, 4=low

If the user doesn't specify a project, use the most recently selected project or ask.
If the user doesn't specify priority, default to 2.
If the user doesn't specify assignee, default to claude-code.
If the user doesn't specify task_type, infer from context (bug fix → dev, make a video → animation, etc).

Respond with ONLY valid JSON, no markdown:
{"project": "slug", "title": "...", "description": "...", "task_type": "...", "priority": 2, "assignee": "claude-code"}
```

Claude model: `claude-haiku-4-5` (fast, cheap, parsing only).
API key: Same Anthropic key stored in Keychain.

### KeychainHelper.swift

Port from KitchenInventory. Two keys:
- `.claudeAPIKey` — Anthropic API key for task parsing
- `.vybepmAPIKey` — VybePM X-API-Key header value

### HapticsHelper.swift

Port from KitchenInventory. Light tap, success, warning, error.

## Views

### RecordTabView.swift

Port the recording UI from KitchenInventory's `MicTabView.swift`. Changes:

1. **Hints** — Replace kitchen examples with task examples:
   - "Fix the login bug in YTCombinator"
   - "Add dark mode to RubberJoints"
   - "Create an animation for the kids channel"

2. **Post-recording flow** — Instead of parsing into grocery items, parse into a `ParsedTask`:
   - User taps Done → state goes to `.processing`
   - Call `TaskParserService.parse(transcript, projects)` → get `ParsedTask`
   - State goes to `.confirming` → show `TaskConfirmView`

3. **No SwiftData dependency** — This app has no local database. Everything goes to the VybePM API.

### TaskConfirmView.swift

Shows the parsed task for review before sending:
- Project name (with color dot, tappable to change)
- Title (editable text field)
- Task type (picker: dev, design, animation, content, deploy, report, other)
- Priority (picker: 1-4)
- Assignee (picker: angel, cowork, claude-code)
- Description (optional, editable)

Two buttons:
- **Send** → calls `VybePMAPIService.createTask()`, shows success, returns to Record tab
- **Cancel** → discards, returns to Record tab

### TasksTabView.swift

Simple read-only view:
1. Horizontal scrollable project picker at top (colored pills with names, from GET /api/projects)
2. Task list below, filtered by selected project
3. Each row shows: priority dot, title, task type badge, status badge, assignee
4. Pull to refresh
5. Tapping a task could expand inline to show description — keep it simple

### SettingsView.swift

- Anthropic API Key field (saved to Keychain, masked)
- VybePM API Key field (saved to Keychain, masked)
- VybePM Base URL field (saved to UserDefaults, default: https://vybepm-v2.vercel.app)
- App version / about

## State Management

Use a single `@MainActor` ObservableObject called `AppViewModel`:

```swift
@MainActor
final class AppViewModel: ObservableObject {
    // Voice state (same pattern as KitchenInventory AIViewModel)
    @Published var voiceSessionState: VoiceSessionState = .idle
    @Published var voiceTranscript: String = ""
    @Published var parsedTask: ParsedTask?
    @Published var voiceError: String?

    // Data from API
    @Published var projects: [VybePMProject] = []
    @Published var tasks: [VybePMTask] = []
    @Published var selectedProjectSlug: String?
    @Published var lastUsedProjectSlug: String?  // persisted in UserDefaults

    // Services
    private let speechService = SpeechService()
    private let apiService = VybePMAPIService()
    private let taskParser = TaskParserService()

    // Voice mode methods (port from AIViewModel)
    func enterVoiceMode() { ... }
    func finishVoiceMode() { ... }  // calls taskParser
    func cancelVoiceMode() { ... }

    // API methods
    func loadProjects() async { ... }
    func loadTasks(for projectSlug: String) async { ... }
    func sendTask(_ task: ParsedTask) async throws -> VybePMTask { ... }
}
```

## Design System

- Light theme by default (match VybePM-v2 web)
- Accent color: system blue
- Project colors: use the hex `color` field from the API for project indicators
- Clean, minimal — productivity tool not a showcase
- Large tap targets for voice-first use
- System font stack, no custom fonts

## What NOT to Do

- Do NOT use SwiftData or any local database — all data lives in VybePM API
- Do NOT add a chat/AI conversation view — this is voice-to-task only
- Do NOT add file upload or attachments — that's a web feature
- Do NOT add task editing beyond status changes — edit in the web UI
- Do NOT store any secrets in code — all keys in Keychain
- Do NOT add push notifications
- Do NOT add onboarding flow — just the settings screen for API keys

## Build Configuration

- Deployment target: iOS 17.0
- Swift 5.9+
- No external dependencies (no SPM packages) — use Foundation URLSession for networking
- No CocoaPods, no Carthage
- Frameworks: AVFoundation, Speech, SwiftUI

## Commit Strategy

Single commit when everything works:
"VybePM iOS voice client — speak tasks directly into VybePM API"

Push to main.
