<script>
  import { onMount, setContext, createEventDispatcher } from "svelte";
  import { writable } from "svelte/store";
  import SplitPane from "./SplitPane.svelte";
  import CodeMirror from "./CodeMirror.svelte";
  import ModuleEditor from "./Input/ModuleEditor.svelte";
  import Output from "./Output/index.svelte";
  import Bundler from "./Bundler.js";
  import { is_browser } from "./env.js";
  import { mdsvex } from "mdsvex";
  import attrs from "markdown-it-attrs";

  export let svelteUrl = "https://unpkg.com/svelte";
  export let rollupUrl = "https://unpkg.com/rollup/dist/rollup.browser.js";
  export let embedded = false;
  export let orientation = "columns";
  export let fixed = false;
  export let fixedPos = 50;
  export let injectedJS = "";
  export let injectedCSS = "";
  export let zoom = 75;

  const mds = mdsvex({
    // you can add markdown-it options here, html is always true
    markdownOptions: {
      typographer: true,
      linkify: true
    },
    parser: md => md.use(attrs)
  });

  const preprocess = v =>
    v.type === "svexy"
      ? {
        ...v,
        source: mds.markup({
          content: v.source,
          filename: v.name + ".svexy"
        }).code
      }
      : v;
  export function toJSON() {
    // TODO there's a bug here — Svelte hoists this function because
    // it wrongly things that $components is global. Needs to
    // factor in $ variables when determining hoistability

    svelteUrl; // workaround

    return {
      imports: $bundle.imports,
      components: $components
    };
  }

  export async function set(data) {
    components.set(data.components);
    selected.set(data.components[0]);

    rebundle();

    await module_editor_ready;
    await output_ready;

    injectedCSS = data.css || "";
    module_editor.set($selected.source, $selected.type);
    output.set($selected, $compile_options);
  }

  export function update(data) {
    const { name, type } = $selected || {};

    components.set(data.components);
    const matched_component = data.components.find(
      file => file.name === name && file.type === type
    );
    selected.set(matched_component || data.components[0]);

    injectedCSS = data.css || "";

    if (matched_component) {
      module_editor.update(matched_component.source);
      output.update(matched_component, $compile_options);
    } else {
      module_editor.set(matched_component.source, matched_component.type);
      output.set(matched_component, $compile_options);
    }
  }

  const dispatch = createEventDispatcher();

  const components = writable([]);
  const selected = writable(null);
  const bundle = writable(null);

  const compile_options = writable({
    generate: "dom",
    dev: false,
    css: false,
    hydratable: false,
    customElement: false,
    immutable: false,
    legacy: false
  });

  let module_editor;
  let output;

  let current_token;
  async function rebundle() {
    const token = (current_token = {});
    const result = await bundler.bundle($components.map(preprocess));
    if (result && token === current_token) bundle.set(result);
  }

  // TODO this is a horrible kludge, written in a panic. fix it
  let fulfil_module_editor_ready;
  let module_editor_ready = new Promise(f => (fulfil_module_editor_ready = f));

  let fulfil_output_ready;
  let output_ready = new Promise(f => (fulfil_output_ready = f));

  setContext("REPL", {
    components,
    selected,
    bundle,
    compile_options,

    rebundle,

    handle_change: event => {
      selected.update(component => {
        // TODO this is a bit hacky — we're relying on mutability
        // so that updating components works... might be better
        // if a) components had unique IDs, b) we tracked selected
        // *index* rather than component, and c) `selected` was
        // derived from `components` and `index`
        component.source = event.detail.value;
        return component;
      });

      components.update(c => c);
      output.update($selected, $compile_options);

      rebundle();

      dispatch("change", {
        components: $components
      });
    },

    register_module_editor(editor) {
      module_editor = editor;
      fulfil_module_editor_ready();
    },

    register_output(handlers) {
      output = handlers;
      fulfil_output_ready();
    },

    request_focus() {
      module_editor.focus();
    }
  });

  let workers;

  let input;
  let sourceErrorLoc;

  const bundler = is_browser && new Bundler(svelteUrl, rollupUrl);

  $: if (output && $selected) {
    output.update($selected, $compile_options);
  }
</script>

<style scoped>
  .playground-container {
    position: relative;
    width: 100%;
    height: 100%;
  }

  .playground-container .pane {
    position: relative;
    padding: 42px 0 0 0;
    height: 100%;
    box-sizing: border-box;
  }

  .playground-container .pane > :global(*):first-child {
    position: absolute;
    top: 0;
    left: 0;
    width: 100%;
    height: 42px;
    box-sizing: border-box;
  }

  .playground-container .pane > :global(*):last-child {
    width: 100%;
    height: 100%;
  }
</style>

<div class="playground-container" class:orientation>
  <SplitPane
    type={orientation === 'rows' ? 'vertical' : 'horizontal'}
    pos={45}
    {fixed}>
    <div class="pane" slot="a">
      <ModuleEditor
        bind:this={input}
        errorLoc={sourceErrorLoc} />
    </div>

    <div class="pane" slot="b" style="height: 100%;">
      <Output {zoom} {svelteUrl} {injectedJS} {injectedCSS} />
    </div>
  </SplitPane>
</div>
