# WebMCP: A generic tool calling primitive

Authors: Dominic Farolino & Khushal Sagar

This explainer contains design updates to the WebMCP spec, prompted by Mozilla's feedback collected in [issue \#236](https://github.com/webmachinelearning/webmcp/issues/236) and [issue \#237](https://github.com/webmachinelearning/webmcp/issues/237) to support 2 use-cases:

1. Generic Function Calling (Web to Web): The base infrastructure in WebMCP is function calling/RPC. This is currently polyfilled by authors over postMessage with libraries like [comlink](https://github.com/googlechromelabs/comlink).  
2. Tools for models/agents (Web to model): Defining APIs which AI models can understand and invoke.

# Generic Function Calling

The `modelContext` APIs are split into 3 parts: Managing tool registration and execution, Tool Discovery and Tool Execution. The APIs below are not guarded by any permission policy, the only constraint is that the Documents could correspond to any browsing context within the same browsing context group and the consumer has a window reference to the tool provider. [exposedTo](https://webmachinelearning.github.io/webmcp/#dom-modelcontextregistertooloptions-exposedto) remains the same as the current spec and controls which origins can see the registered tools.

**These APIs support all [serializable objects](https://html.spec.whatwg.org/C#serializable-objects).**

## Managing tool registration and execution

A Document uses `window.toolManager` to register its tools, handle their execution, and control when they are ready. Other naming options are toolHost, toolProvider, toolRegistrar.

```js
dictionary ToolManagerTool {
  required DOMString name;
  required ToolExecuteCallback execute;
};

dictionary ToolExecuteCallbackOptions {
  required AbortSignal signal;
};

// Both `inputObject` and the resolved value must be serializable objects:
// https://html.spec.whatwg.org/C#serializable-objects
callback ToolExecuteCallback = Promise<any> (object inputObject, ToolExecuteCallbackOptions options);
dictionary ToolManagerRegisterToolOptions {
  sequence<USVString> exposedTo;
  AbortSignal signal;
};

partial interface Window {
  // Same-origin only. Returns the ToolManager of this Window's associated
  // Document. Not [SameObject]: the Window is reused when navigating away from
  // the initial about:blank, but the Document (and so the ToolManager) is new.
  [SecureContext] readonly attribute ToolManager toolManager;
};

[Exposed=Window, SecureContext]
interface ToolManager : EventTarget {
  Promise<undefined> registerTool(
ToolManagerTool tool,
       optional ToolManagerRegisterToolOptions options = {});
  undefined waitForToolsUntil(Promise<any> promise);
  attribute EventHandler ontoolactivated;
  attribute EventHandler ontoolcancel;
};
```

## Tool discovery

These APIs let a caller find the tools of a target browsing context's active `Document` and be notified when these tools change. The caller needs a `WindowProxy` for the target: `iframe.contentWindow`, `window.parent`, `window.top`, `window.opener`, or a popup returned by `window.open()`. [fromOrigins](https://webmachinelearning.github.io/webmcp/#dom-modelcontextgettooloptions-fromorigins) remains the same as the existing spec.

```js
partial interface Window {
  // Added to CrossOriginProperties(Window), like postMessage().
  [SecureContext] Promise<sequence<Tool>> getTools(optional GetToolsOptions options = {});
};

dictionary GetToolsOptions {
  sequence<USVString> fromOrigins;
};

callback ToolObserverCallback = undefined (sequence<ToolObserverRecord> records,
                                           ToolObserver observer);

[Exposed=Window, SecureContext]
interface ToolObserver {
  constructor(ToolObserverCallback callback);
  // Queues an initial record for target, then a record each time the
  // filtered result changes.
  undefined observe(Window target, optional GetToolsOptions options = {});
  undefined unobserve(Window target);
  undefined disconnect();
};

[Exposed=Window, SecureContext]
interface ToolObserverRecord {
  readonly attribute Window target;
  // The current filtered result, equivalent to target.getTools(options).
  readonly attribute FrozenArray<Tool> tools;
};
```

`getTools()` provides a convenient way to make a single query without constructing an observer.  For a cross-origin target, HTML creates the `getTools` function that the caller receives in the caller's realm so the returned promise and Tools are also in the caller’s realm. For a same-origin target though, the `getTools` that runs is the target `Window`'s own method. This is a problem because the returned promise and Tools would be created in the target’s realm and are destroyed when the target navigates. *The API must consistently return objects in the incumbent’s realm.*

`ToolObserver` is constructed in the caller's realm by design. The observer is bound to the target's browsing context, so it keeps working across same-origin and cross-origin navigations of the target. Following the [observer pattern](https://w3ctag.github.io/design-principles/#events-vs-observers:~:text=The%20Observer%20pattern%20works%20like%20this%3A), `observe()` queues an initial record even when the list is empty, and records are batched and delivered asynchronously. A record is *only* queued when the filtered result changes. This ensures that navigations between Documents which don't expose tools to the caller are not revealed. When the target's browsing context is destroyed (the iframe is removed, or the popup is closed), the observer queues one final record with an empty `tools` list and stops observing that target. The caller can check `record.target.closed` to tell "the target was destroyed" apart from "the target has no tools for me".

### Alternatives considered

#### EventTarget instead of Observer

```js
[Exposed=Window, SecureContext]
interface ToolList : EventTarget {
  constructor(Window target, optional GetToolsOptions options = {});
  Promise<sequence<Tool>> getTools();
  attribute EventHandler onchange;
};
```

**Pros**

* Web design principles [prefer](https://w3ctag.github.io/design-principles/#events-vs-observers) EventTargets instead of observers.  
* The tool query and change notification API is tied into a single interface.

**Cons**

* After a `change` event, the caller has to make another async `getTools()` call. An observer record delivers the change and the result together.  
* Events are not batched across all observed targets.

#### Tool object on Window

```js
partial interface Window {
  [SecureContext, CrossOrigin] readonly attribute Tools tools;
};
interface Tools : EventTarget {
  Promise<sequence<Tool>> getTools(optional GetToolsOptions options = {});
  attribute EventHandler ontoolchange;
};
```

**Pros**

* The tool query and change notification API is tied into a single interface.  
* There’s only one instance of the interface per (caller, target) pair.

**Cons**

* Managing the `tools` object's lifetime correctly needs complicated new machinery in HTML. The object must be created in the caller's realm and cached per (caller, target browsing context), so that it survives navigations in the target context. There's no place to cache an object with these semantics:  
 
  * [Cross-origin property state](https://html.spec.whatwg.org/multipage/nav-history-apis.html#shared-internal-slot:-crossoriginpropertydescriptormap) lives on the target Window, so it's lost when the browsing context navigates to a new Window.  
  * A browsing context holds no JS objects other than its WindowProxy, and it shouldn't hold objects from other browsing contexts' realms.

  The only existing cross-origin property that returns an object, location, returns the target's own Location. That object goes stale on navigation, which is acceptable only because it's used immediately. Any design that puts event listeners on an object whose lifetime is tied to the target Window loses those listeners when the target navigates.

* Change notifications can't be filtered by `fromOrigins`, so the caller gets redundant events.

## Executing a Tool

`Tools` handles are created in the caller's realm and bound to the Document that registered it. When a caller has a `Tool` reference, it calls `tool.execute()` to run the tool. Executing a tool which has been unregistered throws an error. Other naming options for execution are run, invoke. call was considered but its a JS built-in for functions, which causes problems if a site puts Tool and Function in the same prototype chain.

Since DOMException and the native JavaScript \`Error\` types are both [serializable](https://html.spec.whatwg.org/C#serializable-objects), the error thrown from ToolManager’s `ToolExecuteCallback` is propagated to the tool.execute() caller. The serialization will exclude any internal details (like stack traces).

\[\!WARNING\] It's an open question as to whether we want to simply return userland errors exactly as they are to callers, or wrap all of them in a new error interface that's not constructible from JavaScript, like `ToolError` or `ToolRuntimeError`. The only benefit of this would be to make it impossible for a userland tool error from spoofing one of the errors generated by the browser when a tool fails for reasons outside of the tool implementation's control.

```js
dictionary ToolExecuteOptions {
  AbortSignal signal;
};

[Exposed=Window, SecureContext]
interface Tool {
  readonly attribute DOMString name;
  readonly attribute WindowProxy? source;
  readonly attribute USVString origin;

  // Both `inputObject` and the resolved value must be serializable objects:
  // https://html.spec.whatwg.org/C#serializable-objects
  Promise<any> execute(optional object inputObject, optional ToolExecuteOptions options = {});
};
```

# Tools for models/agents

Tools exposed to models need the input/output to be converted to a well-defined JSON representation. This is not supported for all [serializable objects](http://html.spec.whatwg.org/multipage/structured-data.html#serializable-objects), the browser serializes these objects for transfer between realms but the format of the serialized representation is an internal detail. Having a well-defined format is necessary to allow third-party agent harnesses to understand the advertised representation and avoids browser-specific encodings becoming de facto protocols.

We’ll define a **JSON compatible subset** of serializable objects which can be converted to JSON and can be used with agent exposed tools. Each supported Web type would have a standardized mapping to:

* a valid model-facing JSON Schema;  
* serialization steps from the Web value to that representation; and  
* deserialization steps from model-produced JSON back to the Web value.

A new dictionary is added to combine all primitives needed for agent exposed tools. If naturalLanguageProperties is specified, the author opts-in to exposing tools to the user-agent. Because the API provides access to a powerful browser feature (adding tools to the browser’s built-in agent), exposing tools to the user-agent is gated using a new permission policy called “user-agent-tools” with a default allowlist of 'self'. There’s no other way to prevent the user-agent from using these tools and they are still visible to other web consumers based on exposedTo.

```js
dictionary NaturalLanguageProperties {
  USVString title;
  required DOMString description;
  required object inputSchema;
  required object outputSchema;
  ToolAnnotations annotations;
};

dictionary ToolManagerTool {
  ...
  NaturalLanguageProperties naturalLanguageProperties;
};
```

The json-schema supported by inputSchema/outputSchema corresponds to the JSON convertible subset. The browser validates that the tool input/output generated at execution time match this json-schema.

Similar to tool output serialization on success, the DOM exception thrown from ToolManager’s `ToolExecuteCallback` is serialized to the JSON subset. The values used in the exception can are limited to the JSON compatible subset.

Note: While the primary consumer of these properties is agents, deterministic or automation systems could also be built using them ([example](https://bandarra.me/posts/webmcp-macros)). Other naming options like `ToolMetadata` can be considered to account for such use-cases.

# Open Items

* Detailed json-schema subset supported by the spec, extensibility of the subset and schema validation semantics ([WIP](https://gist.github.com/domfarolino/5d847e7a0638fa67a866a4b1c5d7381f)).&nbsp;  
* Data structure for tool output to the model to distinguish success/error cases.

# Future Work

## Worker Integration

This design needs to be extensible to workers going forward. Here’s how the proposed API split will work there:

1. `ToolManager` is added to [`WorkerGlobalScope`](https://html.spec.whatwg.org/multipage/workers.html#the-workerglobalscope-common-interface), so a worker registers tools with `self.toolManager.registerTool(...)`.

2. `getTools()` moves to the [`WindowOrWorkerGlobalScope`](https://html.spec.whatwg.org/multipage/webappapis.html#windoworworkerglobalscope-mixin) mixin, so any context can query its own tools. For `Window`, it stays in `CrossOriginProperties(Window)`. It's also added to the [`Worker`](https://html.spec.whatwg.org/multipage/workers.html#dedicated-workers-and-the-worker-interface) interface, so the owner of a worker can query that worker's tools. The `Worker` object is created in the caller's realm, so the promise and the `Tool` objects it returns are also in the caller's realm. This is the same handle model that the cross-origin `getTools()` on `Window` uses.

3. `ToolSource` is defined as any object a caller holds that refers to a context that can expose tools: a `WindowProxy` for a Document, or a `Worker` object for a dedicated worker. `ToolObserver.observe()` switches to accept any `ToolSource`.

## Composed Tool Set

Web applications are often made of several frames, from the same origin or from different origins. To build the full set of tools an application offers, a consumer needs to collect tools from the relevant descendant frames, not just the top document. The consumer could be an ancestor frame asking a subframe for its tools, or an opener asking a popup. This was requested at TPAC ([resolution](https://github.com/webmachinelearning/webmcp/issues/227#issuecomment-6609532024)). The API for this *composed tool-set* needs to solve the following problems:

**Frame Selection**
Generic Function Calling requires the consumer to know which origin it is pulling tools from. For subtree discovery, the consumer should only need the origin of the document it queries (the query root). How the application is split across frames is an internal detail, and the consumer shouldn't have to list every origin the query root embeds. It can still filter by each tool's origin if it wants to. So the query root, not the consumer, decides which frames make up its composed tool-set. exposedTo still requires the provider to opt-in to tool exposure.

Note: Just embedding a frame doesn’t imply it should contribute to the embedder’s composed tool-set : a page may embed ads, widgets or other third-party content whose tools shouldn't represent the page's capabilities.

We evaluated using permission policy for this use-case. It's an existing way for an embedder to make decisions about its iframes, and its allowlist syntax works well. For example, `'src'` lets an author include a frame only while its origin matches the frame's `src` attribute. But permission policy decides whether a Document can use a feature based on all ancestors in its chain allowing it while for frame selection, only the frames between the query root and that frame should decide. For example, a portal may embed a self-contained app without wanting that app to represent the portal, but the portal's own agent should still be able to query the app's full tool-set. With permission policy, the portal excluding the app from its own tool-set would also exclude the app's frames from the app's composed tool-set.

Conceptually the user-agent is querying the composed tool-set for the top-level browsing context but it can still use permission policy because traversal always starts at the top-level document.

**Shadow DOM**
Iframes inside a shadow root are removed from the window's frame list (`window.length`, `window[i]`). This is for encapsulation, not security. If subtree discovery included tools from these frames, it would add a new way for the light DOM to see inside a shadow tree.

Each tool also carries a reference to its source `WindowProxy`. That reference isn't always cross-origin, so for a same-origin frame it gives direct access to the DOM the shadow root was hiding. The closest existing pattern is `postMessage`: a frame inside a shadow tree can already pass its window reference out, but only on purpose. We should likely follow the same principle: by default, subtree discovery doesn't cross shadow boundaries but content inside the shadow tree can opt in to exposing its frames' tools upward.
