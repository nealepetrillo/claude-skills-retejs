# Rete.js v2 Code Examples

Comprehensive examples for building node-based editors with Rete.js v2.

## Complete Minimal Editor (Lit)

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="utf-8">
  <title>Rete.js Editor</title>
  <script type="importmap">
  {
    "imports": {
      "rete": "https://cdn.jsdelivr.net/npm/rete@2/+esm",
      "rete-area-plugin": "https://cdn.jsdelivr.net/npm/rete-area-plugin@2/+esm",
      "rete-connection-plugin": "https://cdn.jsdelivr.net/npm/rete-connection-plugin@2/+esm",
      "@retejs/lit-plugin": "https://cdn.jsdelivr.net/npm/@retejs/lit-plugin@2/+esm",
      "rete-render-utils": "https://cdn.jsdelivr.net/npm/rete-render-utils@2/+esm",
      "lit": "https://cdn.jsdelivr.net/npm/lit@3/+esm"
    }
  }
  </script>
  <style>
    body { margin: 0; font-family: system-ui, sans-serif; }
    #editor { width: 100vw; height: 100vh; }
  </style>
</head>
<body>
  <div id="editor"></div>
  <script type="module">
    import { NodeEditor, ClassicPreset, getUID } from 'rete';
    import { AreaPlugin, AreaExtensions } from 'rete-area-plugin';
    import { ConnectionPlugin, Presets as ConnectionPresets } from 'rete-connection-plugin';
    import { LitPlugin, Presets } from '@retejs/lit-plugin';

    const socket = new ClassicPreset.Socket('socket');

    class NumberNode extends ClassicPreset.Node {
      constructor(initial = 0) {
        super('Number');
        this.width = 180;
        this.height = 120;
        this.addControl('value', new ClassicPreset.InputControl('number', { initial }));
        this.addOutput('value', new ClassicPreset.Output(socket, 'Value'));
      }
      data() {
        return { value: this.controls.value.value ?? 0 };
      }
    }

    class AddNode extends ClassicPreset.Node {
      constructor() {
        super('Add');
        this.width = 180;
        this.height = 195;
        this.addInput('a', new ClassicPreset.Input(socket, 'A'));
        this.addInput('b', new ClassicPreset.Input(socket, 'B'));
        this.addOutput('sum', new ClassicPreset.Output(socket, 'Sum'));
      }
      data(inputs) {
        const a = inputs.a?.[0] ?? 0;
        const b = inputs.b?.[0] ?? 0;
        return { sum: a + b };
      }
    }

    async function createEditor(container) {
      const editor = new NodeEditor();
      const area = new AreaPlugin(container);
      const connection = new ConnectionPlugin();
      const render = new LitPlugin();

      render.addPreset(Presets.classic.setup());
      connection.addPreset(ConnectionPresets.classic.setup());

      editor.use(area);
      area.use(connection);
      area.use(render);

      AreaExtensions.selectableNodes(area, AreaExtensions.selector(), {
        accumulating: AreaExtensions.accumulateOnCtrl()
      });
      AreaExtensions.simpleNodesOrder(area);

      const num1 = new NumberNode(5);
      const num2 = new NumberNode(10);
      const add = new AddNode();

      await editor.addNode(num1);
      await editor.addNode(num2);
      await editor.addNode(add);

      await area.translate(num1.id, { x: 0, y: 0 });
      await area.translate(num2.id, { x: 0, y: 200 });
      await area.translate(add.id, { x: 300, y: 100 });

      await editor.addConnection(new ClassicPreset.Connection(num1, 'value', add, 'a'));
      await editor.addConnection(new ClassicPreset.Connection(num2, 'value', add, 'b'));

      AreaExtensions.zoomAt(area, editor.getNodes());

      return { editor, area };
    }

    createEditor(document.getElementById('editor'));
  </script>
</body>
</html>
```

## Custom Node Types

### Node with Multiple Controls
```typescript
class TextProcessorNode extends ClassicPreset.Node {
  width = 200;
  height = 220;

