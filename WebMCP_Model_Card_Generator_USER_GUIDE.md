# WebMCP Model Card Generator -- Guide for the Clueless

**A field-by-field walkthrough so you can fill out every tab without knowing anything about WebMCP beforehand.**

Co-created by Paola Di Maio, PhD (W3C AI-KR CG Chair) & Claude (Anthropic)
February 2026

---

## What You're Making

A **model card** for a browser-side tool. Think of it as a label for a product: it tells AI agents (and humans) what your tool does, what it needs, what can go wrong, and how to use it safely.

You fill in a form. The generator produces two files:
- **JSON** -- for machines to read (registries, agents, validators)
- **Markdown** -- for humans to read (documentation, GitHub, specifications)

**Time needed**: 15-30 minutes if you know your tool. 5 minutes if you just want to test the generator.

---

## Tab 1: Identity & Provenance

*Who made this, what is it, where does it live?*

### Tool/Page Name (required)

The name of your WebMCP-enabled tool or web page. This is what agents will see when they discover your tools.

- Good: "Easely Design Editor", "Flight Search Demo", "Todo Manager"
- Bad: "my-tool", "test", "page1"

### Version (required)

Use semantic versioning: **major.minor.patch**

- `1.0.0` = first stable release
- `0.1.0` = early prototype
- `2.3.1` = mature, updated

If you don't know, use `0.1.0` for prototypes or `1.0.0` for something you'd show people.

### Description (required)

One or two sentences explaining what tools this page exposes to AI agents.

- Good: "Travel booking page exposing flight search, hotel filtering, and itinerary building tools via WebMCP declarative and imperative APIs"
- Bad: "A tool" or "This is my page"

### Author

Your name, team name, or organization. Can be left blank.

### Creation Date

Auto-filled with today's date. Change it if the tool was created earlier.

### License (required)

Pick one:

| License | When to use |
|---------|-------------|
| **MIT** | Most permissive, most common. "Do whatever you want, just include the license." |
| **Apache 2.0** | Permissive + patent protection. Good for company projects. |
| **GPL 3.0** | Copyleft. Anyone who uses your code must also open-source theirs. |
| **Proprietary** | Closed source, commercial. |
| **W3C Community CLA** | If your tool is part of a W3C community group deliverable. |

If unsure, pick MIT.

### Page URL

The web address where your WebMCP tools are registered. This is where agents go to find and call your tools.

- Example: `https://travel-demo.bandarra.me/`
- Example: `https://mysite.com/booking`
- Put "tbd" if you haven't deployed yet.

### Attribution (required)

How was this tool created? Be honest -- this is a transparency field.

| Choice | Meaning |
|--------|---------|
| **Human-authored** | A person wrote all the code |
| **AI co-created** | Human and AI worked together (like us right now!) |
| **AI-generated** | AI generated the code, human reviewed/edited |

### Source Repository

Link to your GitHub/GitLab repo. Leave blank if closed source.

---

## Tab 2: API Mode

*How do agents talk to your tools?*

This is where WebMCP diverges most from backend MCP. You have TWO choices (or both).

### Primary API Mode (required)

| Mode | What it means | When to use |
|------|---------------|-------------|
| **Declarative (HTML forms)** | You add attributes to existing HTML `<form>` elements. No JavaScript needed. | You already have working HTML forms and want the fastest path to agent-readiness. |
| **Imperative (JavaScript)** | You call `navigator.modelContext.registerTool()` in JavaScript. | Complex logic, dynamic tools, multi-step workflows. |
| **Both** | You use forms for simple actions and JS for complex ones. | Large applications with a mix of simple and complex tools. |

### If Declarative: Form toolname

The exact `toolname` attribute you put on your HTML form:
```html
<form toolname="searchFlights" tooldescription="Search flights">
```
Enter: `searchFlights`

### If Declarative: Form tooldescription

The natural language description in the form attribute. This is what agents read to decide whether to use your tool.

Enter: `Search for available flights by origin, destination, and date`

### If Imperative: Registration Method

How you register tools in JavaScript:

| Method | When to use |
|--------|-------------|
| **navigator.modelContext.registerTool()** | W3C standard API. Use for single tool registration. |
| **navigator.modelContext.provideContext()** | W3C standard. Registers multiple tools at once (replaces any existing). |
| **MCP-B @mcp-b/global import** | Community polyfill. Use if you need cross-browser support before native implementation. |
| **WebMCP widget script tag** | Simplest option -- add a `<script>` tag. Good for quick prototyping. |

If unsure, pick `registerTool()` -- it's the W3C standard.

---

## Tab 3: Tool Documentation

*What can agents actually call?*

You'll add one entry for each tool your page exposes. Click **+ Add Tool** after filling in each one.

### Tool Name (required)

The exact name agents use to call this tool. Must match your code exactly.

- Good: `searchFlights`, `addToCart`, `toggleTheme`
- Bad: `tool1`, `myFunction`, `doStuff`

### Description (required)

What this tool does, in plain language. Agents read this to decide when and how to use it.

- Good: "Search for available flights between two airports on a given date. Returns a list of flights with prices, times, and airline information."
- Bad: "Searches stuff"

### Input Schema (required)

A JSON Schema object describing what parameters the tool accepts. If you don't know JSON Schema, here's the pattern:

**Simplest possible** (tool takes no input):
```json
{
  "type": "object",
  "properties": {},
  "required": []
}
```

**Tool with parameters**:
```json
{
  "type": "object",
  "properties": {
    "origin": {
      "type": "string",
      "description": "3-letter IATA airport code"
    },
    "destination": {
      "type": "string",
      "description": "3-letter IATA airport code"
    },
    "date": {
      "type": "string",
      "description": "Travel date in YYYY-MM-DD format"
    }
  },
  "required": ["origin", "destination", "date"]
}
```

**Common types**: `"string"`, `"number"`, `"boolean"`, `"array"`, `"object"`

### Output Format

What does the tool return?

| Format | When |
|--------|------|
| **JSON** | Structured data (most common). Search results, API responses. |
| **Text** | Plain text responses. Summaries, status messages. |
| **HTML** | HTML fragments. Rich content the agent might display. |
| **Mixed** | Multiple types depending on the call. |

### Error States

What can go wrong? One error per line.

Example:
```
InvalidInput: origin must be 3-letter IATA code
NetworkError: upstream API timeout after 30s
AuthRequired: user must be logged in for booking
RateLimited: maximum 10 searches per minute exceeded
```

### Checkboxes

| Checkbox | Check it if... |
|----------|----------------|
| **Read-only** | The tool only READS data, never changes anything. Search, query, list operations. |
| **Destructive** | The tool DELETES or IRREVERSIBLY CHANGES something. Delete account, cancel booking. |
| **Async** | The tool makes network requests or takes time. Almost always yes. |
| **requestUserInteraction()** | The tool pauses mid-execution to ask the human a question (e.g. "Confirm this purchase?"). This is unique to WebMCP -- backend MCP can't do this. |
| **Multimodal** | The tool handles images, audio, or video, not just text/JSON. |

---

## Tab 4: Session & Authentication

*Does the user need to be logged in?*

### Session Type (required)

| Type | Meaning |
|------|---------|
| **Authenticated** | Tools need a logged-in user. Most common for real applications. |
| **Anonymous** | Tools work for anyone, no login needed. Public search, demos. |
| **Mixed** | Some tools are public (search), some need login (booking). |

### Permissions Required

What level of access does the user need?

- "Logged-in user" (basic)
- "Admin role" (elevated)
- "Premium subscriber" (paid tier)
- Leave blank if anonymous.

### OAuth / Cookie Dependencies

What specific auth mechanisms do your tools rely on? One per line.

Example:
```
Session cookie from login flow
OAuth2 bearer token for API calls
CSRF token in X-CSRF-Token header
```

### Inherits Browser Session

**Almost always yes.** This is the key difference from backend MCP: your tools automatically get the same cookies, tokens, and session data as the logged-in user. You don't configure credentials separately.

---

## Tab 5: Browser Compatibility

*What browser does this need?*

### Minimum Chrome Version (required)

Currently WebMCP only works in **Chrome 146 Canary** with the "WebMCP for testing" flag enabled at `chrome://flags`.

Enter: `146` (the default)

### MCP-B Polyfill Support

Check this if your tools also work through the MCP-B Chrome extension, which provides WebMCP-like functionality in current stable Chrome.

