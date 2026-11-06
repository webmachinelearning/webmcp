# WebMCP: A generic tool calling primitive

Authors: Dominic Farolino & Khushal Sagar

This explainer contains three design updates to the WebMCP spec, prompted by Mozilla's feedback
collected in [issue #236](https://github.com/webmachinelearning/webmcp/issues/236) and
[issue #237](https://github.com/webmachinelearning/webmcp/issues/237).

## Generic tool namespace on `WindowProxy`

At TPAC, the group made a few resolutions, recorded in the
[minutes](https://www.w3.org/2026/10/27-webmachinelearning-minutes.html#9330).

 - Split the API into two halves:
     - Provider (`registerTool()`, `toolactivated`, `toolcancel`): only same-origin contexts can use
       it.
     - Consumer (`getTools()`, `executeTool()`, `toolchange`): both same-origin and cross-origin
       contexts can use it. `exposedTo` further controls which origins can see each tool.
 - Expose these APIs to Documents through `WindowProxy`. This means only Documents in the same
   browsing context group can reach them.

One question still open is naming. [Issue #236](https://github.com/webmachinelearning/webmcp/issues/236)
asks that the API become a general communication primitive rather than an LLM-specific feature. We
propose the name `tools` instead of `modelContext`, because tools are currently the only primitive
in the spec.

The other question is exactly how the interface is exposed through `WindowProxy`. The options are
below.

 1. Two interfaces: window.toolRegistry + window.tools (preferred)

    ```js
    await window.toolRegistry.registerTool(...);
    // works cross-origin.
    const theirs = await window.opener.tools.getTools();
    // throws "SecurityError"
    window.opener.toolRegistry;
    ```

    `toolRegistry` is the provider half. It is an ordinary Window member and same-origin only. tools
    is the consumer half which is added to CrossOriginProperties(Window). Every member of Tools is safe
    to use cross-origin. This follows the TPAC resolution directly with naming updates. This option
    best aligns with HTML's existing cross-origin model. Through a cross-origin WindowProxy, a
    caller can reach primitives and functions created in the caller's realm. Every Tools member is
    safe to use cross-origin, so the tools getter can work like the cross-origin functions. It
    returns a Tools handle that is created in the caller's realm and bound to the target window.
    Nothing from the target realm reaches the caller, so Tools needs no exotic behavior, and results
    and toolchange events are delivered in the caller's realm. Worker and ServiceWorker objects
    already follow this handle model, so one Tools interface works for both windows and workers.

 2. Single window.tools interface

    ```js
    await window.tools.registerTool(...);
    // works cross-origin
    const theirs = await window.opener.tools.getTools();
    // throws "SecurityError"
    window.opener.tools.registerTool(...);
    ```

    One `Tools` object holds both halves, and only the consumer members work cross-origin. Tools
    would become a third exotic object with a per-member split, after WindowProxy and Location. It
    would need Location-style overrides of the internal methods, which HTML and WebIDL discourage
    and engines support only for Window and Location. Because the object lives in the target realm,
    toolchange can only be same-origin. Worker and ServiceWorker objects would still need a
    consumer-only interface.

 3. Direct members on window
      - All methods go directly on Window (`otherWindow.getTools()`), and each method's cross-origin
        access depends on which half it belongs to. This adds each consumer method to
        `CrossOriginProperties(Window)` one by one, and it mixes both halves on the global.
        `toolchange` would fire on Window in the target realm, so it has the same problem as in
        option 2. Worker and ServiceWorker would need the methods added to them directly, so there
        is no single interface for windows and workers.


## Tool exposure to external agents

To make tool registrations useful for both web callers, we propose a design for separating the tool
parameters required for web callers, from those required for the AI use case and other natural
language consumers.

```js
window.tools.registerTool({
    name: "search_flight",
    naturalLanguageProperties: {
        description: "Provides real-time flight search data with the given search constraints",
        inputSchema: <foo>, // Possible required.
        outputSchema: <foo>, // Possible required.
        annotations: {...} // Optional.
    },
    exposedTo: ["https://origin.com"],
    execute: () => {...}
});
```

The inclusion of `naturalLanguageProperties` directs the browser and other web consumers to expose
the tool to native or external agents via whatever native mechanism or protocol, while `exposedTo`
continues to remain for controlling exposure to web consumers, by origin.

## Structured cloneable tool inputs/outputs

WebMCP tool inputs and outputs are currently constrained to JSON. This is reasonable for exclusively
model-facing tools, but it's an unnatural restriction for a generic Web-to-Web tool invocation primitive.
JavaScript and the Web platform already have a richer value model for crossing realm boundaries:
[structured serialization](https://html.spec.whatwg.org/C#serializable-objects). We propose allowing WebMCP
tools to accept as input, and return as output, the structured-cloneable types on the web platform,
such as `Date`, `Map`, `Set`, `Blob`, `File`, `ArrayBuffer`, and `ImageData`.  For example, a tool
should eventually be able to express a contract conceptually like:

```js
window.tools.register({
  name: "set-profile-picture",
  model: {
    description: "Set the user's profile picture.",
    inputSchema: {
      type: "object",
      properties: {
        image: { /* ImageData */ }
      },
      required: ["image"]
    }
  },
  execute({ image }) {
    // `image` is an official ImageData DOM object.
  }
});
```

The exact schema syntax for declaring Web types remains an open question. Possibilities include
extending the existing `type` vocabulary or introducing a WebMCP-specific schema keyword.

### Web callers use Web values

When one Web context invokes a tool exposed by another, values would cross the boundary using
structured-clone semantics rather than being forced through JSON.

A tool accepting a `Date` receives a `Date`; a tool returning an `ImageData` returns an
`ImageData`. Authors should not need to create a second, JSON-shaped representation of their
application's data merely because the same primitive can also be exposed to an agent.

This is especially important if `window.tools` is intended to become a generally useful
cross-realm invocation primitive rather than an API whose semantics are intrinsically tied to LLMs.

### Agents see a standardized JSON lowering

Agents external to the browser still require JSON-compatible tool schemas and arguments.
For those callers, the user agent would expose a lowered version of such schemas and arguments.

Each supported Web type would have a standardized mapping to:

 - a valid model-facing JSON Schema;
 - serialization steps from the Web value to that representation; and
 - deserialization steps from model-produced JSON back to the Web value.

For example, a `Date` might be exposed as a standardized date-time string, a `Map` might lower
to an array of key/value pairs, and an `ImageData` might be a simple JSON-ified version of its
[serialization record](https://html.spec.whatwg.org/C#the-imagedata-interface:serialization-steps).

The "JSON lowering" would be specified by WebMCP, and apply to all `[Serializable]` types, rather
than being an implementation-defined browser wire format that browsers actually use under the hood
to represent serialization records. This allows third-party agent harnesses to understand the
advertised representation and avoids browser-specific encodings becoming de facto protocols.

### Better error handling

[Issue #282](https://github.com/webmachinelearning/webmcp/issues/282) asks for a better, structured,
MCP-like way of signaling a userland "error" to agents. While we could specify a special
`isError: true`-shaped JSON object, like MCP, that tools can return to the agent, the proposal to
expand the allowable types to all structured cloneable types gives us an opportunity to express
that same information to agents in a more ergonomic, "webby" way.

Specifically, since DOMException and the native JavaScript `Error` types are both "serializable",
their standardized JSON lowering can include all of the relevant information that the error object
contains, packaged up into an appropriate JSON representation derived from the serialization. The
serialization will exclude any internal details (like stack traces).

### Open design questions

The detailed proposal will need to answer several issues, including:

 - the syntax used to declare Web types in what is otherwise JSON-only schemas;
 - whether containers such as Map and Set declare their key/value/element types, or how
   otherwise-unconstrained nested structured-cloneable values are represented;
 - how cycles and shared references are handled;
 - strict versus forgiving deserialization of model-produced values;
 - the exact representation files, images, and other binary values;
 - payload-size limits and whether browsers may transform large values;
 - the initial allowlist of supported types; and
 - versioning of standardized lowerings.

These details are intentionally deferred from this joint explainer. A separate explainer will specify
the structured-cloneable type system, model-facing lowerings, prior art, security considerations, and
per-type conversion algorithms in detail.