  constructor() {
    super('Text Processor');

    this.addInput('text', new ClassicPreset.Input(textSocket, 'Input'));
    this.addOutput('text', new ClassicPreset.Output(textSocket, 'Output'));

    this.addControl('prefix', new ClassicPreset.InputControl('text', {
      initial: '',
      readonly: false
    }));
    this.addControl('suffix', new ClassicPreset.InputControl('text', {
      initial: '',
      readonly: false
    }));
    this.addControl('uppercase', new ClassicPreset.InputControl('checkbox', {
      initial: false
    }));
  }

  data(inputs: { text?: string[] }): { text: string } {
    const input = inputs.text?.[0] ?? '';
    const prefix = this.controls.prefix.value ?? '';
    const suffix = this.controls.suffix.value ?? '';
    const uppercase = this.controls.uppercase.value ?? false;

    let result = prefix + input + suffix;
    if (uppercase) result = result.toUpperCase();

    return { text: result };
  }
}
```

### Node with Dropdown Selection
```typescript
// Custom control class for select/dropdown
class SelectControl extends ClassicPreset.Control {
  constructor(
    public value: string,
    public options: { value: string; label: string }[],
    public onChange?: (value: string) => void
  ) {
    super();
  }
}

class MathOperationNode extends ClassicPreset.Node {
  width = 180;
  height = 195;

  constructor() {
    super('Math');

    this.addInput('a', new ClassicPreset.Input(numberSocket, 'A'));
    this.addInput('b', new ClassicPreset.Input(numberSocket, 'B'));
    this.addOutput('result', new ClassicPreset.Output(numberSocket, 'Result'));

    this.addControl('operation', new SelectControl('add', [
      { value: 'add', label: 'Add (+)' },
      { value: 'subtract', label: 'Subtract (-)' },
      { value: 'multiply', label: 'Multiply (*)' },
      { value: 'divide', label: 'Divide (/)' }
    ]));
  }

  data(inputs: { a?: number[]; b?: number[] }): { result: number } {
    const a = inputs.a?.[0] ?? 0;
    const b = inputs.b?.[0] ?? 0;
    const op = (this.controls.operation as SelectControl).value;

    switch (op) {
      case 'add': return { result: a + b };
      case 'subtract': return { result: a - b };
      case 'multiply': return { result: a * b };
      case 'divide': return { result: b !== 0 ? a / b : 0 };
      default: return { result: 0 };
    }
  }
}
```

### Node with Dynamic Inputs
```typescript
class MergeNode extends ClassicPreset.Node {
  width = 180;
  height = 120;
  inputCount = 2;

  constructor(inputs = 2) {
    super('Merge');
    this.inputCount = inputs;

    for (let i = 0; i < inputs; i++) {
      this.addInput(`in${i}`, new ClassicPreset.Input(anySocket, `Input ${i + 1}`));
    }
    this.addOutput('out', new ClassicPreset.Output(arraySocket, 'Array'));

    this.height = 80 + inputs * 40;
  }

  addDynamicInput() {
    const key = `in${this.inputCount}`;
    this.addInput(key, new ClassicPreset.Input(anySocket, `Input ${this.inputCount + 1}`));
    this.inputCount++;
    this.height = 80 + this.inputCount * 40;
  }

  data(inputs: Record<string, unknown[]>): { out: unknown[] } {
    const result: unknown[] = [];
    for (let i = 0; i < this.inputCount; i++) {
      const val = inputs[`in${i}`]?.[0];
      if (val !== undefined) result.push(val);
    }
    return { out: result };
  }
}
```

## Custom Socket Types

### Type-Safe Socket System
```typescript
// Base abstract socket
abstract class TypedSocket extends ClassicPreset.Socket {
  abstract isCompatibleWith(socket: ClassicPreset.Socket): boolean;
}

// Number socket
class NumberSocket extends TypedSocket {
  constructor() { super('Number'); }
  isCompatibleWith(socket: ClassicPreset.Socket): boolean {
    return socket instanceof NumberSocket;
  }
}

// String socket
class StringSocket extends TypedSocket {
  constructor() { super('String'); }
  isCompatibleWith(socket: ClassicPreset.Socket): boolean {
    return socket instanceof StringSocket;
  }
}

// Boolean socket
class BooleanSocket extends TypedSocket {
  constructor() { super('Boolean'); }
  isCompatibleWith(socket: ClassicPreset.Socket): boolean {
    return socket instanceof BooleanSocket;
  }
}