### Browser-Specific Notes

Anything agents or developers need to know about browser support. One note per line.

Example:
```
Chrome DevTools MCP bridge required for Claude Desktop integration
Edge support expected mid-2026
Firefox: no timeline announced
Mobile Chrome: not yet supported
```

### Graceful Degradation

What happens for users whose browser doesn't support WebMCP?

- Good: "Falls back to standard HTML form submission. UI remains fully functional for human users."
- Bad: "Doesn't work."

---

## Tab 6: Security Properties

*What could go wrong? How bad could it get?*

### Checkboxes

| Checkbox | What it means |
|----------|---------------|
| **Same-origin scope** | Tools can only access data from their own domain. Almost always yes. |
| **CSP compatible** | Tools work with Content Security Policy headers. Check if you use CSP. |
| **Human-in-the-loop** | A human can see what the agent is doing. Core WebMCP design principle. |

### Data Exposure

What data do your tools expose to AI agents? Be specific.

- "Search results including flight prices and availability"
- "User profile name and email (for personalization)"
- "No user data exposed -- only public catalog information"

### Exfiltration Risk (required)

The **lethal trifecta** assessment. Three factors combine to create data theft risk:

1. **Data access** -- tool reads user data
2. **Untrusted content** -- page shows third-party content (ads, user-generated content, embeds)
3. **External communication** -- tool can send data to outside servers

| Risk | When |
|------|------|
| **Low** | Read-only tools OR no external communication. Most tools. |
| **Medium** | Some data access with controlled output. Tools that modify state but don't send data externally. |
| **High** | All three factors present. Tools that read user data on a page with third-party content and can make external network requests. |

### Lethal Trifecta Present

Check this ONLY if all three factors above are present simultaneously. Be honest -- this is a security assessment, not a marketing checkbox.

### Rate Limiting

Any throttling on tool invocations? Examples:
- "10 calls/minute per session"
- "100 calls/hour, shared with UI rate limits"
- "No rate limiting" (be careful with this one)

---

## Tab 7: Interaction Patterns

*How do tools behave in context?*

### Statefulness (required)

| Type | Meaning |
|------|---------|
| **Stateless** | Each tool call is independent. Most common. |
| **Stateful** | Tools remember previous calls. searchFlights result feeds into bookFlight. |
| **Mixed** | Some tools are stateless (search), some stateful (multi-step booking). |

### Tool Dependencies

Does calling one tool require another tool to have been called first? One dependency per line.

Example:
```
bookFlight depends on searchFlights result
selectSeat depends on bookFlight confirmation
```

### Modifies Visible Page State

Check this if the tool changes what the human sees on screen (updates DOM, changes UI). This is important because it IS the human-in-the-loop mechanism -- the human can see what the agent did.

### Latency Estimate

How long do tool calls typically take?

- "50-200ms for search operations"
- "1-3 seconds for booking confirmation"
- "Under 100ms for UI toggles"

### Blocks UI

Check if the tool freezes the page while executing. Ideally this should be unchecked -- tools should run without blocking the user.

---

## Tab 8: Context Requirements

*What does the tool read from the page?*

### provideContext() Data

What data does your page explicitly share with agents through the provideContext() API?

Example: "Current search filters, selected dates, passenger count, user locale preference"

### Reads DOM State

Check if the tool reads the visible page (HTML elements, text content, form values) during execution.

### Accesses React/Redux State

Check if the tool reads framework-level state (React hooks, Redux store, Vue reactive data). Note: this means your tool is coupled to a specific framework version.

### Navigation Dependencies

Does the tool only work on certain pages/routes?

- "Must be on /search results page for bookFlight to work"
- "Works on any page"
- Leave blank if no restrictions.

---

## Tab 9: Limitations & Known Issues

*What can't the tool do? Be honest.*

### Cannot Do (required)

One limitation per line. This is the most important honesty section.

Example:
```
Cannot book international flights
Cannot process payments directly
Maximum 10 passengers per search
Does not support multi-city routing
No wheelchair-accessible filtering yet
```

### Edge Cases

Known problematic scenarios. Things that might surprise users or agents.

Example:
```
Dates more than 330 days out return empty results
Airports with only charter flights may not appear
Search during maintenance windows returns stale data
```

