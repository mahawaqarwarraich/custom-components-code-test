<template>
  <div class="container">
    <div class="editor-panel">
      <h2>Component Editor</h2>
      <textarea ref="editorElement"></textarea>
      <button @click="compileComponent" class="compile-btn">Compile Component</button>
    </div>

    <div class="right-panel">
      <div class="props-panel">
        <h2>Props Schema</h2>
        <div class="props-schema">
          <pre v-if="!propsSchema" class="empty-state">
Compile a component to see props schema</pre>
          <pre v-else>{{ JSON.stringify(propsSchema, null, 2) }}</pre>
        </div>

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
        <iframe ref="previewFrame" class="component-mount"></iframe>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, onMounted, watch } from "vue";
import CodeMirror from "codemirror";
import "codemirror/lib/codemirror.css";
import "codemirror/mode/vue/vue.js";
import { WebContainer } from "@webcontainer/api";

export default {
  name: "App",
  setup() {
    const editorElement = ref(null);
    const previewFrame = ref(null);
    const propsSchema = ref(null);
    const propValues = ref({});
    let editor = null;
    let webcontainer = null;

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
    increment() {
      this.count++
    }
  }
}
<\/script>`;

    // Initialize WebContainer and CodeMirror
    onMounted(async () => {
      editor = CodeMirror.fromTextArea(editorElement.value, {
        mode: "vue",
        theme: "default",
        lineNumbers: true,
      });
      editor.setValue(sampleComponent);

      console.log("Booting WebContainer...");
      webcontainer = await WebContainer.boot();

      // Ensure directories exist
      await webcontainer.fs.mkdir("/src", { recursive: true });

      const files = {
        "package.json": {
          file: {
            contents: 
             `{
  "name": "vue-sfc-playground",
  "private": true,
  "scripts": {
    "dev": "vite --port 5173 --host 0.0.0.0",
    "build": "vite build",
    "preview": "vite preview"
  },
  "dependencies": {
    "vue": "latest"
  },
  "devDependencies": {
    "vite": "latest",
    "@vitejs/plugin-vue": "latest"
  }
}`
          
          },
        },
        "vite.config.js": {
          file: {
            contents: `
import vue from '@vitejs/plugin-vue'
export default {
  plugins: [vue()]
}
`,
          },
        },
        "index.html": {
          file: {
            contents: '<!DOCTYPE html>\n<' + 'html>\n  <' + 'head>\n    <title>Preview</title>\n    <meta charset="UTF-8" />\n  </' + 'head>\n  <' + 'body>\n    <div id="app"></div>\n    <' + 'script type="module" src="/src/main.js"></' + 'script>\n  </' + 'body>\n</' + 'html>',
          },
        },
        src: {
          directory: {
            "main.js": {
              file: {
                contents: `
import { createApp } from 'vue'
import App from './App.vue'
createApp(App).mount('#app')
`,
              },
            },
            "App.vue": {
              file: {
                contents: `${sampleComponent}`,
              },
            },
          },
        },
      };

      await webcontainer.mount(files);
      
      // Add delay to ensure files are written
      await new Promise(r => setTimeout(r, 200));
      console.log("Installing dependencies...");
      const installProcess = await webcontainer.spawn("npm", ["install"]);
      await installProcess.exit;
      console.log("Dependencies installed.");

      console.log("Starting Vite dev server...");
      const serverProcess = await webcontainer.spawn("npm", ["run", "dev"]);

//       const output = await serverProcess.output;
// for await (const chunk of output) {
//   console.log("i am a chunk of server process", chunk);
// }

      console.log("after running npm run dev command");
      const url = "http://localhost:5173";
      webcontainer.on("server-ready", (port, url) => {
        console.log("Vite server ready at", url);
        if (previewFrame.value) previewFrame.value.src = url;
      });
     // if (previewFrame.value) previewFrame.value.src = url;
    });
   

    // Recompile when user clicks button
    async function compileComponent() {
      const source = editor.getValue();
      propsSchema.value = extractProps(source);
      await webcontainer.fs.writeFile("/src/App.vue", source);
      console.log("Updated App.vue in WebContainer.");

      // Update props in iframe dynamically
      if (previewFrame.value?.contentWindow) {
        previewFrame.value.contentWindow.postMessage(
          { type: "updateProps", props: JSON.parse(JSON.stringify(propValues.value))},
          "*"
        );
      }
    }

    // Watch prop changes and recompile
    watch(
      propValues,
      async () => {
        await compileComponent();
      },
      { deep: true }
    );

   
    function extractProps(componentSource) {
      const scriptMatch = componentSource.match(/<script[^>]*>([\s\S]*?)<\/script>/);
      if (!scriptMatch) return null;

      const scriptContent = scriptMatch[1];
      const propsStart = scriptContent.indexOf("props:");
      if (propsStart === -1) return null;

      const propsBlock = scriptContent.slice(propsStart);
      const braceStart = propsBlock.indexOf("{");
      let braceCount = 0,
        end = -1;
      for (let i = braceStart; i < propsBlock.length; i++) {
        if (propsBlock[i] === "{") braceCount++;
        if (propsBlock[i] === "}") braceCount--;
        if (braceCount === 0) {
          end = i;
          break;
        }
      }
      if (end === -1) return null;

      const propsString = propsBlock.slice(braceStart + 1, end);
      const properties = {};
      const regex = /(\w+)\s*:\s*\{([\s\S]*?)\}/g;
      let match;
      while ((match = regex.exec(propsString))) {
        const [_, name, body] = match;
        const typeMatch = body.match(/type:\s*(\w+)/);
        const defaultMatch = body.match(/default:\s*(.+?)(,|\n|$)/);
        const typeMap = {
          String: "string",
          Number: "number",
          Boolean: "boolean",
        };
        const schema = { type: typeMap[typeMatch?.[1]] || "string" };
        if (defaultMatch) {
          const val = defaultMatch[1].trim().replace(/['",]/g, "");
          schema.default =
            schema.type === "number" ? Number(val) : val || "";
        }
        properties[name] = schema;
      }
      return {
        $schema: "https://json-schema.org/draft/2020-12/schema",
        type: "object",
        properties,
      };
    }

    return { editorElement, previewFrame, propsSchema, propValues, compileComponent };
  },
};
</script>

<style>
.container {
  display: flex;
  gap: 1rem;
  padding: 1rem;
}
.editor-panel,
.right-panel {
  flex: 1;
}
.component-mount {
  width: 100%;
  height: 400px;
  border: 1px solid #ccc;
}
.compile-btn {
  margin-top: 1rem;
}
</style>
