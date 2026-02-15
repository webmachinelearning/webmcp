# WebMCP Technical Note 3: WebMCP Is Not an MCP Server

**Anthropomorphic Press -- Technical Note 3**
**15 February 2026**

---

A persistent claim in the WebMCP ecosystem is that WebMCP turns a website into an MCP server. The W3C specification repository itself states that web pages using WebMCP "can be thought of as Model Context Protocol (MCP) servers that implement tools in client-side script instead of on the backend." Early independent implementations by Jason McGhee and Alex Nahas (MCP-B) literally did function as MCP servers, bridging browser JavaScript to MCP clients through localhost websocket connections using the standard MCP protocol.
([W3C spec repo](https://github.com/webmachinelearning/webmcp))
([McGhee implementation](https://github.com/jasonjmcghee/WebMCP))
([Nahas MCP-B](https://github.com/MiguelsPizza/WebMCP))

The framing is understandable. It is also architecturally misleading, and the confusion has consequences for how developers, security reviewers, and standards participants evaluate the specification.

## The Analogy and Its Limits

WebMCP and Anthropic's Model Context Protocol share a conceptual ancestor: both define "tools" as functions with natural language descriptions and structured schemas that AI agents can discover and invoke. That is where the meaningful similarity ends.

**Anthropic's MCP** is a backend protocol. It uses JSON-RPC 2.0 as its message format, transported over stdio, HTTP with Server-Sent Events, or Streamable HTTP. MCP servers are hosted processes -- typically written in Python or Node.js -- that run on backend infrastructure. They connect AI platforms like Claude, ChatGPT, or Gemini to external services. Authentication follows OAuth 2.1 or custom API key schemes. No browser is required. No human user needs to be present. Headless, fully automated operation is the norm.
([Source](https://modelcontextprotocol.io/introduction))

**WebMCP** is a frontend browser API. It uses the browser's native postMessage system for communication between the web page and the agent. Tools are registered and executed as client-side JavaScript within an active browser tab. Authentication is inherited from the browser session -- whatever cookies or federated login the user already has. A human user must be present in an active browser session. Headless browsing is explicitly out of scope.
([Source](https://webmachinelearning.github.io/webmcp/))

The specification's own language -- "can be thought of as" -- acknowledges this is an analogy, not an identity. But the README, the press coverage, and the developer ecosystem have largely dropped the qualifier. The result is that WebMCP is widely discussed as though it were MCP running in the browser, with all the assumptions that entails.

## What the Framing Gets Wrong

When a developer hears "your website becomes an MCP server," they import a set of assumptions from the MCP architecture. Every one of these assumptions is wrong for WebMCP.

**Transport.** MCP uses JSON-RPC 2.0, a well-specified request-response protocol with defined error codes, batching, and notification semantics. WebMCP uses postMessage, the browser's cross-origin communication mechanism. These have different reliability characteristics, different error handling models, and different security boundaries. Code written for one transport does not work with the other.

**Execution context.** An MCP server runs in a controlled backend environment -- a container, a VM, a serverless function -- where the service provider manages the runtime, dependencies, and resource limits. WebMCP tools run in the browser's JavaScript engine, in the same execution context as the web page's own code. They are subject to the browser's security sandbox, but also to its constraints: single-threaded execution, same-origin policy, and the full surface area of client-side attack vectors.

**Authentication.** MCP's specification has adopted OAuth 2.1 for authentication between clients and servers. This was, notably, the problem that motivated WebMCP's creation -- Alex Nahas at Amazon found that OAuth 2.1 was impractical for internal MCP deployments. WebMCP sidesteps this entirely by inheriting the browser session. This is elegant for usability but means the authentication model is whatever the website happens to use, with no protocol-level guarantees.

**Trust direction.** In MCP, the AI platform (client) connects to a known, registered server. The platform decides which servers to trust. In WebMCP, any website the user visits can register tools. The trust decision shifts from the AI platform to the browser, and potentially to the user -- who may not know that tools have been registered at all, since the current specification provides no visible indicator.

**Operational mode.** MCP servers are designed for automated, programmatic access. They can run continuously, handle concurrent requests, and operate without human involvement. WebMCP requires an active browser tab with a human user present. The specification explicitly excludes headless browsing. These are fundamentally different operational paradigms with different scaling characteristics, different failure modes, and different abuse surfaces.

## Why This Matters for Standards Review

The "MCP server" framing is not just imprecise. It actively interferes with rigorous evaluation of the specification.

**Security reviewers** who approach WebMCP as "MCP in the browser" will evaluate it against MCP's threat model. But MCP's threat model assumes a controlled backend environment, authenticated client-server connections, and server-side access control. WebMCP's actual threat model involves client-side JavaScript execution, browser-based trust boundaries, and the full range of web security concerns including cross-site scripting, prompt injection via tool responses, and silent tool registration. Importing the wrong threat model means asking the wrong security questions.

**Developers** who approach WebMCP as "MCP in the browser" may expect protocol-level interoperability -- that a WebMCP tool definition could be used interchangeably with an MCP server tool definition, or that MCP client libraries could connect to WebMCP pages. They cannot. The tool schema format may be similar, but the transport, discovery, and invocation mechanisms are incompatible.

**Standards participants** who approach WebMCP as "MCP in the browser" may underestimate the scope of new specification work required. WebMCP is not an adaptation of MCP to a new environment. It is a new browser API that borrows one concept (the tool abstraction) from MCP and implements everything else differently. It needs its own security review, its own privacy analysis, its own accessibility evaluation, and its own consent model -- none of which can be inherited from MCP.

## What WebMCP Actually Is

WebMCP is a proposed browser API -- specifically, a new interface on navigator.modelContext -- that allows web pages to declare JavaScript functions as tools that browser-based AI agents can discover and invoke. It uses the browser's existing communication, security, and session management infrastructure rather than introducing a new protocol.

The design has real strengths. Authentication reuse eliminates one of the hardest problems in AI-service integration. Client-side execution means no backend infrastructure is needed. The human-in-the-loop requirement provides a natural consent and oversight mechanism -- if implemented correctly.

But these strengths are specific to WebMCP's actual architecture, not to the MCP analogy. Evaluating WebMCP on its own terms -- as a browser API with browser security characteristics -- leads to better questions, better testing, and better specifications than evaluating it as a variant of MCP.

## A Suggested Clarification

The W3C specification and its README should explicitly state that WebMCP is not an implementation of the Model Context Protocol and does not use the MCP wire protocol. It borrows the "tool" abstraction -- functions with schemas and natural language descriptions -- but implements discovery, registration, invocation, and communication through browser-native mechanisms that are architecturally distinct from MCP.

The analogy is useful for first contact. A developer unfamiliar with WebMCP can quickly grasp the concept by thinking "it is like an MCP server, but in the browser." But the specification itself, the security review, and the community evaluation should not rely on the analogy. They should address WebMCP as what it is: a new browser API with its own architecture, its own threat model, and its own design space.

## Both Can Coexist

None of this is an argument against WebMCP or against MCP. A company might maintain an MCP server for direct API integrations with AI platforms and simultaneously implement WebMCP tools on its consumer-facing website for browser-based agent interaction. The two are complementary, not competing, and not identical. Recognizing the distinction is necessary for evaluating each on its own merits.

---

*Anthropomorphic Press, indexed in Dow Jones Factiva. CWRE*
