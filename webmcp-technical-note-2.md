# WebMCP Technical Note 2: What to Test, What to Watch, What to Tell the Standards Body

**Anthropomorphic Press -- Technical Note 2**
**15 February 2026**

---

Google's WebMCP early preview is live in Chrome 146 Canary. The specification is still a draft. The community group process is still open. This means the window for meaningful community input is right now -- before implementation momentum makes the current design effectively permanent.

This note is a practical guide. It is written for developers, accessibility practitioners, security researchers, standards participants, and anyone who builds things for the web and wants to understand what WebMCP means for their work. It covers what WebMCP is for, what to test, what the benefits are, what the risks are, and how to communicate findings to the W3C community group that hosts the specification.

## What WebMCP Is For: The Use Cases

WebMCP allows a website to declare a set of tools -- JavaScript functions with structured schemas and natural language descriptions -- that AI agents can discover and invoke. The specification targets several categories of use.

The first is **e-commerce and transactional sites**. A travel booking site could register tools like searchFlights(origin, destination, dates), filterResults(price, stops, airline), and bookFlight(flightId, passengerDetails). Instead of an AI agent trying to parse a complex search interface by reading pixels or DOM elements, it calls the function directly and gets structured JSON back. The site controls exactly what the agent can do and how.

The second is **productivity and SaaS applications**. A project management tool could expose createTask(title, assignee, dueDate), moveCard(cardId, column), and generateReport(dateRange). Browser-based AI assistants could help users manage workflows without the application needing to build and maintain a separate backend MCP server or API integration for every AI platform.

The third is **content and media**. A news site could register searchArticles(topic, dateRange) and getArticleSummary(articleId). A mapping service could expose getDirections(from, to, mode) and findNearby(category, radius). These tools let agents interact with content in structured ways rather than scraping and guessing.

The fourth -- and potentially most significant -- is **accessibility**. The specification claims WebMCP could benefit assistive technologies by providing structured, semantically meaningful interfaces to website functionality. A screen reader enhanced with agent capabilities could invoke tools directly rather than navigating complex visual layouts. This is a strong claim that deserves rigorous testing.

The fifth is **form automation and multi-step workflows**. Complex processes like insurance applications, government forms, or account setup flows could be exposed as sequences of tool calls, allowing agents to guide users through them step by step while the site maintains control over validation, sequencing, and data handling.

## What the Benefits Are

WebMCP offers several concrete advantages over current approaches to AI-web interaction.

**Reliability** is the most immediate. Today's browser agents -- whether using visual parsing or DOM inspection -- are brittle. A minor CSS change can break a visual agent. A DOM restructuring can invalidate a scraping approach. WebMCP tools are explicit contracts: the site declares what is available, the agent calls it, the response is structured. This should dramatically reduce failure rates for agent-web interaction.

**Performance** is the second. Visual agents must capture screenshots, send them to a vision model, interpret the response, generate mouse coordinates, and repeat. WinBuzzer reported a 67% reduction in computational overhead with WebMCP compared to visual approaches. Even if that number proves optimistic in production, the architectural advantage is clear: a function call is faster than a screenshot-interpret-click loop.

**Developer control** is the third. With visual or DOM-based agents, the website has no say in how an agent interacts with it. The agent reverse-engineers the interface. With WebMCP, the developer explicitly defines the interaction surface. Tools can include rate limits, validation, permission requirements, and structured error messages. The site becomes a willing participant in the interaction rather than a passive target.

**Authentication reuse** is the fourth, and was the original motivation. Because WebMCP runs in the browser session, it inherits whatever authentication the user already has. No OAuth flows, no API keys, no separate credential management. The user is already logged in. The agent operates within that session. This solves one of the hardest problems in AI-service integration.

**Standardization** is the fifth. If WebMCP succeeds, a developer implements tools once and every conformant agent can use them -- rather than building separate integrations for ChatGPT, Claude, Gemini, and whatever comes next. This is the "USB-C" argument: one interface, many devices.

## What the Risks Are

The risks are significant, and several are not yet adequately addressed in the specification.

**Prompt injection** is the most acute. WebMCP tools return data to AI agents that then process it in their language model context. A malicious or compromised website could craft tool responses that manipulate the agent's behavior -- injecting instructions, altering the agent's understanding of the task, or causing it to take unintended actions on other sites. The specification does not currently define a defense against this beyond same-origin policy boundaries.

**Scope creep of agent permissions** is the second. WebMCP is designed for human-in-the-loop workflows, with headless browsing explicitly out of scope. But the technical mechanism -- JavaScript functions callable by external code -- does not inherently enforce this. If browser vendors later relax the human-presence requirement, or if extensions find ways to invoke WebMCP tools without user awareness, the permission model collapses. The specification should define what "human in the loop" means technically, not just philosophically.

**Consent and transparency** is the third. When a user visits a site that registers WebMCP tools, do they know? The current design provides no visible indicator to the user that tools have been registered, what data they expose, or when an agent invokes them. Compare this to other browser permission systems -- camera, microphone, location -- where the user explicitly grants access. WebMCP tools operate silently.