// Any socket (connects to anything)
class AnySocket extends TypedSocket {
  constructor() { super('Any'); }
  isCompatibleWith(socket: ClassicPreset.Socket): boolean {
    return true;
  }
}

// Array socket (parameterized)
class ArraySocket extends TypedSocket {
  constructor(public elementType: TypedSocket) {
    super(`Array<${elementType.name}>`);
  }
  isCompatibleWith(socket: ClassicPreset.Socket): boolean {
    if (socket instanceof ArraySocket) {
      return this.elementType.isCompatibleWith(socket.elementType);
    }
    return false;
  }
}
```

### Validation Pipeline
```typescript
import { getConnectionSockets } from 'rete-render-utils';

function setupValidation(editor: NodeEditor<Schemes>) {
  editor.addPipe((context) => {
    if (context.type === 'connectioncreate') {
      const { source, target } = getConnectionSockets(editor, context.data);

      if (!source || !target) {
        console.warn('Invalid connection: missing socket');
        return; // Block
      }

      if (source instanceof TypedSocket && !source.isCompatibleWith(target)) {
        console.warn(`Type mismatch: ${source.name} -> ${target.name}`);
        return; // Block
      }

      // Check for cycles (optional)
      if (wouldCreateCycle(editor, context.data)) {
        console.warn('Connection would create cycle');
        return; // Block
      }
    }
    return context;
  });
}

function wouldCreateCycle(editor: NodeEditor<Schemes>, newConnection: Schemes['Connection']): boolean {
  const visited = new Set<string>();

  function dfs(nodeId: string): boolean {
    if (nodeId === newConnection.source) return true;
    if (visited.has(nodeId)) return false;
    visited.add(nodeId);

    const incomingConnections = editor.getConnections()
      .filter(c => c.target === nodeId);

    for (const conn of incomingConnections) {
      if (dfs(conn.source)) return true;
    }
    return false;
  }

  return dfs(newConnection.target);
}
```

## Dataflow Processing

### Complete Dataflow Example
```typescript
import { DataflowEngine } from 'rete-engine';

// Nodes for math operations
class ConstantNode extends ClassicPreset.Node {
  constructor(value: number) {
    super('Constant');
    this.addControl('value', new ClassicPreset.InputControl('number', { initial: value }));
    this.addOutput('value', new ClassicPreset.Output(numberSocket, 'Value'));
  }
  data() {
    return { value: this.controls.value.value ?? 0 };
  }
}

class MultiplyNode extends ClassicPreset.Node {
  constructor() {
    super('Multiply');
    this.addInput('a', new ClassicPreset.Input(numberSocket, 'A'));
    this.addInput('b', new ClassicPreset.Input(numberSocket, 'B'));
    this.addOutput('product', new ClassicPreset.Output(numberSocket, 'Product'));
  }
  data(inputs: { a?: number[]; b?: number[] }) {
    return { product: (inputs.a?.[0] ?? 1) * (inputs.b?.[0] ?? 1) };
  }
}

class DisplayNode extends ClassicPreset.Node {
  constructor() {
    super('Display');
    this.addInput('value', new ClassicPreset.Input(numberSocket, 'Value'));
    this.addControl('display', new ClassicPreset.InputControl('text', { readonly: true }));
  }
  data(inputs: { value?: number[] }) {
    const val = inputs.value?.[0] ?? 0;
    (this.controls.display as ClassicPreset.InputControl<'text'>).setValue(String(val));
    return {};
  }
}

// Setup engine
const dataflow = new DataflowEngine<Schemes>();
editor.use(dataflow);

// Process on changes
editor.addPipe((context) => {
  if (['connectioncreated', 'connectionremoved', 'nodecreated'].includes(context.type)) {
    // Find display nodes and update them
    setTimeout(async () => {
      dataflow.reset();
      for (const node of editor.getNodes()) {
        if (node instanceof DisplayNode) {
          await dataflow.fetch(node.id);
          area.update('node', node.id);
        }
      }
    }, 10);
  }
  return context;
});
```

## Control Flow Processing

### Event-Driven Execution
```typescript
import { ControlFlowEngine } from 'rete-engine';

const execSocket = new ClassicPreset.Socket('Exec');

