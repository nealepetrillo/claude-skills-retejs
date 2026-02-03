# Rete.js E2E Testing Guide

Comprehensive guide for end-to-end testing of Rete.js editors using Playwright.

## Overview

Rete.js recommends E2E testing over unit tests for validating editor functionality from a user perspective. The `rete-qa` package provides regression testing across multiple frameworks and browsers.

## Test Environment Setup

### Playwright Configuration

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './tests/e2e',
  fullyParallel: true,
  forbidOnly: !!process.env.CI,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: 'html',
  use: {
    baseURL: 'http://localhost:3000',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },
  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    { name: 'webkit', use: { ...devices['Desktop Safari'] } },
  ],
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Package.json Scripts

```json
{
  "scripts": {
    "test:e2e": "playwright test",
    "test:e2e:ui": "playwright test --ui",
    "test:e2e:debug": "playwright test --debug",
    "test:e2e:headed": "playwright test --headed"
  },
  "devDependencies": {
    "@playwright/test": "^1.40.0"
  }
}
```

## Adding Test Attributes

### Node Components with Test IDs

```typescript
// Add data-testid attributes for reliable selection
render.addPreset(Presets.classic.setup({
  customize: {
    node(context) {
      return (props) => html`
        <div
          data-testid="node"
          data-node-id="${props.data.id}"
          data-node-type="${props.data.label}"
        >
          ${Presets.classic.Node({ ...props })}
        </div>
      `;
    },
    socket(context) {
      const { side, key } = context.payload;
      return (props) => html`
        <div
          data-testid="${side}-socket"
          data-socket-key="${key}"
          data-node-id="${context.payload.nodeId}"
        >
          ${Presets.classic.Socket({ ...props })}
        </div>
      `;
    },
    connection() {
      return (props) => html`
        <g
          data-testid="connection"
          data-connection-id="${props.data.id}"
          data-source="${props.data.source}"
          data-target="${props.data.target}"
        >
          ${Presets.classic.Connection({ ...props })}
        </g>
      `;
    },
    control(context) {
      if (context.payload instanceof ClassicPreset.InputControl) {
        return (props) => html`
          <div
            data-testid="control"
            data-control-key="${context.payload.key}"
          >
            ${Presets.classic.Control({ ...props })}
          </div>
        `;
      }
    }
  }
}));
```

### Editor Container Setup

```typescript
// Add identifiable container
function createEditor(container: HTMLElement) {
  container.setAttribute('data-testid', 'rete-editor');

  const area = new AreaPlugin<Schemes, AreaExtra>(container);

  // The area plugin adds .rete-background class
  // Use this for clicking on empty space
}
```

## Test Utilities

### Page Object Model

