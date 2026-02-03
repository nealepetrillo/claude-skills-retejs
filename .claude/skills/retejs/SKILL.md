---
name: retejs
description: Create visual node-based editors and workflow builders using Rete.js v2. Build interactive node graphs with custom nodes, dataflow/control-flow processing, and E2E testing support. Generates framework-agnostic code using Lit or vanilla JavaScript without React, Vue, or Angular dependencies.
---

# Rete.js v2 Visual Editor Skill

Create professional node-based editors and visual programming interfaces using Rete.js v2. Supports custom nodes, connections, dataflow/control-flow processing, and comprehensive E2E testing.

**Version**: Rete.js 2.x | Documentation: https://retejs.org/docs

## Core Concepts

### Framework-Agnostic Architecture
Rete.js is NOT bound to any UI framework. For lightweight implementations without React/Vue/Angular:
- Use **Lit** (`@retejs/lit-plugin`) - Web Components based, minimal overhead
- Use **vanilla JS** with manual DOM rendering for maximum control

### Plugin System
Rete.js uses scope-based plugin architecture where signals propagate from parent to child:

```typescript
import { Scope } from 'rete';

const parentScope = new Scope<number>('parent');
const childScope = new Scope<string, [number]>('child');

parentScope.addPipe((context) => {
  console.log('parent', context);
  return context;
});

parentScope.use(childScope);
await parentScope.emit(1);
```

## Installation

### NPM (Recommended for Lit)
```bash
npm i rete rete-area-plugin rete-connection-plugin @retejs/lit-plugin rete-render-utils
```

### CDN (Vanilla JS)
```html
<script src="https://cdn.jsdelivr.net/npm/rete/rete.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/rete-area-plugin/rete-area-plugin.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/rete-connection-plugin/rete-connection-plugin.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/rete-render-utils/rete-render-utils.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/@retejs/lit-plugin/retejs-lit-plugin.min.js"></script>
```

CDN namespaces: `Rete`, `ReteAreaPlugin`, `ReteConnectionPlugin`, `ReteLitPlugin`

## Core Pattern (Lit-based)

```typescript
import { NodeEditor, GetSchemes, ClassicPreset } from "rete";
import { AreaPlugin, AreaExtensions } from "rete-area-plugin";
import { ConnectionPlugin, Presets as ConnectionPresets } from "rete-connection-plugin";
import { LitPlugin, Presets } from "@retejs/lit-plugin";

// Type definitions
type Schemes = GetSchemes<
  ClassicPreset.Node,
  ClassicPreset.Connection<ClassicPreset.Node, ClassicPreset.Node>
>;
type AreaExtra = LitArea2D<Schemes>;

// Initialize editor
const editor = new NodeEditor<Schemes>();
const area = new AreaPlugin<Schemes, AreaExtra>(container);
const connection = new ConnectionPlugin<Schemes, AreaExtra>();
const render = new LitPlugin<Schemes, AreaExtra>();

// Configure presets
render.addPreset(Presets.classic.setup());
connection.addPreset(ConnectionPresets.classic.setup());

// Connect plugins
editor.use(area);
area.use(connection);
area.use(render);

// Create socket
const socket = new ClassicPreset.Socket("socket");

// Create and add nodes
const nodeA = new ClassicPreset.Node("Node A");
nodeA.addOutput("out", new ClassicPreset.Output(socket, "Output"));
await editor.addNode(nodeA);

// Position nodes
await area.translate(nodeA.id, { x: 100, y: 100 });

// Fit viewport
AreaExtensions.zoomAt(area, editor.getNodes());
```

## Custom Node Classes