class StartNode extends ClassicPreset.Node {
  constructor() {
    super('Start');
    this.addOutput('exec', new ClassicPreset.Output(execSocket, 'Start'));
  }
  execute(_: never, forward: (output: 'exec') => void) {
    console.log('Execution started');
    forward('exec');
  }
}

class DelayNode extends ClassicPreset.Node {
  constructor(ms: number = 1000) {
    super('Delay');
    this.addInput('exec', new ClassicPreset.Input(execSocket, 'In'));
    this.addOutput('exec', new ClassicPreset.Output(execSocket, 'Out'));
    this.addControl('delay', new ClassicPreset.InputControl('number', { initial: ms }));
  }
  async execute(_: 'exec', forward: (output: 'exec') => void) {
    const delay = this.controls.delay.value ?? 1000;
    await new Promise(resolve => setTimeout(resolve, delay));
    forward('exec');
  }
}

class BranchNode extends ClassicPreset.Node {
  constructor() {
    super('Branch');
    this.addInput('exec', new ClassicPreset.Input(execSocket, 'In'));
    this.addInput('condition', new ClassicPreset.Input(boolSocket, 'Condition'));
    this.addOutput('true', new ClassicPreset.Output(execSocket, 'True'));
    this.addOutput('false', new ClassicPreset.Output(execSocket, 'False'));
  }
  execute(_: 'exec', forward: (output: 'true' | 'false') => void, inputs: { condition?: boolean[] }) {
    const condition = inputs.condition?.[0] ?? false;
    forward(condition ? 'true' : 'false');
  }
}

class PrintNode extends ClassicPreset.Node {
  constructor(message: string = 'Hello') {
    super('Print');
    this.addInput('exec', new ClassicPreset.Input(execSocket, 'In'));
    this.addOutput('exec', new ClassicPreset.Output(execSocket, 'Out'));
    this.addControl('message', new ClassicPreset.InputControl('text', { initial: message }));
  }
  execute(_: 'exec', forward: (output: 'exec') => void) {
    console.log(this.controls.message.value);
    forward('exec');
  }
}

// Setup control flow
const controlflow = new ControlFlowEngine<Schemes>();
editor.use(controlflow);

// Run execution
document.getElementById('run')?.addEventListener('click', () => {
  const startNodes = editor.getNodes().filter(n => n instanceof StartNode);
  for (const start of startNodes) {
    controlflow.execute(start.id);
  }
});
```

## Serialization

### JSON Export/Import
```typescript
interface NodeData {
  id: string;
  type: string;
  x: number;
  y: number;
  controls: Record<string, unknown>;
}

interface ConnectionData {
  id: string;
  source: string;
  sourceOutput: string;
  target: string;
  targetInput: string;
}

interface GraphData {
  nodes: NodeData[];
  connections: ConnectionData[];
}

// Node registry for reconstruction
const nodeRegistry: Record<string, (data: Record<string, unknown>) => ClassicPreset.Node> = {
  'Number': (d) => new NumberNode(d.value as number ?? 0),
  'Add': () => new AddNode(),
  'Multiply': () => new MultiplyNode(),
  'Display': () => new DisplayNode(),
  // Add all node types...
};

async function exportGraph(
  editor: NodeEditor<Schemes>,
  area: AreaPlugin<Schemes, AreaExtra>
): Promise<GraphData> {
  const nodes: NodeData[] = [];

  for (const node of editor.getNodes()) {
    const view = area.nodeViews.get(node.id);
    const controls: Record<string, unknown> = {};

    for (const [key, control] of Object.entries(node.controls)) {
      if (control instanceof ClassicPreset.InputControl) {
        controls[key] = control.value;
      }
    }

    nodes.push({
      id: node.id,
      type: node.label,
      x: view?.position.x ?? 0,
      y: view?.position.y ?? 0,
      controls
    });
  }

  const connections: ConnectionData[] = editor.getConnections().map(c => ({
    id: c.id,
    source: c.source,
    sourceOutput: c.sourceOutput,
    target: c.target,
    targetInput: c.targetInput
  }));

  return { nodes, connections };
}