```typescript
// tests/e2e/fixtures/editor-page.ts
import { Page, Locator } from '@playwright/test';

export class EditorPage {
  readonly page: Page;
  readonly editor: Locator;
  readonly background: Locator;

  constructor(page: Page) {
    this.page = page;
    this.editor = page.locator('[data-testid="rete-editor"]');
    this.background = page.locator('.rete-background');
  }

  async goto() {
    await this.page.goto('/');
    await this.editor.waitFor({ state: 'visible' });
  }

  // Node operations
  async getNodes() {
    return this.page.locator('[data-testid="node"]');
  }

  async getNodeById(id: string) {
    return this.page.locator(`[data-node-id="${id}"]`);
  }

  async getNodeByType(type: string) {
    return this.page.locator(`[data-node-type="${type}"]`);
  }

  async getNodeCount() {
    return (await this.getNodes()).count();
  }

  // Connection operations
  async getConnections() {
    return this.page.locator('[data-testid="connection"]');
  }

  async getConnectionCount() {
    return (await this.getConnections()).count();
  }

  // Socket operations
  async getOutputSocket(nodeId: string, key: string) {
    return this.page.locator(
      `[data-node-id="${nodeId}"] [data-testid="output-socket"][data-socket-key="${key}"]`
    );
  }

  async getInputSocket(nodeId: string, key: string) {
    return this.page.locator(
      `[data-node-id="${nodeId}"] [data-testid="input-socket"][data-socket-key="${key}"]`
    );
  }

  // Context menu
  async openContextMenu(x?: number, y?: number) {
    const target = x !== undefined && y !== undefined
      ? { x, y }
      : this.background;

    await (typeof target === 'object' && 'x' in target
      ? this.page.mouse.click(target.x, target.y, { button: 'right' })
      : target.click({ button: 'right' }));

    await this.page.locator('.context-menu').waitFor({ state: 'visible' });
  }

  async clickContextMenuItem(label: string) {
    await this.page.locator(`.context-menu >> text=${label}`).click();
  }

  async createNodeViaContextMenu(nodeType: string, x = 200, y = 200) {
    await this.openContextMenu(x, y);
    await this.clickContextMenuItem(nodeType);
  }

  // Control operations
  async getControl(nodeId: string, key: string) {
    return this.page.locator(
      `[data-node-id="${nodeId}"] [data-control-key="${key}"]`
    );
  }

  async setControlValue(nodeId: string, key: string, value: string | number) {
    const control = await this.getControl(nodeId, key);
    const input = control.locator('input');
    await input.fill(String(value));
    await input.press('Enter');
  }

  // Connection creation
  async createConnection(
    sourceNodeId: string,
    sourceKey: string,
    targetNodeId: string,
    targetKey: string
  ) {
    const outputSocket = await this.getOutputSocket(sourceNodeId, sourceKey);
    const inputSocket = await this.getInputSocket(targetNodeId, targetKey);

    await outputSocket.dragTo(inputSocket);
  }

  // Selection
  async selectNode(nodeId: string) {
    const node = await this.getNodeById(nodeId);
    await node.click();
  }

  async selectMultipleNodes(nodeIds: string[]) {
    await this.page.keyboard.down('Control');
    for (const id of nodeIds) {
      await this.selectNode(id);
    }
    await this.page.keyboard.up('Control');
  }

  // Keyboard shortcuts
  async undo() {
    await this.page.keyboard.press('Control+z');
  }

  async redo() {
    await this.page.keyboard.press('Control+y');
  }

  async deleteSelected() {
    await this.page.keyboard.press('Delete');
  }

  // Viewport
  async zoomIn() {
    await this.editor.hover();
    await this.page.mouse.wheel(0, -100);
  }

  async zoomOut() {
    await this.editor.hover();
    await this.page.mouse.wheel(0, 100);
  }

  async pan(deltaX: number, deltaY: number) {
    await this.editor.hover();
    await this.page.mouse.down({ button: 'middle' });
    await this.page.mouse.move(deltaX, deltaY);
    await this.page.mouse.up({ button: 'middle' });
  }
}
```

### Custom Fixtures

```typescript
// tests/e2e/fixtures/index.ts
import { test as base } from '@playwright/test';
import { EditorPage } from './editor-page';

type Fixtures = {
  editorPage: EditorPage;
};

export const test = base.extend<Fixtures>({
  editorPage: async ({ page }, use) => {
    const editorPage = new EditorPage(page);
    await editorPage.goto();
    await use(editorPage);
  },
});

export { expect } from '@playwright/test';
```

## Test Suites

### Basic Editor Tests

```typescript
// tests/e2e/editor.spec.ts
import { test, expect } from './fixtures';

test.describe('Editor Initialization', () => {
  test('should render empty editor', async ({ editorPage }) => {
    const nodeCount = await editorPage.getNodeCount();
    expect(nodeCount).toBe(0);
  });

  test('should have zoom/pan capabilities', async ({ editorPage, page }) => {
    const editor = editorPage.editor;

    // Check editor is interactive
    await expect(editor).toBeVisible();

    // Zoom should work
    await editorPage.zoomIn();
    // Verify transform changed (implementation specific)
  });
});
```

### Node Creation Tests

```typescript
test.describe('Node Creation', () => {
  test('should create node via context menu', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 200, 200);

    const nodeCount = await editorPage.getNodeCount();
    expect(nodeCount).toBe(1);

    const node = await editorPage.getNodeByType('Number');
    await expect(node).toBeVisible();
  });

  test('should create multiple nodes', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 100, 100);
    await editorPage.createNodeViaContextMenu('Number', 100, 300);
    await editorPage.createNodeViaContextMenu('Add', 350, 200);

    const nodeCount = await editorPage.getNodeCount();
    expect(nodeCount).toBe(3);
  });

  test('should position node at context menu location', async ({ editorPage, page }) => {
    const x = 300, y = 250;
    await editorPage.createNodeViaContextMenu('Number', x, y);

    const node = await editorPage.getNodeByType('Number');
    const box = await node.boundingBox();

    // Node should be near click position (with some tolerance)
    expect(box?.x).toBeGreaterThan(x - 50);
    expect(box?.x).toBeLessThan(x + 50);
    expect(box?.y).toBeGreaterThan(y - 50);
    expect(box?.y).toBeLessThan(y + 50);
  });
});
```

