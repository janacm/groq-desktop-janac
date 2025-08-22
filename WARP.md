# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Commands for Development

### General Development

- **Start development server**: `pnpm dev` - Runs both Vite dev server and Electron app concurrently
- **Build frontend**: `pnpm build` - Builds React app with Vite to `dist/`
- **Build production app**: `pnpm dist` - Builds installable packages for current platform

### Platform-Specific Operations

#### macOS
- **Build for macOS**: `pnpm dist:mac` - Creates .dmg and .zip in `release/`
- **Fix app permissions**: `xattr -c /Applications/Groq\ Desktop.app` - Required after installation
- **Homebrew install**: Use unofficial tap:
  ```bash
  brew tap ricklamers/groq-desktop-unofficial
  brew install --cask groq-desktop
  xattr -c /Applications/Groq\ Desktop.app
  ```

#### Windows
- **Build for Windows**: `pnpm dist:win` - Creates NSIS installer and portable exe
- **Test Windows scripts**: `.\test-windows.ps1` - PowerShell test script

#### Linux
- **Build for Linux**: `pnpm dist:linux` - Creates AppImage, .deb, and .rpm packages

### Cross-Platform Testing

- **Run all platform tests**: `pnpm test:platforms` - Executes `./test-cross-platform.sh`
- **Test path handling only**: `pnpm test:paths` - Runs `test-paths.js`
- **Manual Docker test**: `docker build -f test-linux.Dockerfile -t groq-desktop-linux-test . && docker run --rm groq-desktop-linux-test`

## High-Level Architecture Overview

Groq Desktop is built on **Electron's dual-process architecture** with a React frontend and sophisticated MCP (Model Context Protocol) server integration:

**Core Architecture Pattern:**
- **Main Process** (`electron/main.js`) → Manages native system integration, MCP connections, and IPC coordination
- **Renderer Process** (`src/renderer/`) → React SPA with routing, chat UI, and settings management
- **Preload Script** (`electron/preload.js`) → Secure bridge exposing main process functions to renderer via `window.electron`

**Data Flow Overview:**
1. **User Input** → React Components → IPC calls via preload bridge → Main process handlers
2. **Chat Streaming**: Renderer → `chat-stream` IPC → Groq SDK → Streamed responses back to UI
3. **Tool Execution**: Model tool calls → MCP client → External tools → Results back through streaming
4. **Settings**: React settings page → JSON persistence in userData directory → Live reload across processes
5. **MCP Integration**: Settings-driven server connections → Tool discovery → Runtime execution with approval flow

**Cross-Platform Command Resolution:**
- Platform-specific script wrappers in `electron/scripts/` (`.sh`, `.cmd`, `.ps1`)
- Dynamic path resolution based on OS and environment (`electron/commandResolver.js`)
- Health checking and reconnection logic for MCP servers

## Key Architectural Components

### Core Electron Modules

#### MCP Manager (`electron/mcpManager.js`)
- **Responsibility**: Manages connections to MCP (Model Context Protocol) servers, tool discovery, and execution
- **Key Functions**: `connectMcpServerProcess()`, `getMcpState()`, `setupServerHealthCheck()`
- **Transport Support**: STDIO, SSE, StreamableHTTP with authentication via `authManager`
- **IPC Channels**: `get-mcp-tools`, `connect-mcp-server`, `disconnect-mcp-server`, `mcp-server-status-changed`
- **Health Monitoring**: Periodic health checks with automatic reconnection on failure

#### Settings Manager (`electron/settingsManager.js`)
- **Responsibility**: Persistent settings storage in userData directory as JSON
- **Key Functions**: `loadSettings()`, settings validation and defaults
- **IPC Channels**: `get-settings`, `save-settings`, `reload-settings`, `get-settings-path`
- **Settings Schema**: API key, model selection, temperature/top_p, MCP server configs, custom system prompt

#### Chat Handler (`electron/chatHandler.js`) 
- **Responsibility**: Streams chat completions from Groq SDK with tool calling support
- **Key Functions**: `handleChatStream()` with message pruning and retry logic
- **Model Support**: Vision detection, context window management from `shared/models.js`
- **IPC Channels**: `chat-stream` (send), streaming response events (receive)

#### Tool Handler (`electron/toolHandler.js`)
- **Responsibility**: Executes MCP tool calls with validation and error handling
- **Key Functions**: `handleExecuteToolCall()` with argument parsing and result limiting
- **IPC Channels**: `execute-tool-call` (handle)
- **Tool Approval**: Integrates with frontend approval modal flow