async function importGraph(
  editor: NodeEditor<Schemes>,
  area: AreaPlugin<Schemes, AreaExtra>,
  data: GraphData
): Promise<void> {
  // Clear existing
  for (const conn of [...editor.getConnections()]) {
    await editor.removeConnection(conn.id);
  }
  for (const node of [...editor.getNodes()]) {
    await editor.removeNode(node.id);
  }

  // Create nodes
  for (const nodeData of data.nodes) {
    const factory = nodeRegistry[nodeData.type];
    if (!factory) {
      console.warn(`Unknown node type: ${nodeData.type}`);
      continue;
    }

    const node = factory(nodeData.controls);
    node.id = nodeData.id;

    // Restore control values
    for (const [key, value] of Object.entries(nodeData.controls)) {
      const control = node.controls[key];
      if (control instanceof ClassicPreset.InputControl) {
        control.setValue(value as never);
      }
    }

    await editor.addNode(node);
    await area.translate(node.id, { x: nodeData.x, y: nodeData.y });
  }

  // Create connections
  for (const connData of data.connections) {
    const source = editor.getNode(connData.source);
    const target = editor.getNode(connData.target);

    if (!source || !target) continue;

    const connection = new ClassicPreset.Connection(
      source, connData.sourceOutput,
      target, connData.targetInput
    );
    connection.id = connData.id;

    await editor.addConnection(connection);
  }

  AreaExtensions.zoomAt(area, editor.getNodes());
}

// Usage
const json = await exportGraph(editor, area);
localStorage.setItem('graph', JSON.stringify(json));

const saved = JSON.parse(localStorage.getItem('graph') ?? '{}');
await importGraph(editor, area, saved);
```

## Context Menu Setup

```typescript
import { ContextMenuPlugin, Presets as ContextMenuPresets } from 'rete-context-menu-plugin';

const contextMenu = new ContextMenuPlugin<Schemes>({
  items: ContextMenuPresets.classic.setup([
    // Single items
    ['Number', () => new NumberNode()],
    ['Display', () => new DisplayNode()],

    // Nested categories
    ['Math', [
      ['Add', () => new AddNode()],
      ['Subtract', () => new SubtractNode()],
      ['Multiply', () => new MultiplyNode()],
      ['Divide', () => new DivideNode()]
    ]],

    ['Logic', [
      ['Branch', () => new BranchNode()],
      ['Compare', () => new CompareNode()]
    ]],

    ['Flow', [
      ['Start', () => new StartNode()],
      ['Delay', () => new DelayNode()],
      ['Print', () => new PrintNode()]
    ]]
  ])
});

area.use(contextMenu);

// Add rendering preset for context menu
render.addPreset(Presets.contextMenu.setup());
```

## Undo/Redo Integration

```typescript
import {
  HistoryPlugin,
  HistoryActions,
  HistoryExtensions,
  Presets as HistoryPresets
} from 'rete-history-plugin';

const history = new HistoryPlugin<Schemes, HistoryActions<Schemes>>();
history.addPreset(HistoryPresets.classic.setup());
area.use(history);

// Keyboard shortcuts (Ctrl+Z / Ctrl+Y)
HistoryExtensions.keyboard(history);

// UI Buttons
document.getElementById('undo')?.addEventListener('click', () => history.undo());
document.getElementById('redo')?.addEventListener('click', () => history.redo());

// Get history state
const snapshot = history.getHistorySnapshot();
console.log(`History has ${snapshot.length} entries`);

// Group related actions (e.g., batch node creation)
history.separate(); // Mark boundary
await editor.addNode(node1);
await editor.addNode(node2);
await editor.addConnection(connection);
history.separate(); // These will undo as a group
```

## Auto Arrange Layout

```typescript
import { AutoArrangePlugin, Presets as ArrangePresets, ArrangeAppliers } from 'rete-auto-arrange-plugin';

const arrange = new AutoArrangePlugin<Schemes>();
arrange.addPreset(ArrangePresets.classic.setup());
area.use(arrange);

// Basic layout
await arrange.layout();

// Animated layout
const applier = new ArrangeAppliers.TransitionApplier<Schemes, AreaExtra>({
  duration: 500,
  timingFunction: (t) => t * (2 - t), // ease-out
  async onTick() {
    await AreaExtensions.zoomAt(area, editor.getNodes());
  }
});

await arrange.layout({ applier });

