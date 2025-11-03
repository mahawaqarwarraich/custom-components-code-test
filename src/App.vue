<template>
  <div class="container">
    <div class="editor-panel">
      <h2>Component Editor</h2>
      <textarea ref="editorElement"></textarea>
      <button @click="compileComponent" class="compile-btn">
        Compile Component
      </button>
    </div>

    <div class="right-panel">
      <div class="props-panel">
        <h2>Props Schema</h2>
        <div class="props-schema">
          <!-- Displays extracted props as JSON schema -->
          <pre v-if="!propsSchema" class="empty-state">
Compile a component to see props schema</pre
          >
          <pre v-else>{{ JSON.stringify(propsSchema, null, 2) }}</pre>
        </div>

        <!-- Displays extracted props as input fields -->
        <div v-if="propsSchema?.properties" class="props-inputs">
          <h3>Props Values</h3>
          <div
            v-for="(propDef, propName) in propsSchema.properties"
            :key="propName"
            class="prop-input"
          >
            <label>{{ propName }}</label>
            <input
              v-if="propDef.type === 'string'"
              v-model="propValues[propName]"
              type="text"
              :placeholder="propDef.default"
            />
            <input
              v-else-if="propDef.type === 'number'"
              v-model.number="propValues[propName]"
              type="number"
              :placeholder="propDef.default"
              :min="propDef.minimum"
              :max="propDef.maximum"
            />
          </div>
        </div>
      </div>

      <div class="preview-panel">
        <h2>Component Preview</h2>
        <div ref="componentMount" class="component-mount">
          <!-- Compiled component mounts here -->
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch, h } from "vue";
import { createApp } from "vue";
import CodeMirror from "codemirror";
import "codemirror/lib/codemirror.css";
import "codemirror/mode/vue/vue.js";
import { WebContainer } from "@webcontainer/api";