### Connection Tests

```typescript
test.describe('Connections', () => {
  test.beforeEach(async ({ editorPage }) => {
    // Create two nodes for connection tests
    await editorPage.createNodeViaContextMenu('Number', 100, 200);
    await editorPage.createNodeViaContextMenu('Display', 400, 200);
  });

  test('should create connection by dragging', async ({ editorPage, page }) => {
    const nodes = await editorPage.getNodes();
    const numberNode = nodes.first();
    const displayNode = nodes.last();

    const numberNodeId = await numberNode.getAttribute('data-node-id');
    const displayNodeId = await displayNode.getAttribute('data-node-id');

    await editorPage.createConnection(
      numberNodeId!,
      'value',
      displayNodeId!,
      'value'
    );

    const connectionCount = await editorPage.getConnectionCount();
    expect(connectionCount).toBe(1);
  });

  test('should prevent invalid connections', async ({ editorPage }) => {
    // Try to connect two outputs (should fail)
    // Implementation depends on socket types
    const initialCount = await editorPage.getConnectionCount();

    // Attempt invalid connection...

    const finalCount = await editorPage.getConnectionCount();
    expect(finalCount).toBe(initialCount);
  });

  test('should delete connection on click', async ({ editorPage, page }) => {
    // Setup: create valid connection first
    const nodes = await editorPage.getNodes();
    const numberNodeId = await nodes.first().getAttribute('data-node-id');
    const displayNodeId = await nodes.last().getAttribute('data-node-id');

    await editorPage.createConnection(
      numberNodeId!,
      'value',
      displayNodeId!,
      'value'
    );

    expect(await editorPage.getConnectionCount()).toBe(1);

    // Click on connection to select, then delete
    const connection = (await editorPage.getConnections()).first();
    await connection.click();
    await editorPage.deleteSelected();

    expect(await editorPage.getConnectionCount()).toBe(0);
  });
});
```

### Control Tests

```typescript
test.describe('Controls', () => {
  test('should update number control value', async ({ editorPage, page }) => {
    await editorPage.createNodeViaContextMenu('Number', 200, 200);

    const nodes = await editorPage.getNodes();
    const nodeId = await nodes.first().getAttribute('data-node-id');

    // Set value
    await editorPage.setControlValue(nodeId!, 'value', 42);

    // Verify value persisted
    const control = await editorPage.getControl(nodeId!, 'value');
    const input = control.locator('input');
    await expect(input).toHaveValue('42');
  });

  test('should update text control value', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Print', 200, 200);

    const nodes = await editorPage.getNodes();
    const nodeId = await nodes.first().getAttribute('data-node-id');

    await editorPage.setControlValue(nodeId!, 'message', 'Hello World');

    const control = await editorPage.getControl(nodeId!, 'message');
    const input = control.locator('input');
    await expect(input).toHaveValue('Hello World');
  });
});
```

### Selection Tests

```typescript
test.describe('Selection', () => {
  test.beforeEach(async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 100, 100);
    await editorPage.createNodeViaContextMenu('Number', 100, 300);
    await editorPage.createNodeViaContextMenu('Add', 350, 200);
  });

  test('should select single node on click', async ({ editorPage }) => {
    const nodes = await editorPage.getNodes();
    const firstNodeId = await nodes.first().getAttribute('data-node-id');

    await editorPage.selectNode(firstNodeId!);

    // Check selection state (implementation specific)
    const firstNode = await editorPage.getNodeById(firstNodeId!);
    await expect(firstNode).toHaveClass(/selected/);
  });

  test('should select multiple nodes with Ctrl+Click', async ({ editorPage }) => {
    const nodes = await editorPage.getNodes();
    const nodeIds = await Promise.all(
      (await nodes.all()).map(n => n.getAttribute('data-node-id'))
    );

    await editorPage.selectMultipleNodes(nodeIds.filter(Boolean) as string[]);

    // All nodes should be selected
    for (const id of nodeIds) {
      const node = await editorPage.getNodeById(id!);
      await expect(node).toHaveClass(/selected/);
    }
  });

  test('should delete selected nodes', async ({ editorPage }) => {
    const nodes = await editorPage.getNodes();
    const firstNodeId = await nodes.first().getAttribute('data-node-id');

    await editorPage.selectNode(firstNodeId!);
    await editorPage.deleteSelected();

    expect(await editorPage.getNodeCount()).toBe(2);
  });
});
```