### Basic Custom Node
```typescript
class NumberNode extends ClassicPreset.Node {
  width = 180;
  height = 120;

  constructor(initial: number = 0) {
    super("Number");
    this.addControl("value", new ClassicPreset.InputControl("number", { initial }));
    this.addOutput("value", new ClassicPreset.Output(socket, "Value"));
  }

  // For dataflow engine
  data(): { value: number } {
    const control = this.controls["value"] as ClassicPreset.InputControl<"number">;
    return { value: control.value ?? 0 };
  }
}

class AddNode extends ClassicPreset.Node {
  width = 180;
  height = 195;

  constructor() {
    super("Add");
    this.addInput("left", new ClassicPreset.Input(socket, "Left"));
    this.addInput("right", new ClassicPreset.Input(socket, "Right"));
    this.addOutput("value", new ClassicPreset.Output(socket, "Result"));
  }

  data(inputs: { left?: number[]; right?: number[] }): { value: number } {
    const left = inputs.left?.[0] ?? 0;
    const right = inputs.right?.[0] ?? 0;
    return { value: left + right };
  }
}
```

### Custom Socket Types (for validation)
```typescript
class NumberSocket extends ClassicPreset.Socket {
  constructor() { super("Number"); }
  isCompatibleWith(socket: ClassicPreset.Socket) {
    return socket instanceof NumberSocket;
  }
}

class StringSocket extends ClassicPreset.Socket {
  constructor() { super("String"); }
  isCompatibleWith(socket: ClassicPreset.Socket) {
    return socket instanceof StringSocket;
  }
}
```

## Processing Engines

### Dataflow Engine (Data-Driven)
Processes graphs where data flows from outputs to inputs:

```typescript
import { DataflowEngine } from "rete-engine";

const engine = new DataflowEngine<Schemes>();
editor.use(engine);

// Fetch result from a node
const result = await engine.fetch(resultNode.id);
console.log(result); // { value: 42 }

// Clear cache when inputs change
engine.reset();
```

### Control Flow Engine (Execution-Driven)
Processes graphs where execution flows through explicit control connections:

```typescript
import { ControlFlowEngine } from "rete-engine";

class StartNode extends ClassicPreset.Node {
  constructor() {
    super("Start");
    this.addOutput("exec", new ClassicPreset.Output(execSocket, "Execute"));
  }

  execute(_: never, forward: (output: "exec") => void) {
    forward("exec"); // Trigger next node
  }
}

class LogNode extends ClassicPreset.Node {
  constructor(message: string) {
    super("Log");
    this.addInput("exec", new ClassicPreset.Input(execSocket, "Execute"));
    this.addOutput("exec", new ClassicPreset.Output(execSocket, "Execute"));
    this.addControl("message", new ClassicPreset.InputControl("text", { initial: message }));
  }

  execute(_: "exec", forward: (output: "exec") => void) {
    const msg = (this.controls["message"] as ClassicPreset.InputControl<"text">).value;
    console.log(msg);
    forward("exec");
  }
}

const engine = new ControlFlowEngine<Schemes>();
editor.use(engine);

// Start execution
engine.execute(startNode.id);
```

### Hybrid Engine (Combined)
For nodes that need both data and execution flow:

```typescript
const dataflow = new DataflowEngine<Schemes>(({ inputs, outputs }) => ({
  inputs: () => Object.keys(inputs).filter(name => name !== "exec"),
  outputs: () => Object.keys(outputs).filter(name => name !== "exec")
}));

const controlflow = new ControlFlowEngine<Schemes>(() => ({
  inputs: () => ["exec"],
  outputs: () => ["exec"]
}));

editor.use(dataflow);
editor.use(controlflow);
```

## Connection Validation

```typescript
import { getConnectionSockets } from "rete-render-utils";

function canCreateConnection(editor: NodeEditor<Schemes>, connection: Schemes["Connection"]) {
  const { source, target } = getConnectionSockets(editor, connection);
  return source && target && source.isCompatibleWith(target);
}

editor.addPipe((context) => {
  if (context.type === "connectioncreate") {
    if (!canCreateConnection(editor, context.data)) {
      console.warn("Incompatible sockets");
      return; // Block connection
    }
  }
  return context;
});
```

## Import/Export (Serialization)