### Scale Limits

Default is "50 tools per page recommended" -- this is the practical performance guideline from the WebMCP spec. Adjust if your page has specific constraints.

### Failure Modes

How does the tool fail? What does the agent see?

Example:
```
Returns {error: "TIMEOUT", message: "..."} after 30 seconds
Throws TypeError if called before page fully loads
Returns empty results array (not error) when no matches found
```

---

## Tab 10: Discovery Metadata

*Can agents find your tools before visiting the page?*

### Registry Listed

Check if your tools are listed in any tool registry or catalog.

### .well-known/webmcp

Check if your site serves a `.well-known/webmcp` file (proposed standard, not yet finalized). This would let agents discover your tools before navigating to the page.

### Pre-Visit Discoverability

How can agents know your tools exist before visiting the page?

- "Listed in site sitemap with WebMCP annotations"
- "Documented in this model card"
- "No pre-visit discovery -- tools only available after page load"

This is an unsolved problem in WebMCP. Your model card itself is part of the solution.

---

## Tab 11: Testing & Validation

*Has this been tested with real AI agents?*

### Test Methodology

How did you test your WebMCP tools?

Example:
```
Manual testing via Chrome DevTools MCP bridge
Automated benchmark comparing token usage vs screenshot method
User testing with 5 participants using Claude Desktop
```

### Agents Tested

Check all AI agents you've tested with. This builds trust -- agents that know they've been tested are more likely to be used confidently.

### Pass/Fail Criteria

What counts as a successful tool invocation?

Example:
```
Returns valid JSON with results array
No console errors during execution
Completes within 5 seconds
Human can see and verify state changes
```

### Test Suite Link

URL to your test scripts, benchmark suite, or CI pipeline. Leave blank if no automated tests.

---

## Tab 12: Backend MCP Relationship

*Is there also a backend MCP server for this service?*

### Corresponding Backend MCP Server

Check if your service also has a traditional backend MCP server (the Anthropic-style kind that runs on Node.js/Python via JSON-RPC).

Many real applications will have both: browser-side WebMCP for UI-interactive operations, and backend MCP for heavy processing, payments, or database operations.

### Backend Server Name

The name of the corresponding backend MCP server, if it exists.

Example: `flights-mcp-server`, `@company/booking-mcp`

### Capability Differences

What does the browser tool do differently from the backend server?

Example:
```
Browser: live UI updates, session-aware search, human confirmation dialogs
Backend: payment processing, database writes, batch operations
Browser inherits user session; backend needs separate API key
```

### Authentication Shared

Check if the browser session authenticates backend calls too (e.g. the same OAuth token works for both).

---

## After You Generate

1. **Review** the JSON and Markdown output
2. **Copy** whichever format you need (or both)
3. **Publish** alongside your WebMCP-enabled page:
   - Add the JSON to your repository
   - Add the Markdown to your README
   - Consider serving the JSON at `.well-known/webmcp-model-card`
4. **Share** with the WebMCP community for feedback

---

## Quick Reference: MCP vs WebMCP

| Aspect | Anthropic MCP (Backend) | W3C WebMCP (Browser) |
|--------|------------------------|---------------------|
| **Runs on** | Server (Node.js, Python, Docker) | Web page (browser JavaScript) |
| **Transport** | stdio, HTTP+SSE, JSON-RPC | postMessage, navigator.modelContext |
| **Auth** | API keys, env vars, OAuth config | Inherits browser session automatically |
| **Discovery** | MCP Registry, npm, .well-known | Page must be visited first (gap) |
| **Tool registration** | Server-side code only | HTML forms OR JavaScript (or both) |
| **Human oversight** | Separate approval flow | Human sees the page -- IS the oversight |
| **State** | Typically stateless | Can be deeply stateful (shares page runtime) |
| **Security boundary** | API key management | Same-origin policy + CSP + human-in-loop |
| **Unique capability** | Heavy compute, DB access, payment | requestUserInteraction(), DOM mutation, session inheritance |

---

*Part of the W3C AI Knowledge Representation Community Group's work on AI agent documentation standards.*

*Co-created by Paola Di Maio, PhD & Claude (Anthropic) -- February 2026*