#### Command Resolver (`electron/commandResolver.js`)
- **Responsibility**: Cross-platform executable path resolution
- **Key Functions**: `resolveCommandPath()`, platform-specific script selection
- **Platform Logic**: Uses `.sh` for Unix, `.cmd/.ps1` for Windows, falls back to PATH lookup
- **Script Location**: `electron/scripts/run-{command}{-platform}.{ext}`

#### Auth Manager (`electron/authManager.js`) 
- **Responsibility**: OAuth2 flows for MCP servers requiring authentication
- **Key Functions**: `initiateAuthFlow()`, token management
- **IPC Channels**: `start-mcp-auth-flow`, `mcp-auth-reconnect-complete`

#### Window Manager (`electron/windowManager.js`)
- **Responsibility**: Main window lifecycle, development/production modes
- **Key Functions**: `initializeWindowManager()`, preload script setup

### React Frontend Components

#### App Component (`src/renderer/App.jsx`)
- **Core chat interface** with message history, model selection, and tool approval modal
- **Tool Approval Flow**: LocalStorage preferences (always/once/yolo/deny)
- **State Management**: Uses `ChatContext` for message persistence across routes

#### Settings Page (`src/renderer/pages/Settings.jsx`)
- **Settings management** with live saving, MCP server configuration
- **MCP Server Config**: Supports STDIO and SSE transports, environment variables
- **Validation**: JSON parsing for complex server configurations

#### Chat Context (`src/renderer/context/ChatContext.jsx`)
- **Global state** for chat messages shared between App and Settings
- **Provider Pattern**: Wraps entire app in `main.jsx`

### IPC Communication Layer

#### Preload Bridge (`electron/preload.js`)
- **Security boundary** between main and renderer processes
- **Exposed API**: `window.electron` with streaming chat, settings, MCP, and auth methods
- **Event Handling**: Cleanup functions for streaming listeners, status change notifications

## Important Development Context

### Prerequisites

- **Node.js**: v18+ (specified in `package.json` engines)
- **pnpm**: v10.9.0+ (specified as packageManager)
- **Platform Support**: Windows, macOS, Linux
- **Development Tools**: Docker (for Linux testing), PowerShell (Windows testing)

### Cross-Platform Considerations

- **Build Targets**: Configured in `electron-builder.yml` and `package.json`
  - macOS: .dmg, .zip (requires code signing bypass: `xattr -c`)
  - Windows: NSIS installer, portable exe
  - Linux: AppImage, .deb, .rpm
- **Script Resolution**: Platform-specific wrappers in `electron/scripts/`
  - Unix: `run-{tool}.sh`, `run-{tool}-linux.sh`
  - Windows: `run-{tool}.cmd`, `run-{tool}.ps1`
- **Path Handling**: Dynamic resolution in `commandResolver.js` based on `process.platform`
- **Testing Strategy**: Docker for Linux simulation, cross-platform path validation

### Common Development Pitfalls

- **macOS App Signing**: Unsigned apps require `xattr -c /Applications/Groq\ Desktop.app` after installation
- **Windows Script Execution**: `.sh` scripts fail on Windows - `commandResolver.js` returns direct commands instead
- **MCP Server Health Checks**: 60s intervals with automatic cleanup on failure
- **Tool Execution Timeouts**: 15s default, configurable per transport type
- **Settings Persistence**: Auto-saves with 800ms debounce, stored in `userData/settings.json`
- **Vision Model Detection**: Check `shared/models.js` for `vision_supported` before sending images
- **IPC Event Cleanup**: Use returned cleanup functions from preload bridge to avoid memory leaks

### Configuration and Settings

- **Settings File**: `{userData}/settings.json` - get path via `window.electron.getSettingsPath()`
- **API Key Storage**: Store Groq API key in settings as `GROQ_API_KEY`
- **MCP Server Config**: Define in `settings.mcpServers` object with:
  ```json
  {
    "server-id": {
      "transport": "stdio|sse|streamableHttp",
      "command": "path/to/executable", // stdio only
      "args": ["arg1", "arg2"], // stdio only  
      "url": "https://server.com/mcp", // sse/http only
      "env": { "VAR": "value" }
    }
  }
  ```
- **Model Definitions**: Centralized in `shared/models.js` with context windows and vision support
- **Build Artifacts**: Output to `release/` directory