### Undo/Redo Tests

```typescript
test.describe('Undo/Redo', () => {
  test('should undo node creation', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 200, 200);
    expect(await editorPage.getNodeCount()).toBe(1);

    await editorPage.undo();
    expect(await editorPage.getNodeCount()).toBe(0);
  });

  test('should redo undone action', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 200, 200);
    await editorPage.undo();
    expect(await editorPage.getNodeCount()).toBe(0);

    await editorPage.redo();
    expect(await editorPage.getNodeCount()).toBe(1);
  });

  test('should undo connection creation', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 100, 200);
    await editorPage.createNodeViaContextMenu('Display', 400, 200);

    const nodes = await editorPage.getNodes();
    const numberNodeId = await nodes.first().getAttribute('data-node-id');
    const displayNodeId = await nodes.last().getAttribute('data-node-id');

    await editorPage.createConnection(
      numberNodeId!,
      'value',
      displayNodeId!,
      'value'
    );
    expect(await editorPage.getConnectionCount()).toBe(1);

    await editorPage.undo();
    expect(await editorPage.getConnectionCount()).toBe(0);
  });
});
```

### Serialization Tests

```typescript
test.describe('Import/Export', () => {
  test('should export and reimport graph', async ({ editorPage, page }) => {
    // Create a graph
    await editorPage.createNodeViaContextMenu('Number', 100, 100);
    await editorPage.createNodeViaContextMenu('Number', 100, 300);
    await editorPage.createNodeViaContextMenu('Add', 350, 200);

    const nodes = await editorPage.getNodes();
    const node1Id = await nodes.nth(0).getAttribute('data-node-id');
    const node2Id = await nodes.nth(1).getAttribute('data-node-id');
    const addNodeId = await nodes.nth(2).getAttribute('data-node-id');

    // Create connections
    await editorPage.createConnection(node1Id!, 'value', addNodeId!, 'a');
    await editorPage.createConnection(node2Id!, 'value', addNodeId!, 'b');

    // Set control values
    await editorPage.setControlValue(node1Id!, 'value', 5);
    await editorPage.setControlValue(node2Id!, 'value', 10);

    // Export (assumes export function exposed on window)
    const exportData = await page.evaluate(() => {
      return (window as any).exportGraph();
    });

    // Clear editor
    await page.evaluate(() => {
      return (window as any).clearEditor();
    });
    expect(await editorPage.getNodeCount()).toBe(0);

    // Reimport
    await page.evaluate((data) => {
      return (window as any).importGraph(data);
    }, exportData);

    // Verify restoration
    expect(await editorPage.getNodeCount()).toBe(3);
    expect(await editorPage.getConnectionCount()).toBe(2);

    // Verify control values preserved
    const reimportedNodes = await editorPage.getNodes();
    const firstNodeId = await reimportedNodes.first().getAttribute('data-node-id');
    const control = await editorPage.getControl(firstNodeId!, 'value');
    const input = control.locator('input');
    await expect(input).toHaveValue('5');
  });

  test('should handle invalid import data gracefully', async ({ editorPage, page }) => {
    // Attempt to import invalid data
    await page.evaluate(() => {
      return (window as any).importGraph({ invalid: 'data' });
    });

    // Editor should still be functional
    await editorPage.createNodeViaContextMenu('Number', 200, 200);
    expect(await editorPage.getNodeCount()).toBe(1);
  });
});
```

### Dataflow Processing Tests