**Competitive dynamics** is the fourth. WebMCP gives first-mover advantage to sites that implement tools early, potentially favoring large platforms with engineering resources. Smaller sites that do not implement WebMCP may become invisible to agent-mediated browsing. This could accelerate web consolidation. The specification should consider whether a minimal tool set (search, navigation, content retrieval) should be automatically generated from existing web standards like HTML forms, structured data, and ARIA attributes.

**Data leakage through tool schemas** is the fifth. The natural language descriptions and parameter schemas of registered tools reveal information about a site's internal architecture, business logic, and data models. An agent -- or the platform behind it -- could catalog available tools across thousands of sites to build competitive intelligence. The specification does not address whether tool schemas should be treated as sensitive information.

**Abuse and rate limiting** is the sixth. Agents can invoke tools at machine speed. A poorly defended site could face thousands of tool invocations per second from a single browser session. The specification mentions rate limiting as a consideration but does not define a standard mechanism. Without one, each site must build its own defenses, and many will not.

**Cross-site tool chaining** is the seventh. If an agent can invoke tools on multiple open tabs, it could chain actions across sites in ways no individual site anticipated or authorized. Transfer money on a banking site, then use the confirmation on a shopping site, then post about it on a social network -- all within one agent workflow. The security boundaries for cross-site tool interaction are not yet defined.

## What to Test

For those with access to Chrome 146 Canary, here are the concrete areas that need community evaluation. Each should generate feedback for the W3C community group.

**Test the Declarative API with real HTML forms.** Register tools that wrap existing form actions and verify that validation, error handling, and submission behavior match what a human user would experience. Try edge cases: forms with CAPTCHAs, multi-step forms with session state, forms that redirect on submit. Document where the abstraction breaks.

**Test the Imperative API with dynamic content.** Register tools that interact with JavaScript-heavy applications -- single-page apps, dashboards with real-time data, applications that maintain complex client-side state. Evaluate whether tool calls can reliably interact with application state without causing inconsistencies.

**Test authentication boundaries.** Log into a site, register tools, then observe what happens when the session expires, when the user logs out in another tab, when cookies are cleared. The specification's authentication reuse claim needs verification under adversarial conditions.

**Test tool discovery and enumeration.** If multiple sites in different tabs register tools, how does the agent disambiguate? What happens when two sites register tools with the same name? How does the agent present available tools to the user? Is tool discovery observable by the page (can a site detect that an agent has read its tool list)?

**Test accessibility integration.** If you work with assistive technologies, evaluate whether WebMCP tools provide genuinely better access to site functionality than existing ARIA roles and landmarks. Test with screen readers, switch access devices, and voice control. Document whether WebMCP complements or conflicts with existing accessibility standards.

**Test prompt injection resilience.** Craft tool responses that contain instruction-like text and observe whether the consuming agent's behavior is affected. This is critical safety research. If tool responses can manipulate agent behavior, the security model is fundamentally incomplete.

**Test performance claims.** Measure actual latency and token usage for equivalent tasks performed via WebMCP tools versus visual agent interaction. The 67% overhead reduction claim needs independent verification across different site types and task complexities.

**Test failure modes.** What happens when a tool throws an error? When it returns unexpected data types? When it hangs? When the page navigates away mid-call? The specification should define standard error handling, but the current draft has TODO sections in these areas. Documenting real failure modes will directly shape the specification.

## How to Communicate Findings

Feedback is only useful if it reaches the people writing the specification. Here are the concrete channels, in order of effectiveness.

**File a GitHub issue** at https://github.com/webmachinelearning/webmcp/issues with a clear title, reproducible steps, and a specific recommendation. Tag it with the relevant label if available. The spec editors (Brandon Walderman, Khushal Sagar, Dominic Farolino) monitor this repo. Issues with reproducible test cases and concrete proposals get traction. Issues that say "I don't like this" do not.

**Join the W3C Web Machine Learning Community Group** at https://www.w3.org/community/webmachinelearning/ and participate in discussion. Community Groups are free and open. Participation in CG calls and mailing list threads carries weight in W3C process.

If your findings relate to **agent interoperability** -- how WebMCP tools interact with broader agent ecosystems, discovery protocols, or multi-agent workflows -- also engage with the AI Agent Protocol Community Group, which the WebML CG charter identifies as a coordination partner.

If your findings relate to **security or privacy**, file issues with clear severity assessments. W3C specifications have a tradition of security and privacy self-review questionnaires. Check whether the WebMCP specification has completed one, and if not, request it.

If you publish your findings -- on a blog, in a report, in an academic paper -- link back to the relevant GitHub issues so the discussion stays connected to the specification process.

## The Window

The pattern in web standards is well established. Once an implementation ships in a dominant browser and developers build on it, the specification follows the code. Chrome holds roughly 65% of browser market share. The early preview is live. Developer adoption is beginning. The longer the community waits to engage, the narrower the design space becomes.

This is not an argument against WebMCP. The technical concept is sound, the use cases are real, and the problem it solves -- giving developers control over AI agent interaction -- is important. But a good idea implemented badly, or without adequate security review, or without accessibility testing, or without community input, becomes a liability embedded in the web platform for decades.

The specification is at https://webmachinelearning.github.io/webmcp/. The implementation is in Chrome 146 Canary. The issues page is at https://github.com/webmachinelearning/webmcp/issues. The community group is open to all at no cost. The work is now.

---

*Anthropomorphic Press, indexed in Dow Jones Factiva. CWRE*