### Export Pattern
```typescript
interface SerializedNode {
  id: string;
  type: string;
  position: { x: number; y: number };
  data: Record<string, unknown>;
}

interface SerializedConnection {
  id: string;
  source: string;
  sourceOutput: string;
  target: string;
  targetInput: string;
}

interface SerializedGraph {
  nodes: SerializedNode[];
  connections: SerializedConnection[];
}

async function exportGraph(editor: NodeEditor<Schemes>, area: AreaPlugin<Schemes, AreaExtra>): Promise<SerializedGraph> {
  const nodes: SerializedNode[] = [];

  for (const node of editor.getNodes()) {
    const view = area.nodeViews.get(node.id);
    nodes.push({
      id: node.id,
      type: node.label,
      position: view?.position ?? { x: 0, y: 0 },
      data: extractNodeData(node) // Custom function to extract control values
    });
  }

  const connections: SerializedConnection[] = editor.getConnections().map(conn => ({
    id: conn.id,
    source: conn.source,
    sourceOutput: conn.sourceOutput,
    target: conn.target,
    targetInput: conn.targetInput
  }));

  return { nodes, connections };
}
```

### Import Pattern
```typescript
async function importGraph(
  editor: NodeEditor<Schemes>,
  area: AreaPlugin<Schemes, AreaExtra>,
  data: SerializedGraph
) {
  await editor.clear();

  // Create node factory
  const nodeFactory: Record<string, (data: Record<string, unknown>) => ClassicPreset.Node> = {
    "Number": (d) => new NumberNode(d.value as number),
    "Add": () => new AddNode(),
    // Add more node types...
  };

  // Import nodes
  for (const nodeData of data.nodes) {
    const factory = nodeFactory[nodeData.type];
    if (!factory) continue;

    const node = factory(nodeData.data);
    node.id = nodeData.id; // Preserve original ID
    await editor.addNode(node);
    await area.translate(node.id, nodeData.position);
  }

  // Import connections
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
}
```

## Essential Plugins

### Auto Arrange
```bash
npm i rete-auto-arrange-plugin elkjs web-worker
```

```typescript
import { AutoArrangePlugin, Presets as ArrangePresets } from "rete-auto-arrange-plugin";

const arrange = new AutoArrangePlugin<Schemes>();
arrange.addPreset(ArrangePresets.classic.setup());
area.use(arrange);

await arrange.layout(); // Auto-arrange all nodes
```

### Context Menu
```bash
npm i rete-context-menu-plugin
```

```typescript
import { ContextMenuPlugin, Presets as ContextMenuPresets } from "rete-context-menu-plugin";

const contextMenu = new ContextMenuPlugin<Schemes>({
  items: ContextMenuPresets.classic.setup([
    ["Number", () => new NumberNode()],
    ["Add", () => new AddNode()],
    ["Math", [
      ["Multiply", () => new MultiplyNode()],
      ["Divide", () => new DivideNode()]
    ]]
  ])
});

area.use(contextMenu);
render.addPreset(Presets.contextMenu.setup());
```

### History (Undo/Redo)
```bash
npm i rete-history-plugin
```

```typescript
import { HistoryPlugin, HistoryActions, HistoryExtensions, Presets as HistoryPresets } from "rete-history-plugin";

const history = new HistoryPlugin<Schemes, HistoryActions<Schemes>>();
history.addPreset(HistoryPresets.classic.setup());
area.use(history);

// Enable keyboard shortcuts (Ctrl+Z, Ctrl+Y)
HistoryExtensions.keyboard(history);

// Programmatic undo/redo
await history.undo();
await history.redo();
```

### Minimap
```bash
npm i rete-minimap-plugin
```

```typescript
import { MinimapPlugin } from "rete-minimap-plugin";

const minimap = new MinimapPlugin<Schemes>();
area.use(minimap);
render.addPreset(Presets.minimap.setup({ size: 200 }));
```

### Readonly Mode
```bash
npm i rete-readonly-plugin
```

