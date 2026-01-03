# Tachi-back - Paperback Extension Development Guide

## Project Overview

Tachi-back is a Paperback extension that provides a client interface for Tachiyomi/Tachidesk servers. It allows users to browse, search, and read manga from their self-hosted Tachiyomi server through the Paperback iOS app.

**Core Architecture**: Single-file bundled source extension (`TachiBack/source.js`) that communicates with Tachidesk GraphQL API.

## Key Components

### 1. Main Extension Class (`TachiBack`)
Located at the end of [TachiBack/source.js](TachiBack/source.js#L1183-1754), this implements the Paperback `Source` interface with:
- **RequestManager**: Handles API requests with rate limiting (8 req/sec, 20s timeout)
- **StateManager**: Manages user settings (server URL, credentials, pagination)
- **CacheManager**: 180-second cache for search results and metadata
- **TachiBackRequestInterceptor**: Adds Basic Auth headers when configured

### 2. GraphQL Communication Pattern
All server interactions use GraphQL queries executed through `executeGraphQL()` helper ([source.js#L783-795](TachiBack/source.js#L783-795)):
```javascript
const QUERY = `query myQuery($var: Type!) { ... }`;
const data = await executeGraphQL(QUERY, variables, requestManager, stateManager);
```

**Critical**: Always wrap GraphQL calls in try-catch blocks. Server unavailability returns placeholder tiles via `getServerUnavailableMangaTiles()`.

### 3. State Management Architecture
Two-tier storage system:
- **Regular state** (`stateManager.store()`): Server URL, page size, UI toggles
- **Secure keychain** (`stateManager.keychain.store()`): Username/password credentials

Default values defined in `DEFAULT_VALUES` object ([source.js#L851-859](TachiBack/source.js#L851-859)). Access via `getTachiBackAPI()` and `getOptions()` helpers.

### 4. Dynamic UI (DUI) Forms
Settings use Paperback's DUI system with data binding pattern:
```javascript
App.createDUIBinding({
  async get() { return await stateManager.retrieve("key"); },
  async set(value) { await stateManager.store("key", value); }
})
```

See `serverSettingsMenu()` implementation ([source.js#L886-1024](TachiBack/source.js#L886-1024)) for complete example.

## Development Workflow

### Building & Testing
This is a **pre-compiled** extension repository:
1. [TachiBack/source.js](TachiBack/source.js) is the bundled output (DO NOT edit directly)
2. Source TypeScript files are not included in this repo (built elsewhere)
3. [versioning.json](versioning.json) tracks build metadata and extension info
4. [index.html](index.html) serves as the repository landing page

**To update the extension**: Modify source TypeScript files in the original development environment, rebuild, then replace [TachiBack/source.js](TachiBack/source.js).

### Version Management
When updating:
1. Increment version in `TachiBackInfo.version` ([source.js#L1167](TachiBack/source.js#L1167))
2. Update `buildTime` in [versioning.json](versioning.json)
3. Update version and name in [versioning.json sources array](versioning.json#L1)

### Testing Connection
Default test server: `http://192.168.29.109:4567` (local network)
- Test server availability via `interceptor.isServerAvailable()`
- Authentication uses Base64-encoded Basic Auth: `btoa(username:password)`

## Project-Specific Patterns

### Pagination
- Page size is user-configurable (default: 40 items)
- Metadata object carries `offset` between page loads: `{ offset: offset + pageSize }`
- Return `undefined` metadata to signal last page

### Image URL Construction
Images are relative paths from Tachidesk:
```javascript
const tachiBackAPI = await getTachiBackAPI(stateManager);
const fullImageURL = `${tachiBackAPI.url}${manga.thumbnailUrl}`;
```

### Chapter Progress Display
Special format in chapter group field ([source.js#L1238-1240](TachiBack/source.js#L1238-1240)):
```
{pageCount} pages · Reading (lastPage/totalPages) · scanlator
{pageCount} pages · Read · scanlator
```

### Homepage Sections
Dynamic sections based on user toggles (`showContinueReading`, `showRecentlyUpdated`, etc.). Each section uses GraphQL with ordering, filtering, and pagination.

### Error Handling Convention
Always check server availability before GraphQL operations:
```javascript
if (!await interceptor.isServerAvailable()) {
  return App.createPagedResults({ results: getServerUnavailableMangaTiles() });
}
```

## Repository Structure
```
.github/              # GitHub-specific files
Kavya/
  source.js           # Bundled extension (1724 lines)
  includes/
    icon.png          # Extension icon
index.html            # Repository landing page
versioning.json       # Build metadata
.gitignore            # Excludes node_modules, build artifacts
```

## Integration Points

**External Dependency**: Tachidesk/Tachiyomi server running Suwayomi GraphQL API
- GitHub: https://github.com/Suwayomi/Tachidesk-Server
- Expects GraphQL endpoint at `/api/graphql`
- All image paths are relative, server URL is base

**Paperback Platform**: Uses `@paperback/types` v0.8.7 toolchain
- Full DUI form system for settings
- Collection management intents for library sync
- Manga tracking support for read progress
