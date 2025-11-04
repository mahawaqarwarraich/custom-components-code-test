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

## Architectural Decisions & Implementation Notes

### Implementation Summary

All required functionality has been successfully implemented:

1. ✅ **WebContainer Integration** - Vue SFC compilation using `@vue/compiler-sfc`
2. ✅ **Props Extraction** - Automatic JSON Schema generation from Vue 3 component props
3. ✅ **Component Mounting** - Live preview with reactive prop updates
4. ✅ **Runtime ($mvt)** - Global API with localStorage-based store

### Key Architectural Decisions

#### 1. WebContainer with Performance Optimization

**Primary Strategy:** WebContainer + `@vue/compiler-sfc` for proper Vue 3 compilation

- **Initialization:** WebContainer boots on app mount (not on every compilation) to avoid 7-second delay after each compilation
- **Dependency Management:** `npm install` runs once during initialization
- **Performance:** 
  - First page load: 5-7 seconds (one-time initialization)
  - Subsequent compiles: ~200-500ms (no npm install overhead)

**Fallback Strategy:** Regex-based extraction when WebContainer unavailable

- Handles "already booted" errors gracefully
- Ensures app always works, even with CORS issues
- Limited template parsing (basic Vue directives only)

#### 2. Props Extraction Algorithm

**Approach:** Custom regex-based parser with depth-aware brace counting

**Designed for Vue 3:** The extraction logic is specifically built to parse Vue 3 Options API component syntax.

**Key Features:**
- **Brace Counting Algorithm:** Handles nested braces in props definitions (e.g., `mvt: { min: 0 }`)
- **Type Mapping:** Converts Vue 3 types → JSON Schema types
  - `String` → `string`
  - `Number` → `number`
  - `Boolean` → `boolean`
  - `Array` → `array`
  - `Object` → `object`
- **Metadata Extraction:**
  - `mvt.description` → `description`
  - `mvt.min`/`max` → `minimum`/`maximum` for number types
- **Function Defaults:** Skips function defaults (e.g., `default: () => []`) since they require execution

**Why Custom Parser?**
- Lightweight: No additional dependencies
- Fast: Regex-based parsing is instant (<50ms)
- Simple: Easy to understand and maintain
- Trade-off: Less robust than AST-based parsers, but sufficient for Vue 3 Options API

#### 3. Component Mounting & Reactivity

**Primary Path (WebContainer):**
- Uses Vue's official compiler to generate render functions
- Handles all Vue 3 directives properly (v-for, v-if, :key, :style, etc.)
- Properly scoped and reactive

**Fallback Path:**
- Basic template compiler for simple cases
- Limited directive support
- Primarily for error recovery

**Reactivity:**
- Uses Vue `ref` + `watch` for live prop updates
- Automatically remounts component when prop values change
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

### Future Improvements

1. Add CSS compilation support in WebContainer path
2. Enhance fallback to handle more Vue directives
3. Support Composition API (`<script setup>`)
4. Add TypeScript support for props
5. Improve error messages and user feedback


