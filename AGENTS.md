# AGENTS.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Project Overview

IBM Data Shift (also known as "Boson Bridge" when rebranded) is a native macOS SwiftUI application for peer-to-peer migration of user data between Macs. It uses Apple's Network.framework with a custom protocol over Bonjour (DNS-SD) for device discovery and direct file transfer.

## Build Commands

```bash
# Build the project
xcodebuild -project migrator.xcodeproj -scheme "Data Shift" -configuration Debug build

# Build for release
xcodebuild -project migrator.xcodeproj -scheme "Data Shift" -configuration Release build

# Run unit tests
xcodebuild -project migrator.xcodeproj -scheme "Data Shift" test

# Run UI tests
xcodebuild -project migrator.xcodeproj -scheme "Data Shift" -destination 'platform=macOS' test

# Open in Xcode
open migrator.xcodeproj
```

## Code Style

This project uses SwiftLint. If not installed, the build will warn but continue. Install via `brew install swiftlint`.

## Architecture

### Core Data Flow

```
Source Mac                                    Destination Mac
───────────────────────────────────────────────────────────────
NetworkBrowser ─────discovers via Bonjour────► NetworkServer
       │                                              │
       ▼                                              ▼
NetworkConnection ◄────TLS encrypted────► NetworkConnection
       │              custom protocol              │
       ▼                                              ▼
MigrationController                      MigrationController
       │                                              │
       ▼                                              ▼
MigrationViewModel ◄──progress updates──► MigrationViewModel
```

### Key Controllers (migrator/Controllers/)

- **MigrationController.swift**: Singleton orchestrator. Manages connection lifecycle, state machine (`MigrationState` enum), and coordinates between network layer and UI. Uses Combine publishers for reactive state updates.

- **NetworkConnection.swift**: Handles bidirectional file transfer. Contains `sendFile()` (recursive for directories), `sendSymlinks()`, and `receiveNextMessage()`. Uses async/await with `sendAsyncWrapper()` for retry logic.

- **NetworkBrowser.swift** / **NetworkServer.swift**: Bonjour discovery and connection acceptance. Browser runs on source Mac, Server runs on destination Mac.

- **MigrationReportController.swift**: Collects migration statistics and errors for optional report generation.

### Custom Network Protocol (migrator/Model/Network/)

- **MigratorNetworkProtocol.swift**: NWProtocolFramer implementation defining message framing over TLS
- **MigratorMessageType.swift**: Message types: `.file`, `.directory`, `.symlink`, `.multipartFile`, `.hostname`, `.metadata`, `.result`, etc.
- **FileMessage.swift** / **SymbolicLinkMessage.swift**: Codable payloads for file metadata and symlink info

### Configuration (migrator/AppContext.swift)

All app settings are centralized in `AppContext`. Settings can be managed via MDM profiles (UserDefaults). Key patterns:
- Private `fallback*` constants define defaults
- Public computed properties read from UserDefaults, falling back to defaults
- File exclusion/inclusion lists control what gets migrated

### View Layer (migrator/Views/)

SwiftUI views organized by migration phase. Each major view has a corresponding ViewModel in the same directory. Views use `@Published` properties from `MigrationController.shared` for state.

## Known Concurrency Considerations

The `symlinks` array in `NetworkConnection.swift` is managed by a `SymlinksManager` actor for thread safety. When processing directories recursively, multiple async tasks may process symlinks concurrently. Always use `await symlinksManager.append()` and `await symlinksManager.takeAll()`.

## File Transfer Chunking

Large files are split into 32MB chunks (`chunkSize = 33_554_432`). The receiver reassembles using `partNumber` in `FileMessage`. Empty files and small files are sent as single `.file` messages; large files use `.multipartFile`.

## License Header Requirement

All source files must include:
```swift
//
// © Copyright IBM Corp. 2023, 2025
// SPDX-License-Identifier: Apache2.0
//
```

## Commits

Include `Signed-off-by: Name <email>` in commit messages per DCO 1.1.
