# WebMCP Model Card Generator

Generate standardized documentation for WebMCP browser-side tools.

Extends [MCP Model Card Specification v1.0](https://github.com/Starborn/MCP-Model-Card-Generator) for the browser ecosystem.

## Use It Now

**Open `index.html` in any browser.** That's it. No installation, no build step, no dependencies to manage.

If hosted on GitHub Pages: [Open the Generator](https://starborn.github.io/WebMCP-Model-Card-Generator/)

You can also paste the `.jsx` file into a Claude artifact to run it inside Claude.

## What It Does

You fill in a 12-tab form. The generator produces two files:

- **JSON** -- machine-readable model card for registries, agents, and validators
- **Markdown** -- human-readable documentation for GitHub, specs, and READMEs

Each tab has:
- **Micro-guidance** right above every field telling you exactly what to type
- **Tutorial sidebar** (toggle on/off) explaining MCP vs WebMCP differences
- **Smart defaults** so you can generate a valid card without filling everything

## What is a WebMCP Model Card?

A model card is like a nutrition label for a tool. It tells AI agents and developers what your tool does, what it needs, what can go wrong, and how to use it safely.

[WebMCP](https://webmachinelearning.github.io/webmcp/) is a W3C draft specification (Feb 2026) that lets web pages expose JavaScript functions as structured tools for AI agents. Instead of agents scraping your page, you register tools they can call directly.

This generator documents those tools in a standardized format, extending the MCP Model Card Specification v1.0 that covers backend MCP servers.

## The 12 Sections

1. **Identity & Provenance** -- name, version, author, attribution (human/AI co-created/AI-generated)
2. **API Mode** -- declarative (HTML forms) vs imperative (JavaScript) vs both
3. **Tool Documentation** -- each tool's name, description, input schema, error states
4. **Session & Auth** -- authenticated vs anonymous, browser session inheritance
5. **Browser Compatibility** -- Chrome version, MCP-B polyfill, graceful degradation
6. **Security** -- same-origin scope, CSP, exfiltration risk, lethal trifecta assessment
7. **Interaction Patterns** -- statefulness, tool dependencies, page state modification
8. **Context Requirements** -- provideContext() data, DOM/framework state access
9. **Limitations** -- what the tool cannot do, edge cases, failure modes
10. **Discovery** -- registry listing, .well-known/webmcp, pre-visit discoverability
11. **Testing** -- methodology, which agents tested, pass/fail criteria
12. **Backend MCP Relationship** -- does a corresponding backend server exist?

## How This Was Built

### Architecture

The generator is a single-page React application compiled at runtime in the browser. No build tools, no npm, no webpack, no server.

### Technical Stack

- **React 18** -- loaded from CDN (cdnjs.cloudflare.com), provides the component model and state management
- **Babel Standalone** -- loaded from CDN, compiles JSX (React's HTML-like syntax) to JavaScript directly in the browser at page load
- **Google Fonts** -- DM Sans (body text), DM Mono (code/schemas), Outfit (headings)
- **No other dependencies** -- no CSS framework, no UI library, no build pipeline

### How It Works

1. Browser loads `index.html`
2. React and Babel load from CDN (two small JavaScript files, cached after first visit)
3. Babel compiles the inline JSX code to plain JavaScript
4. React renders the 12-tab form interface
5. All state lives in React's `useState` hooks -- nothing is sent to any server
6. When you click "Generate", two pure functions transform the form state into JSON and Markdown strings
7. You copy the output. That's it.

### Why This Approach?

- **Zero installation** -- open the file, it works
- **No server needed** -- everything runs client-side
- **GitHub Pages ready** -- push one file, enable Pages, done
- **No data leaves your browser** -- the generator is entirely local
- **Portable** -- download the HTML file, use it offline

### File Structure

```
index.html                              -- The complete generator (one file, ~960 lines)
WebMCP_Model_Card_Generator_USER_GUIDE.md  -- Field-by-field reference guide
webmcp-model-card-generator-v2.jsx      -- React source (for running inside Claude artifacts)
README.md                               -- This file
```

### Development Process

This generator was built in a single session through human-AI co-creation:

1. Analyzed the [W3C WebMCP specification](https://webmachinelearning.github.io/webmcp/) (Draft, 12 Feb 2026)
2. Catalogued existing WebMCP implementations (Google travel demo, MCP-B examples, Chrome DevTools quickstart, Jason McGhee's original)
3. Mapped the existing [MCP Model Card Specification v1.0](https://github.com/Starborn/MCP-Model-Card-Generator) schema to browser-side concepts
4. Identified 12 documentation sections covering identity, API mode, tools, session, browser compatibility, security, interaction patterns, context requirements, limitations, discovery, testing, and backend relationship
5. Built the React form interface with contextual tutorial sidebar
6. Added micro-guidance to every field based on usability testing
7. Packaged as standalone HTML for GitHub Pages deployment

## MCP vs WebMCP -- Quick Reference

| Aspect | Anthropic MCP (Backend) | W3C WebMCP (Browser) |
|--------|------------------------|---------------------|
| Runs on | Server (Node.js, Python, Docker) | Web page (browser JavaScript) |
| Transport | stdio, HTTP+SSE, JSON-RPC | postMessage, navigator.modelContext |
| Auth | API keys, env vars, OAuth config | Inherits browser session automatically |
| Discovery | MCP Registry, npm, .well-known | Page must be visited first (gap) |
| Tool registration | Server-side code only | HTML forms OR JavaScript (or both) |
| Human oversight | Separate approval flow | Human sees the page -- IS the oversight |
| Security boundary | API key management | Same-origin policy + CSP + human-in-loop |

## Related Projects

- [MCP Model Card Generator](https://github.com/Starborn/MCP-Model-Card-Generator) -- the original generator for backend MCP servers
- [MCP Model Card Specification v1.0](https://github.com/Starborn/MCP-Model-Card-Generator/blob/main/MCP_Model_Card_Specification_v1_0.md) -- the specification this extends
- [W3C WebMCP Spec](https://webmachinelearning.github.io/webmcp/) -- the W3C draft specification
- [WebMCP GitHub](https://github.com/webmachinelearning/webmcp) -- the official W3C repository

## Credits

**Co-created by:** Paola (concept, design) and Claude (Anthropic)  

**W3C Groups:**
- [AI Knowledge Representation Community Group](https://www.w3.org/community/aikr/)
- [Web Machine Learning Community Group](https://www.w3.org/community/webmachinelearning/)

**License:** MIT

Released February 2026
