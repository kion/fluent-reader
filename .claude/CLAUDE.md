# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fluent Reader is a modern desktop RSS reader built with Electron, React, and Redux. It supports local reading and syncing with multiple RSS services (Fever, Feedbin, Google Reader API, Inoreader, Miniflux, Nextcloud News).

## Common Commands

### Development
```bash
npm install              # Install dependencies
npm run build            # Compile TypeScript and bundle with webpack
npm run electron         # Start the application (requires build first)
npm start                # Build and start in one command
npm run format           # Format code with Prettier
```

### Packaging
```bash
# Generate certificate for Windows signing (one-time)
electron-builder create-self-signed-cert

# Platform-specific builds
npm run package-win      # Windows AppX packages (x64, ia32, arm64)
npm run package-mac      # macOS DMG (x64)
npm run package-mas      # macOS App Store (uses build/resignAndPackage.sh)
npm run package-linux    # Linux AppImage (x64)
```

## Architecture

### Three-Process Electron Architecture

The app uses standard Electron multi-process architecture with three webpack bundles:

1. **Main Process** (`src/electron.ts` → `dist/electron.js`)
   - Application lifecycle, window management, native menus
   - Platform-specific behavior (macOS edit menu, single-instance lock)
   - Settings persistence via electron-store
   - Entry point: `electron.ts`, config: `main/` directory

2. **Preload Script** (`src/preload.ts` → `dist/preload.js`)
   - Context isolation bridge using `contextBridge`
   - Exposes two APIs: `window.settings` and `window.utils`
   - Bridges are in `src/bridges/` (settings.ts, utils.ts)

3. **Renderer Process** (`src/index.tsx` → `dist/index.js`)
   - React UI with Redux state management
   - All UI code lives here

### Redux State Structure

The app uses Redux with thunk middleware. State is divided into seven slices (see `src/scripts/reducer.ts`):

- `sources`: RSS feed sources
- `items`: Individual RSS items/articles
- `feeds`: Feed display state
- `groups`: Source organization/folders
- `page`: Current page/view state
- `service`: Sync service configuration and state
- `app`: Application-level state (theme, notifications, UI state)

Each slice has its own model file in `src/scripts/models/` with actions, reducers, and thunks.

### Data Persistence

- **Lovefield databases** (IndexedDB wrapper):
  - `sourcesDB`: RSS sources (schema version 3)
  - `itemsDB`: RSS articles (schema version 1)
  - Schema definitions in `src/scripts/db.ts`

- **electron-store**: Application settings and service configurations

### Service Architecture

RSS service syncing is abstracted through a hooks pattern (`src/scripts/models/service.ts`):

```typescript
interface ServiceHooks {
    authenticate?: (configs) => Promise<boolean>
    reauthenticate?: (configs) => Promise<ServiceConfigs>
    updateSources?: () => AppThunk<Promise<[RSSSource[], Map<string, string>]>>
    fetchItems?: () => AppThunk<Promise<[RSSItem[], ServiceConfigs]>>
    syncItems?: () => AppThunk<Promise<[Set<string>, Set<string>]>>
    markRead/markUnread?: (item) => AppThunk
    star/unstar?: (item) => AppThunk
}
```

Service implementations are in `src/scripts/models/services/`:
- `fever.ts` - Fever API
- `feedbin.ts` - Feedbin
- `greader.ts` - Google Reader API (also used by Inoreader)
- `miniflux.ts` - Miniflux
- `nextcloud.ts` - Nextcloud News

### Component Structure

- **Containers** (`src/containers/`): Redux-connected components
  - `page-container.tsx`: Main layout
  - `nav-container.tsx`: Navigation sidebar
  - `article-container.tsx`: Article list
  - `feed-container.tsx`: Feed content view
  - `settings-container.tsx`: Settings panel (with sub-containers in `settings/`)

- **Components** (`src/components/`): Presentational components
  - Uses Fluent UI (@fluentui/react) for UI primitives
  - `context-menu.tsx` and `log-menu.tsx` refactored as function components

### Internationalization

- Uses `react-intl-universal` for i18n
- Locale files in `src/scripts/i18n/` (JSON format)
- Supported languages: en-US, zh-CN, zh-TW, fr-FR, de, es, it, ja, ko, nl, pt-BR, pt-PT, ru, sv, tr, uk, cs, fi-FI

## TypeScript Configuration

- Target: ES2019
- Module: CommonJS (for Electron compatibility)
- JSX: React
- Minimal tsconfig - relies on webpack/ts-loader for bundling

## Key Implementation Details

- **Window state persistence**: Uses `electron-window-state` for remembering size/position
- **Dark mode**: Native dark mode support via `nativeTheme.themeSource`
- **Single instance**: Non-Mac App Store builds use `app.requestSingleInstanceLock()`
- **Context menus**: Custom implementation in `components/context-menu.tsx`
- **Article rendering**: Built-in article view with Mercury Parser for full-content extraction
- **Background sync**: Configurable fetch frequency per source with push notifications
- **Rules engine**: Automatic article actions (hide/read/star) based on regex patterns

## Important Notes

- When adding new service integrations, implement `ServiceHooks` interface in `src/scripts/models/services/`
- Database schema changes require version bumps in `src/scripts/db.ts`
- Platform-specific code should use `process.platform` checks (see `electron.ts` for examples)
- All electron main/renderer communication goes through the preload bridge - never use `nodeIntegration`