```typescript
test.describe('Dataflow Processing', () => {
  test('should compute result through graph', async ({ editorPage, page }) => {
    // Create computation graph: 5 + 10 = 15
    await editorPage.createNodeViaContextMenu('Number', 100, 100);
    await editorPage.createNodeViaContextMenu('Number', 100, 300);
    await editorPage.createNodeViaContextMenu('Add', 350, 200);
    await editorPage.createNodeViaContextMenu('Display', 600, 200);

    const nodes = await editorPage.getNodes();
    const num1Id = await nodes.nth(0).getAttribute('data-node-id');
    const num2Id = await nodes.nth(1).getAttribute('data-node-id');
    const addId = await nodes.nth(2).getAttribute('data-node-id');
    const displayId = await nodes.nth(3).getAttribute('data-node-id');

    // Set values
    await editorPage.setControlValue(num1Id!, 'value', 5);
    await editorPage.setControlValue(num2Id!, 'value', 10);

    // Connect graph
    await editorPage.createConnection(num1Id!, 'value', addId!, 'a');
    await editorPage.createConnection(num2Id!, 'value', addId!, 'b');
    await editorPage.createConnection(addId!, 'sum', displayId!, 'value');

    // Trigger computation and check result
    await page.waitForTimeout(100); // Allow processing

    const displayControl = await editorPage.getControl(displayId!, 'display');
    const displayValue = displayControl.locator('input');
    await expect(displayValue).toHaveValue('15');
  });

  test('should update when input changes', async ({ editorPage, page }) => {
    // Setup same graph as above...
    // Then change input value
    await editorPage.setControlValue(num1Id!, 'value', 20);

    await page.waitForTimeout(100);

    const displayControl = await editorPage.getControl(displayId!, 'display');
    const displayValue = displayControl.locator('input');
    await expect(displayValue).toHaveValue('30'); // 20 + 10
  });
});
```

## Using rete-qa

For comprehensive regression testing across frameworks:

```bash
# Install globally
npm i -g rete-qa

# Initialize with custom dependencies (optional)
rete-qa init --deps-alias dependencies.json

# Run all tests
rete-qa test

# Run specific framework
rete-qa test --framework react

# Run specific browser
rete-qa test --browser chromium
```

### dependencies.json Example

```json
{
  "rete": "./path/to/local/rete.tgz",
  "rete-area-plugin": "^2.0.0",
  "rete-connection-plugin": "^2.0.0"
}
```

## Visual Regression Testing

```typescript
// tests/e2e/visual.spec.ts
import { test, expect } from './fixtures';

test.describe('Visual Regression', () => {
  test('empty editor matches snapshot', async ({ editorPage }) => {
    await expect(editorPage.editor).toHaveScreenshot('empty-editor.png');
  });

  test('editor with nodes matches snapshot', async ({ editorPage }) => {
    await editorPage.createNodeViaContextMenu('Number', 100, 100);
    await editorPage.createNodeViaContextMenu('Add', 350, 100);

    await expect(editorPage.editor).toHaveScreenshot('nodes-created.png');
  });

  test('connected graph matches snapshot', async ({ editorPage }) => {
    // Create and connect nodes...
    await expect(editorPage.editor).toHaveScreenshot('connected-graph.png');
  });
});
```

## Performance Testing

```typescript
test.describe('Performance', () => {
  test('should handle 100 nodes', async ({ editorPage, page }) => {
    const startTime = Date.now();

    // Bulk create nodes
    await page.evaluate(async () => {
      const { createNode } = window as any;
      for (let i = 0; i < 100; i++) {
        await createNode('Number', {
          x: (i % 10) * 200,
          y: Math.floor(i / 10) * 150
        });
      }
    });

    const duration = Date.now() - startTime;
    console.log(`Created 100 nodes in ${duration}ms`);

    expect(await editorPage.getNodeCount()).toBe(100);
    expect(duration).toBeLessThan(5000); // Should complete in under 5s
  });

  test('should maintain responsiveness with many connections', async ({ editorPage, page }) => {
    // Create nodes and many connections
    // Measure frame rate or interaction latency
  });
});
```

## CI/CD Integration

### GitHub Actions

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps

      - name: Run E2E tests
        run: npm run test:e2e

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30
```

## Best Practices

1. **Use Page Object Model** - Encapsulate element locators and actions
2. **Add data-testid attributes** - Prefer stable selectors over CSS classes
3. **Wait for state changes** - Use explicit waits instead of timeouts
4. **Test one thing per test** - Keep tests focused and independent
5. **Run tests in parallel** - Use Playwright's parallelization
6. **Use visual regression** - Catch unintended UI changes
7. **Test edge cases** - Empty graphs, cycles, disconnected nodes
8. **Mock external services** - Isolate tests from network dependencies
