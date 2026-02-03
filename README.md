# Rete.js Skill for Claude Code

A Claude Code skill for creating visual node-based editors using Rete.js v2. Build interactive workflow builders, visual programming interfaces, and data processing pipelines with custom nodes and E2E testing support.

## Features

- **Framework-agnostic**: Uses Lit (Web Components) by default - no React, Vue, or Angular required
- **Custom node types**: Create nodes with typed sockets, controls, and processing logic
- **Dual processing engines**: Dataflow (data-driven) and Control Flow (execution-driven)
- **E2E testing ready**: Playwright test patterns with data-testid attributes
- **Serialization**: Import/export graphs as JSON for persistence
- **Full plugin ecosystem**: Context menus, undo/redo, minimap, auto-arrange, and more

## Installation

### 1. Clone or download this repository

```bash
git clone https://github.com/nealepetrillo/claude-skills-retejs.git
```

### 2. Add to your Claude Code skills

Copy the `.claude/skills/retejs` folder to your project's `.claude/skills/` directory:

```bash
# From your project root
mkdir -p .claude/skills
cp -r /path/to/claude-skills-retejs/.claude/skills/retejs .claude/skills/
```

Or add it globally to `~/.claude/skills/` for access in all projects:

```bash
mkdir -p ~/.claude/skills
cp -r /path/to/claude-skills-retejs/.claude/skills/retejs ~/.claude/skills/
```

## Usage

Once installed, invoke the skill in Claude Code by asking for node editor creation:

```
Create a node editor for a simple calculator with number and math operation nodes
```

```
Build a visual workflow builder with start, condition, and action nodes
```

```
Make a data processing pipeline editor with input, transform, and output nodes
```

For explicit invocation, use:

```
/retejs Create a node editor for image processing with filter nodes
```

## What the Skill Provides

### Framework Choice
- **Lit (default)** - Lightweight Web Components, works everywhere
- **Vanilla JS** - Maximum control, manual DOM rendering
- No React, Vue, or Angular dependencies required

### Processing Engines
- **Dataflow** - Data flows through connections, computed on demand
- **Control Flow** - Explicit execution path through nodes
- **Hybrid** - Combine both for complex workflows

### E2E Testing Support
- Playwright test patterns
- Data-testid attributes for reliable selectors
- Page Object Model examples
- Visual regression testing

### Serialization
- JSON export/import for graph persistence
- Node factory pattern for reconstruction
- Position and control value preservation

## Example Output

```typescript
import { NodeEditor, ClassicPreset } from 'rete';
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

  const node = new NumberNode(42);
  await editor.addNode(node);
  await area.translate(node.id, { x: 100, y: 100 });

  AreaExtensions.zoomAt(area, editor.getNodes());
  return { editor, area };
}
```

## Skill Structure

```
.claude/skills/retejs/
├── SKILL.md                     # Main skill definition
└── references/
    ├── editor-examples.md       # Comprehensive code examples
    └── testing-guide.md         # E2E testing patterns
```

## Plugin Ecosystem

| Plugin | Purpose |
|--------|---------|
| rete-area-plugin | 2D canvas, zoom, pan, node positioning |
| rete-connection-plugin | Interactive connection creation |
| @retejs/lit-plugin | Lit-based rendering (Web Components) |
| rete-engine | Dataflow/Control flow processing |
| rete-context-menu-plugin | Right-click node creation menus |
| rete-history-plugin | Undo/redo functionality |
| rete-minimap-plugin | Overview navigation |
| rete-auto-arrange-plugin | Automatic layout (elkjs) |
| rete-readonly-plugin | Lock editor for viewing |
| rete-structures | Graph traversal utilities |

## Requirements

- Node.js 18+ (for npm packages)
- Modern browser with ES modules support
- Optional: Playwright for E2E testing

## Resources

- [Rete.js Documentation](https://retejs.org/docs)
- [Rete.js Examples](https://retejs.org/examples)
- [GitHub Organization](https://github.com/retejs)
- [Lit Documentation](https://lit.dev/)
- [Playwright Documentation](https://playwright.dev/)

## License

MIT License - see [LICENSE](LICENSE) for details.
