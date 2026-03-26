# Technologies & Dependencies

CCO is built on a deliberately minimal technology stack: Node.js 20+ with pure ESM, a single production dependency, and zero build tooling. This section catalogs every runtime requirement, dependency, and infrastructure component.

## Runtime Requirements

- **Node.js >= 20**: declared in `package.json` as `"engines": { "node": ">=20" }`. Required for native ES module support and modern `fs/promises` APIs.
- **No OS restriction**: works on macOS, Linux, and Windows. Platform-specific paths are handled in `scanner.mjs` (lines 17-20): macOS uses `/Library/Application Support/ClaudeCode/` for managed configuration, Linux uses `/etc/claude-code/`.

## Production Dependencies

CCO has exactly one production dependency:

- **`@modelcontextprotocol/sdk` ^1.27.1**: the MCP server framework. Provides `McpServer` for tool registration, `StdioServerTransport` for stdio communication, and re-exports `zod` for input schema validation. Used only in `src/mcp-server.mjs` -- the dashboard mode does not import it at startup.

## Optional Dependency

- **`ai-tokenizer` ^1.0.6**: a Claude-specific tokenizer for accurate token counting. Declared in `"optionalDependencies"` in `package.json`, so `npm install` will not fail if it cannot be installed. When available, the context budget calculator uses it for measured token counts (confidence: `"measured"`). When absent, `src/tokenizer.mjs` falls back to `Math.ceil(Buffer.byteLength(text, "utf-8") / 4)` (confidence: `"estimated"`), which is roughly 75-85% accurate.

## Dev Dependencies

- **`@playwright/test` ^1.58.2**: end-to-end browser testing framework. Tests launch the dashboard, interact with the UI, and verify that scanning, moving, and deleting work through the full stack.

## CDN-Loaded Client Libraries

The dashboard frontend loads two libraries from CDN, avoiding any need for `node_modules` on the client side:

- **SortableJS 1.15.6**: `https://cdn.jsdelivr.net/npm/sortablejs@1.15.6/Sortable.min.js` -- provides drag-and-drop functionality for moving items between scope zones.
- **Google Fonts**: Lato (weights 400, 700, 900) for UI text and Geist Mono (weights 400, 500) for code and monospace elements.

## HTTP Server

The dashboard runs on Node's built-in `http.createServer` -- no Express, Koa, or other HTTP framework. Key details:

- **Default port**: 3847
- **Auto-retry**: if the default port is in use, the server tries up to 10 consecutive ports (server.mjs lines 650-687)
- **Auto-open**: on macOS (`open`) and Linux (`xdg-open`), the browser opens automatically on startup (cli.mjs lines 108-113)
- **Non-blocking update check**: on startup, queries the npm registry to check for newer versions without blocking the server from accepting requests

## Zero-Build Architecture

The project uses no transpilation, no bundler, and no build step:

- `package.json` declares `"type": "module"` for native ESM
- All source files use `import`/`export` syntax directly
- `src/ui/` files (HTML, CSS, JS) are served as static files by the Node server
- CSS uses modern features (`oklch` color space, CSS custom properties) with separate light/dark theme variable sets
- No TypeScript, no JSX, no preprocessors

## CI/CD Pipeline

The GitHub Actions workflow `.github/workflows/publish.yml` triggers on push of `v*` tags and runs a single job:

1. **Checkout** via `actions/checkout@v5`
2. **Setup Node.js** with LTS version and npm registry configuration
3. **Install dependencies** with `npm ci` or `npm install --ignore-scripts`
4. **Build** runs `npm run build` if the script exists (it does not, so this is a no-op)
5. **Dedup check**: queries the npm registry to see if the version is already published, skipping the publish if so
6. **Publish to npm**: `npm publish --access public` using an `NPM_TOKEN` secret, with provenance enabled (`"publishConfig": { "provenance": true }` in `package.json`)
7. **Create GitHub Release**: via `softprops/action-gh-release@v2` with auto-generated release notes
8. **Install mcp-publisher**: downloads the MCP Registry publisher binary
9. **OIDC authentication**: authenticates to the MCP Registry via GitHub Actions OIDC
10. **Publish to MCP Registry**: registers or updates the MCP server entry (continues on error to avoid blocking the release)

### Required Permissions

The workflow requires two GitHub Actions permissions:
- `id-token: write` -- for MCP Registry OIDC authentication
- `contents: write` -- for creating GitHub Releases

## Why This Design Matters

The single production dependency and zero-build architecture mean CCO installs in seconds via `npx` with no compilation delays or platform-specific build failures. The optional tokenizer dependency degrades gracefully -- users who do not install it still get useful (if approximate) context budget calculations. CDN-loaded client libraries keep the npm package small while providing production-quality UI components. And the CI/CD pipeline publishes to three distribution channels (npm, GitHub Releases, MCP Registry) in a single automated workflow, ensuring that every tagged release is immediately available everywhere users might look for it.