// Custom spacing
await arrange.layout({
  options: {
    'elk.spacing.nodeNode': 80,
    'elk.layered.spacing.nodeNodeBetweenLayers': 100,
    'elk.direction': 'RIGHT' // LEFT, RIGHT, UP, DOWN
  }
});
```

## Minimap

```typescript
import { MinimapPlugin, MinimapExtra } from 'rete-minimap-plugin';

// Ensure nodes have dimensions
class NodeWithSize extends ClassicPreset.Node {
  width = 180;  // Required for minimap
  height = 120; // Required for minimap
}

// Add to type
type AreaExtra = LitArea2D<Schemes> | MinimapExtra;

const minimap = new MinimapPlugin<Schemes>();
area.use(minimap);

// Configure rendering
render.addPreset(Presets.minimap.setup({
  size: 200 // Size in pixels
}));
```

## Read-Only Mode

```typescript
import { ReadonlyPlugin } from 'rete-readonly-plugin';

const readonly = new ReadonlyPlugin<Schemes>();

// IMPORTANT: Order matters
editor.use(readonly.root);  // Before area
editor.use(area);
area.use(readonly.area);    // Before other area plugins
area.use(connection);
area.use(render);

// Initially editable, enable when needed
// readonly.enable();

// Toggle with button
document.getElementById('toggle-readonly')?.addEventListener('click', () => {
  if (readonly.enabled) {
    readonly.disable();
  } else {
    readonly.enable();
  }
});

// Note: For full readonly, also remove ConnectionPlugin
// The readonly plugin blocks editor operations but ConnectionPlugin
// still allows drag-to-connect interactions
```

## Custom Connection Styles

```typescript
// Connection with custom styling (Lit)
import { html, css, LitElement } from 'lit';
import { customElement, property } from 'lit/decorators.js';

@customElement('custom-connection')
class CustomConnection extends LitElement {
  static styles = css`
    :host { display: contents; }
    path.connection {
      fill: none;
      stroke: #4477AA;
      stroke-width: 3px;
      transition: stroke 0.2s;
    }
    path.connection:hover { stroke: #EE6677; }
    path.connection.selected { stroke: #228833; stroke-width: 4px; }
  `;

  @property() path = '';
  @property({ type: Boolean }) selected = false;

  render() {
    return html`
      <svg style="overflow: visible; position: absolute; pointer-events: none;">
        <path
          class="connection ${this.selected ? 'selected' : ''}"
          d=${this.path}
          pointer-events="stroke"
        />
      </svg>
    `;
  }
}

// Register with renderer
render.addPreset(Presets.classic.setup({
  customize: {
    connection() {
      return ({ path, ...props }) => html`
        <custom-connection
          .path=${path}
          .selected=${props.data.selected}
        ></custom-connection>
      `;
    }
  }
}));
```

## Graph Structure Utilities

```typescript
import { structures } from 'rete-structures';

const graph = structures(editor);

// Find entry points (no inputs)
const entryNodes = graph.roots().nodes();
console.log('Entry nodes:', entryNodes.map(n => n.label));

// Find exit points (no outputs)
const exitNodes = graph.leaves().nodes();
console.log('Exit nodes:', exitNodes.map(n => n.label));

// Get node dependencies
function getDependencies(nodeId: string) {
  return graph.predecessors(nodeId).nodes();
}

// Get dependent nodes
function getDependents(nodeId: string) {
  return graph.successors(nodeId).nodes();
}

// Filter to specific type
const mathNodes = graph.filter(
  node => node instanceof AddNode || node instanceof MultiplyNode
).nodes();

// Check if graph is a DAG (no cycles)
function isDAG(): boolean {
  const visited = new Set<string>();
  const recursionStack = new Set<string>();

  function hasCycle(nodeId: string): boolean {
    if (recursionStack.has(nodeId)) return true;
    if (visited.has(nodeId)) return false;

    visited.add(nodeId);
    recursionStack.add(nodeId);

    for (const successor of graph.outgoers(nodeId).nodes()) {
      if (hasCycle(successor.id)) return true;
    }

    recursionStack.delete(nodeId);
    return false;
  }

  for (const node of editor.getNodes()) {
    if (hasCycle(node.id)) return false;
  }
  return true;
}
```
