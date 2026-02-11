# UserPreferences Plugin Architecture

## Overview

The UserPreferences plugin is a WPEFramework/Thunder-based service that provides backward-compatible UI language preference management. It acts as a compatibility layer between legacy systems and the modern UserSettings plugin, ensuring seamless migration and interoperability.

## System Architecture

### Component Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                    Client Applications                       │
│              (Legacy Systems, UI Components)                 │
└──────────────────────┬──────────────────────────────────────┘
                       │ JSON-RPC API
                       │ (getUILanguage, setUILanguage)
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              UserPreferences Plugin                          │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  JSON-RPC Interface Layer                           │   │
│  │  - Request validation & routing                     │   │
│  │  - Response formatting                              │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                        │
│  ┌──────────────────▼──────────────────────────────────┐   │
│  │  Format Conversion Engine                           │   │
│  │  - UI Format (US_en) ⟷ Presentation (en-US)       │   │
│  │  - Bidirectional transformation                     │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                        │
│  ┌──────────────────▼──────────────────────────────────┐   │
│  │  Migration Manager                                   │   │
│  │  - One-time data migration                          │   │
│  │  - File ⟷ UserSettings synchronization            │   │
│  └──────────────────┬──────────────────────────────────┘   │
│                     │                                        │
│  ┌──────────────────▼──────────────────────────────────┐   │
│  │  File Persistence Layer (GLib KeyFile)              │   │
│  │  - /opt/user_preferences.conf                       │   │
│  └─────────────────────────────────────────────────────┘   │
└──────────────────────┬──────────────────────────────────────┘
                       │ IUserSettings Interface
                       ▼
┌─────────────────────────────────────────────────────────────┐
│              UserSettings Plugin                             │
│  - Centralized settings management                          │
│  - Persistent storage backend                               │
│  - Notification infrastructure                              │
└─────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. JSON-RPC Interface
- **Purpose**: Exposes legacy-compatible API endpoints
- **Methods**:
  - `getUILanguage()`: Retrieves UI language in legacy format (e.g., "US_en")
  - `setUILanguage(language)`: Sets UI language using legacy format
- **Transport**: HTTP/WebSocket via Thunder framework
- **Version**: API v1.0.0

### 2. Format Conversion Engine
Bidirectional language code transformation:
- **UI Format**: `CC_ll` (Country_language) - e.g., "US_en"
- **Presentation Format**: `ll-CC` (language-Country) - e.g., "en-US"

**Conversion Rules**:
- Length validation: Exactly 5 characters
- Separator validation: '_' for UI, '-' for presentation
- Position validation: Separator at index 2
- Component swapping: First and last 2-character segments

### 3. Migration Manager
Ensures seamless transition from file-based to UserSettings-based storage.

**Migration States**:
1. **Migration Required + File Exists**:
   - Read UI language from file
   - Convert to presentation format
   - Set in UserSettings
   - Mark migration complete

2. **Migration Required + No File**:
   - Read presentation language from UserSettings
   - Convert to UI format
   - Create file with value
   - Mark migration complete

3. **Migration Complete**:
   - Synchronize UserSettings to file
   - Maintain consistency across both stores

### 4. File Persistence Layer
- **Implementation**: GLib KeyFile API
- **Location**: `/opt/user_preferences.conf`
- **Format**: INI-style configuration
  ```ini
  [General]
  ui_language=US_en
  ```
- **Thread Safety**: Protected by Critical Section lock

### 5. Notification System
Subscribes to UserSettings change events via `IUserSettings::INotification`:
- **OnPresentationLanguageChanged**: Synchronizes changes back to file
- **Optimization**: Updates file only when value changes (prevents redundant writes)
- **Execution Context**: Runs in UserSettings notification thread (non-blocking)

## Data Flow

### Get UI Language Flow
```
Client Request → JSON-RPC Handler → Check Migration Status
    ↓
Migration Required? → Perform Migration → Update File
    ↓
Query UserSettings → Get Presentation Language
    ↓
Convert to UI Format → Cache Last Value → Return to Client
```

### Set UI Language Flow
```
Client Request → JSON-RPC Handler → Validate Input Format
    ↓
Check Migration Status → Migration Required? → Perform Migration
    ↓
Convert to Presentation Format → Set in UserSettings
    ↓
UserSettings Notification → Convert Back to UI Format
    ↓
Update File (if changed) → Cache Last Value
```

## Dependencies

### Thunder Framework
- **PluginHost::IShell**: Plugin lifecycle management
- **PluginHost::JSONRPC**: RPC interface implementation
- **Core::CriticalSection**: Thread synchronization

### External Interfaces
- **Exchange::IUserSettings**: Settings storage and retrieval
- **Exchange::IUserSettingsInspector**: Migration state management

### System Libraries
- **GLib**: KeyFile API for INI file parsing/writing
- **IARMBus**: Inter-process communication (indirect dependency)
- **DeviceSettings (DS)**: Device-level settings integration

## Threading Model

### Synchronization Strategy
- **Critical Section Lock**: Protects `_service` pointer access
- **Lock Scope**: Minimal - only during interface queries
- **Notification Context**: Executes in UserSettings thread (safe for non-blocking operations)

### Concurrency Considerations
- File operations are atomic (GLib guarantees)
- Migration flag prevents race conditions
- Last value caching reduces file I/O

## Configuration

### Build-time Options
- `PLUGIN_USERPREFERENCES`: Enable/disable plugin compilation
- `RDK_SERVICE_L2_TEST`: Link test mock libraries
- `COMCAST_CONFIG`: Include platform-specific settings

### Runtime Configuration
- Plugin config file: `UserPreferences.conf.in`
- Startup order: Configurable via `PLUGIN_USERPREFERENCE_STARTUPORDER`
- UserSettings dependency: Required interface

## Error Handling

### Retry Mechanism
- UserSettings interface acquisition: 5 retries with 1-second intervals
- Handles race conditions during plugin initialization

### Failure Scenarios
1. **UserSettings unavailable**: Return error, log failure
2. **Invalid format**: Reject request, return false
3. **Migration failure**: Abort operation, maintain consistency
4. **File I/O error**: Log error, continue with UserSettings as source of truth

## Performance Characteristics

### Optimization Techniques
- **Lazy Migration**: Performed once on first access
- **Value Caching**: Prevents redundant file writes
- **Minimal Locking**: Critical sections only for pointer access
- **In-context Notifications**: Avoids thread pool overhead

### Resource Usage
- **Memory**: Minimal (~1KB for plugin state)
- **File I/O**: Only on language changes
- **Network**: None (local IPC only)
- **CPU**: Negligible (string conversions only)

## Security Considerations

- **Input Validation**: Format checks prevent injection attacks
- **File Permissions**: Relies on OS-level protection for `/opt/`
- **No Authentication**: Assumes trusted environment (Thunder provides authentication layer)

## Extension Points

The plugin is designed for potential future enhancements:
- Additional notification handlers (stubs present for audio, captions, accessibility)
- Extended format support (beyond 5-character codes)
- Multiple preference types (currently language-only)