```typescript
import { ReadonlyPlugin } from "rete-readonly-plugin";

const readonly = new ReadonlyPlugin<Schemes>();
editor.use(readonly.root);
editor.use(area);
area.use(readonly.area);

// Enable/disable readonly
readonly.enable();  // Block user modifications
readonly.disable(); // Allow modifications
```

## Selection System

### Selectable Nodes
```typescript
const selector = AreaExtensions.selector();
const accumulating = AreaExtensions.accumulateOnCtrl();

AreaExtensions.selectableNodes(area, selector, { accumulating });
AreaExtensions.simpleNodesOrder(area); // Bring selected to front

// Programmatic selection
selectableNodes.select(nodeId);        // Select single
selectableNodes.select(nodeId, true);  // Add to selection
selectableNodes.unselect(nodeId);      // Remove from selection
```

## E2E Testing with Playwright

See `references/testing-guide.md` for comprehensive testing patterns.

### Basic Test Structure
```typescript
import { test, expect } from '@playwright/test';

test.describe('Node Editor', () => {
  test.beforeEach(async ({ page }) => {
    await page.goto('/editor');
    await page.waitForSelector('.rete-background'); // Wait for editor
  });

  test('should create node via context menu', async ({ page }) => {
    // Right-click to open context menu
    await page.click('.rete-background', { button: 'right' });
    await page.click('text=Number');

    // Verify node created
    const nodes = await page.locator('[data-testid="node"]').count();
    expect(nodes).toBe(1);
  });

  test('should connect two nodes', async ({ page }) => {
    // Create nodes and connect them
    const outputSocket = page.locator('[data-testid="output-socket"]').first();
    const inputSocket = page.locator('[data-testid="input-socket"]').first();

    await outputSocket.dragTo(inputSocket);

    // Verify connection
    const connections = await page.locator('[data-testid="connection"]').count();
    expect(connections).toBe(1);
  });
});
```

### Adding Test IDs to Components
```typescript
// Custom node component for testing
render.addPreset(Presets.classic.setup({
  customize: {
    node() {
      return (props) => html`
        <div data-testid="node" data-node-id="${props.data.id}">
          ${Presets.classic.Node(props)}
        </div>
      `;
    },
    socket(context) {
      const side = context.payload.side;
      return (props) => html`
        <div data-testid="${side}-socket" data-socket-key="${context.payload.key}">
          ${Presets.classic.Socket(props)}
        </div>
      `;
    }
  }
}));
```

## Graph Operations (rete-structures)

```bash
npm i rete-structures
```

```typescript
import { structures } from 'rete-structures';

const graph = structures(editor);

// Traversal
graph.roots();                 // Nodes without incoming connections
graph.leaves();                // Nodes without outgoing connections
graph.incomers(nodeId);        // Direct predecessors
graph.outgoers(nodeId);        // Direct successors
graph.predecessors(nodeId);    // All upstream nodes
graph.successors(nodeId);      // All downstream nodes

// Set operations
graph.filter(nodePredicate, connectionPredicate);
graph.union(otherGraph);
graph.difference(otherGraph);
graph.intersection(otherGraph);
```

## Best Practices

### Node Design
1. **Fixed dimensions** - Define `width` and `height` on custom nodes for minimap/arrange
2. **Descriptive labels** - Use clear node names that explain purpose
3. **Typed sockets** - Use custom socket classes for type safety
4. **Control validation** - Validate control input values

### Performance
1. **Limit visible nodes** - Use virtualization for large graphs
2. **Debounce processing** - Don't recompute on every change
3. **Cache engine results** - Call `engine.reset()` only when necessary

### Testing
1. **Add data-testid attributes** - Enable reliable E2E selectors
2. **Test serialization** - Verify import/export roundtrips
3. **Test edge cases** - Cycles, disconnected nodes, invalid connections

## Resources

- **Documentation**: https://retejs.org/docs
- **Examples**: https://retejs.org/examples
- **GitHub**: https://github.com/retejs
- **Testing Guide**: `references/testing-guide.md`
- **Code Examples**: `references/editor-examples.md`