export default {
  name: "App",
  setup() {
    const editorElement = ref(null);
    const componentMount = ref(null);
    const propsSchema = ref(null);
    const propValues = ref({});

    let editor = null;
    let webcontainer = null;
    let mountedApp = null;
    let lastComponentSource = null;

    // Sample component to start with
    const sampleComponent = `<template>
  <div class="metric-card">
    <h3>{{ title }}</h3>
    <p>Count: {{ count }}</p>
    <button @click="increment">+1</button>
  </div>
</template>

<script>
export default {
  props: {
    title: {
      type: String,
      default: 'My Counter',
      mvt: {
        type: 'text',
        description: 'Counter title'
      }
    },
    startValue: {
      type: Number,
      default: 0,
      mvt: {
        type: 'number',
        description: 'Starting value',
        min: 0,
        max: 100
      }
    }
  },
  data() {
    return {
      count: this.startValue
    }
  },
  methods: {
    async increment() {
      this.count++
      await $mvt.store.setItem('count', this.count)
    }
  }
}
<\/script>`;

    onMounted(async () => {
      // Initialize CodeMirror
      editor = CodeMirror.fromTextArea(editorElement.value, {
        mode: "vue",
        theme: "default",
        lineNumbers: true,
      });

      editor.setValue(sampleComponent);
      
      // Initialize WebContainer in background immediately
      console.log('Initializing WebContainer...');
      try {
        webcontainer = await WebContainer.boot();
        console.log('WebContainer booted');
        
        // Set up package.json
        await webcontainer.fs.writeFile('/package.json', JSON.stringify({
          name: 'vue-compiler',
          type: 'module',
          dependencies: {
            '@vue/compiler-sfc': '^3.4.0'
          }
        }, null, 2));
        
        // Install dependencies once
        console.log('Installing dependencies...');
        const installProcess = await webcontainer.spawn('npm', ['install']);
        await installProcess.exit;
        console.log('WebContainer ready!');
      } catch (error) {
        console.error('WebContainer initialization failed:', error);
      }
    });

    // Watch for prop value changes and remount component
    watch(propValues, async (newValues) => {
      if (lastComponentSource && mountedApp) {
        // Remount with new prop values
        await mountComponent(lastComponentSource);
      }
    }, { deep: true });

    // TODO 1: Initialize WebContainer and compile Vue SFC
    
    ////////********* COMPILE COMPONENT *******///////////////
    async function compileComponentHandler() {
      const source = editor.getValue();
      console.log("Compiling component:", source);
      
      // Store the source for remounting when props change
      lastComponentSource = source;

      try {
        // Extract props first (instant)
        propsSchema.value = extractProps(source);
        
        // Initialize default prop values
        if (propsSchema.value?.properties) {
          const defaultValues = {};
          Object.keys(propsSchema.value.properties).forEach((key) => {
            const prop = propsSchema.value.properties[key];
            defaultValues[key] = propValues.value[key] ?? prop.default ?? 
              (prop.type === 'number' ? 0 : prop.type === 'boolean' ? false : '');
          });
          propValues.value = defaultValues;
        }

        // Check if WebContainer is ready
        if (!webcontainer) {
          console.warn('WebContainer not ready, using direct mounting');
          await mountComponent(source);
          return;
        }

        try {
          // Write component source
          await webcontainer.fs.writeFile('/component.vue', source);

          // Create/update compiler script
          const compilerCode = `
import { readFileSync } from 'fs';
import { parse, compileScript, compileTemplate } from '@vue/compiler-sfc';

const source = readFileSync('/component.vue', 'utf-8');
const { descriptor } = parse(source);

const scriptResult = compileScript(descriptor, { id: 'compiled-component' });
const templateResult = compileTemplate({
  source: descriptor.template.content,
  id: 'compiled-component',
  filename: 'component.vue'
});

console.log(JSON.stringify({
  script: scriptResult.content,
  template: templateResult.code
}));
`;
          await webcontainer.fs.writeFile('/compile.mjs', compilerCode);

          // Run compilation (should be fast now - no npm install!)
          const compileProcess = await webcontainer.spawn('node', ['compile.mjs']);
          
          let output = '';
          compileProcess.output.pipeTo(new WritableStream({
            write(chunk) {
              output += chunk;
            }
          }));

          const exitCode = await compileProcess.exit;
          
          if (exitCode === 0 && output) {
            try {
              const compiled = JSON.parse(output);
              console.log('Compilation successful');
              await mountCompiledComponent(compiled, source);
            } catch (parseError) {
              console.warn('Failed to parse compiled output, using direct mounting:', parseError);
              await mountComponent(source);
            }
          } else {
            console.warn('Compilation failed, using direct mounting');
            await mountComponent(source);
          }
        } catch (wcError) {
          console.warn('WebContainer compilation error, using direct mounting:', wcError);
          await mountComponent(source);
        }
      } catch (error) {
        console.error('Compilation error:', error);
        await mountComponent(source);
      }
    }

    // Mount compiled component from WebContainer
    async function mountCompiledComponent(compiled, originalSource) {
      try {
        // Unmount previous component if exists
        if (mountedApp) {
          mountedApp.unmount();
          mountedApp = null;
          if (componentMount.value) {
            componentMount.value.innerHTML = '';
          }
        }

        // Execute compiled script to get component definition
        let componentDef = {};
        eval(compiled.script.replace(/export default/, 'componentDef ='));
        
        // Add compiled render function
        componentDef.render = eval(`(${compiled.template})`);

        // Create and mount app with props
        const app = createApp(componentDef, propValues.value);
        app.mount(componentMount.value);
        mountedApp = app;
      } catch (error) {
        console.error('Error mounting compiled component:', error);
        // Fallback to direct mounting
        await mountComponent(originalSource);
      }
    }

    // Mount component function (TODO 4)
    async function mountComponent(componentSource) {
      try {
        // Unmount previous component if exists
        if (mountedApp) {
          mountedApp.unmount();
          mountedApp = null;
          // Clear mount point
          if (componentMount.value) {
            componentMount.value.innerHTML = '';
          }
        }

        // Extract template and script
        const templateMatch = componentSource.match(/<template[^>]*>([\s\S]*?)<\/template>/);
        const scriptMatch = componentSource.match(/<script[^>]*>([\s\S]*?)<\/script>/);
        
        if (!templateMatch || !scriptMatch) {
          console.error('Invalid component format');
          return;
        }

        const template = templateMatch[1].trim();
        const scriptContent = scriptMatch[1];

        // Parse script to get component definition
        let componentDef = {};
        const scriptEval = scriptContent.replace(/export default/, 'componentDef =');
        
        try {
          // Execute in a safe context
          eval(scriptEval);
        } catch (e) {
          console.error('Error parsing component script:', e);
          return;
        }

        // Create a render function that processes the template
        // For a prototype, we'll compile the template to a render function
        componentDef.render = function() {
          // Get current props and data
          const props = this.$props || {};
          const data = this.$data || {};
          
          // Simple template compiler - replace {{ }} with actual values
          let processedTemplate = template;
          
          // Replace template variables
          processedTemplate = processedTemplate.replace(/\{\{([^}]+)\}\}/g, (match, expr) => {
            const keys = expr.trim().split('.');
            let value = this;
            
            for (const key of keys) {
              if (value && typeof value === 'object') {
                value = value[key] ?? props[key] ?? data[key] ?? '';
              } else {
                value = '';
                break;
              }
            }
            
            return value != null ? String(value) : '';
          });

          // Parse HTML and create VNodes
          // For simplicity, we'll use innerHTML for now and enhance later
          // Create a simple VNode structure
          const tempDiv = document.createElement('div');
          tempDiv.innerHTML = processedTemplate;
          
          // Convert to VNode structure
          const convertToVNode = (element) => {
            if (element.nodeType === Node.TEXT_NODE) {
              return element.textContent;
            }
            
            if (element.nodeType !== Node.ELEMENT_NODE) {
              return null;
            }
            
            const tag = element.tagName.toLowerCase();
            const children = Array.from(element.childNodes)
              .map(convertToVNode)
              .filter(child => child !== null);
            
            const props = {};
            Array.from(element.attributes).forEach(attr => {
              const name = attr.name;
              if (name.startsWith('@')) {
                // Event handler
                const eventName = name.slice(1);
                const handlerName = attr.value;
                if (componentDef.methods && componentDef.methods[handlerName]) {
                  const capitalizedEvent = eventName.charAt(0).toUpperCase() + eventName.slice(1);
                  props[`on${capitalizedEvent}`] = 
                    componentDef.methods[handlerName].bind(this);
                }
              } else if (name !== '@click') {
                props[name] = attr.value;
              }
            });
            
            // Handle @click separately (Vue uses onClick)
            const clickHandler = element.getAttribute('@click');
            if (clickHandler && componentDef.methods && componentDef.methods[clickHandler]) {
              props.onClick = componentDef.methods[clickHandler].bind(this);
            }
            
            return h(tag, props, children.length > 0 ? children : undefined);
          };
          
          const rootElement = tempDiv.firstElementChild || tempDiv;
          return convertToVNode(rootElement) || h('div', processedTemplate);
        };

        // Create and mount app with props
        const app = createApp(componentDef, propValues.value);
        app.mount(componentMount.value);
        mountedApp = app;

      } catch (error) {
        console.error('Error mounting component:', error);
        if (componentMount.value) {
          componentMount.value.innerHTML = `<div style="color: red; padding: 1rem;">Error mounting component: ${error.message}</div>`;
        }
      }
    }

    // TODO 2: Extract props from Vue component and convert to JSON Schema
    function extractProps(componentSource) {
      console.log('[DEBUG extractProps] Starting extraction...');

      // Helper function to extract individual prop
      function extractProp(str, startPos) {
        const substring = str.substring(startPos);
        const nameMatch = substring.match(/^\s*(\w+):\s*\{/);
        if (!nameMatch) return null;

        const propName = nameMatch[1];
        const matchLength = nameMatch[0].length;
        const propStart = startPos + matchLength - 1;

        let braceCount = 0;
        let propEnd = -1;

        for (let i = propStart; i < str.length; i++) {
          if (str[i] === '{') braceCount++;
          if (str[i] === '}') {
            braceCount--;
            if (braceCount === 0) {
              propEnd = i;
              break;
            }
          }
        }

        if (braceCount !== 0 || propEnd === -1) return null;

        const propDef = str.substring(propStart + 1, propEnd);
        const nextPos = propEnd + 1;

        return { propName, propDef, nextPos };
      }

      try {
        // Extract script section
        const scriptMatch = componentSource.match(/<script[^>]*>([\s\S]*?)<\/script>/);
        if (!scriptMatch) {
          console.log('[DEBUG extractProps] ❌ No <script> tag found!');
          return null;
        }

        const scriptContent = scriptMatch[1];
        
        // Find the start of props definition
        const propsStartMatch = scriptContent.match(/props:\s*\{/);
        if (!propsStartMatch) {
          console.log('[DEBUG extractProps] ❌ No props definition found!');
          return null;
        }

        // Get the position right after "props: {"
        const startPos = propsStartMatch.index + propsStartMatch[0].length - 1; // Include the opening brace
        
        // Count braces to find the matching closing brace
        let braceCount = 0;
        let endPos = -1;
        
        for (let i = startPos; i < scriptContent.length; i++) {
          if (scriptContent[i] === '{') {
            braceCount++;
          }
          if (scriptContent[i] === '}') {
            braceCount--;
            if (braceCount === 0) {
              endPos = i;
              break;
            }
          }
        }
        
        if (endPos === -1) {
          console.log('[DEBUG extractProps] ❌ Could not find matching closing brace!');
          return null;
        }
        
        // Extract the props content (without the outer braces)
        const propsString = scriptContent.substring(startPos + 1, endPos);
        console.log('[DEBUG extractProps] Props string:', propsString);
        
        const properties = {};
        
        // Parse each prop definition
        let pos = 0;
        let propCount = 0;
        
        while (pos < propsString.length) {
          // Skip whitespace and commas
          const whitespaceMatch = propsString.substring(pos).match(/^[\s,]+/);
          if (whitespaceMatch) {
            pos += whitespaceMatch[0].length;
          }
          if (pos >= propsString.length) break;
          
          const result = extractProp(propsString, pos);
          if (!result) break;
          
          propCount++;
          const { propName, propDef } = result;
          console.log(`[DEBUG extractProps] Processing prop #${propCount}:`, propName);
          
          const schemaProp = {};
          
          // Extract type
          const typeMatch = propDef.match(/type:\s*(\w+)/);
          if (typeMatch) {
            const vueType = typeMatch[1];
            if (vueType === 'String') {
              schemaProp.type = 'string';
            } else if (vueType === 'Number') {
              schemaProp.type = 'number';
            } else if (vueType === 'Boolean') {
              schemaProp.type = 'boolean';
            } else if (vueType === 'Array') {
              schemaProp.type = 'array';
            } else if (vueType === 'Object') {
              schemaProp.type = 'object';
            } else {
              schemaProp.type = 'string'; // Fallback for unknown types
            }
          } else {
            schemaProp.type = 'string';
          }
          
          // Extract default value
          const defaultMatch = propDef.match(/default:\s*(.+?)(?:,|\s*$)/s);
          if (defaultMatch) {
            const defaultStr = defaultMatch[1].trim();
            // Remove trailing comma if present
            const cleanDefault = defaultStr.replace(/,\s*$/, '').trim();
            
            // Skip function defaults (e.g., default: () => [], default: function() {...})
            if (cleanDefault.startsWith('() =>') || 
                cleanDefault.startsWith('function') || 
                cleanDefault.match(/^\([^)]*\)\s*=>/)) {
              // Skip function defaults - don't set schemaProp.default
              // Function defaults need to be executed, which we can't do statically
            } else if (cleanDefault.startsWith("'") || cleanDefault.startsWith('"')) {
              schemaProp.default = cleanDefault.slice(1, -1);
            } else if (!isNaN(cleanDefault) && cleanDefault !== '') {
              schemaProp.default = Number(cleanDefault);
            } else if (cleanDefault === 'true' || cleanDefault === 'false') {
              schemaProp.default = cleanDefault === 'true';
            } else {
              schemaProp.default = cleanDefault;
            }
          }
          
          // Extract mvt metadata
          const mvtMatch = propDef.match(/mvt:\s*\{([\s\S]*?)\}/);
          if (mvtMatch) {
            const mvtContent = mvtMatch[1];
            
            const descMatch = mvtContent.match(/description:\s*['"](.*?)['"]/);
            if (descMatch) {
              schemaProp.description = descMatch[1];
            }
            
            if (schemaProp.type === 'number') {
              const minMatch = mvtContent.match(/min:\s*(\d+)/);
              if (minMatch) {
                schemaProp.minimum = Number(minMatch[1]);
              }
              
              const maxMatch = mvtContent.match(/max:\s*(\d+)/);
              if (maxMatch) {
                schemaProp.maximum = Number(maxMatch[1]);
              }
            }
          }
          
          properties[propName] = schemaProp;
          pos = result.nextPos;
        }
        
        console.log(`[DEBUG extractProps] Total props found: ${Object.keys(properties).length}`);
        
        if (Object.keys(properties).length === 0) {
          return null;
        }
        
        return {
          $schema: "https://json-schema.org/draft/2020-12/schema",
          type: "object",
          properties,
        };
      } catch (error) {
        console.error('[DEBUG extractProps] ❌ Error extracting props:', error);
        return null;
      }
    }

    return {
      editorElement,
      componentMount,
      propsSchema,
      propValues,
      compileComponent: compileComponentHandler,
    };
  },
};
</script>


