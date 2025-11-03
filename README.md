# Code Test: Movement Component System Prototype

Build a standalone prototype that demonstrates the core Movement component system architecture.

## What You're Building

A simple web app that:
1. Lets users write Vue components in a CodeMirror editor
2. Compiles them using WebContainers
3. Extracts props automatically
4. Renders the component with a props UI
5. Provides the $mvt global runtime

## Getting Started

### Prerequisites
- Node.js 18+ installed
- A modern browser (Chrome/Edge recommended for WebContainer support)

### Setup (2 minutes)

```bash
# Install dependencies
npm install

# Start dev server
npm run dev
```

The app will open at `http://localhost:3000`

**Important:** WebContainers require special CORS headers which are configured in `vite.config.js`. The dev server must be running for the app to work properly.

## Your Task

This is a Vue 3 application. Complete the TODOs in the following files:

### `src/App.vue`
1. **TODO 1**: Initialize WebContainer and compile Vue SFC when the user clicks "Compile Component"
2. **TODO 2**: Extract props from the Vue component and convert to JSON Schema format
   - Parse the Vue component's `props` definition
   - Convert Vue types (String, Number) to JSON Schema types (string, number)
   - Extract `default` values and `mvt.description` into the schema
   - Map `mvt.min`/`mvt.max` to `minimum`/`maximum` for number types
   - Return a proper JSON Schema object (see [JSON Schema spec](https://json-schema.org/))
   - The schema will automatically display in the right panel
3. **TODO 4**: Mount the compiled component in the preview panel

### `src/runtime.js`
4. **TODO 5**: Implement the `$mvt.store` methods (localStorage is recommended)

A sample Vue component is provided in the CodeMirror editor to help you get started. It includes props with custom `mvt` metadata that should be extracted. The app structure and reactive data bindings are already set up - you just need to implement the compilation, extraction, and mounting logic.

## Deliverables

1. GitHub repo with your completed solution
2. Update this README with your architectural decisions and any notes

## Time Limit: 24 hours

## Your Notes

### Implementation Summary

Successfully implemented all required functionality:

1. ✅ **WebContainer Integration** - Vue SFC compilation with @vue/compiler-sfc

2. ✅ **Props Extraction** - Automatic JSON Schema generation from Vue props

3. ✅ **Component Mounting** - Live preview with reactive prop updates  

4. ✅ **Runtime ($mvt)** - Global API with localStorage-based store

### Key Architectural Decisions

#### 1. WebContainer with Fallback Strategy

**Primary:** WebContainer + @vue/compiler-sfc for proper Vue compilation

- Boots on app mount to avoid 7-second first-compile delay

- npm install runs once during initialization

- Subsequent compiles: ~200-500ms

**Fallback:** Regex-based extraction when WebContainer unavailable

- Handles "already booted" errors and initialization failures

- Ensures app always works, even with CORS issues

#### 2. Props Extraction Algorithm

**Designed for Vue 3:** The extraction logic is specifically built to parse Vue 3 component syntax, including Options API with props definitions.

**Challenge:** Nested braces in props definitions (e.g., `mvt: { min: 0 }`)

**Solution:** Depth-aware parser that:

- Tracks brace depth to find correct prop boundaries

- Extracts only top-level props with `type:` field

- Maps Vue 3 types (String, Number, Boolean) → JSON Schema types (string, number, boolean)

- Extracts `mvt.description` → `description`

- Maps `mvt.min`/`max` → `minimum`/`maximum` for numbers

- Handles Vue 3 Options API `props` object syntax

#### 3. Component Mounting & Reactivity

- Wraps compiled component in reactive proxy

- Uses Vue `ref` + `watch` for live prop updates

- Injects `$mvt` via `globalProperties` for component access

- Proper cleanup prevents memory leaks

#### 4. $mvt Runtime

```javascript
window.$mvt = {
  currentUser: () => ({ id, name }),
  store: {
    getItem: async (key) => localStorage,
    setItem: async (key, value) => localStorage
  }
}
```

Used async/await for future extensibility (API calls, IndexedDB, etc.)

### Performance

- **First load:** 5-7s (WebContainer boot + npm install)

- **Subsequent compiles:** 200-500ms

- **Props extraction:** <50ms (pure regex, instant feedback)

### Known Limitations

1. Fallback has limited template parsing (no complex directives)

2. WebContainer requires Chrome/Edge with proper CORS headers

3. Limited to Options API (no full `<script setup>` in fallback)

### Testing

Verified with provided sample component:

- ✅ Compiles and displays in preview

- ✅ Props schema generated correctly

- ✅ Live prop updates trigger re-render

- ✅ $mvt.store persists data across refreshes

**Time invested:** ~14 hours
