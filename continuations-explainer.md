# Continuation Tokens

[*mark a. foltz*](mailto:mfoltz@google.com)

# Overview

[WebMCP](https://webmachinelearning.github.io/webmcp/) allows sites to declare
imperative script tools to be invoked by agents. To complete a task, control
flow is typically exchanged between the site and the agent.  The agent initiates
a tool call on the site, the site executes the tool with site-defined
Javascript, and returns output to the agent, which consumes the output and plans
the next action.

However, tool calls are restricted to execute in a single document, and a single
Promise per tool call.  This leads to unfortunate limitations:

* If a site executes a server request that navigates the document, the original
  tool request is ended and the agent does not receive a success signal or tool
  output.
* If a tool call wants to append further output, for example to list a following
  page of results, it has no way of doing so.

We propose a *Tool Continuation* as a way for the tool author to request that
the agent invoke the tool at a future time on a related document.

**This introduces a higher level of abstraction for the agent.** Instead of
acting on the site with a fragmented collection of tools partitioned according
to the site structure (pages/frames), it can act on the site as a cohesive
application with tools matching end-user use cases (like "checkout").  Tool
continuations allow these use cases to be implemented across documents and
frames without the agent needing to reverse engineer the site's internal
structure.

# Use Cases (In Scope)

Let's consider a clothing e-commerce site [dresswear.com](http://dresswear.com)
that allows the user to browse items of clothing, add them to a cart and check
out.

## Cross-document tool invocation

After completing add-to-cart, the user wants to check out. The site offers a
tool to do so:

```js
checkout(shipping_address, billing_address, card_details)
```

However, the implementation of this tool is actually spread across three
distinct pages (`/shipping`, `/billing`, `/checkout`). For the tool to complete,
the site needs to do a hard navigation across these three pages before finally
submitting the checkout request to the server.

## Cross-frame tool invocation

[dresswear.com](http://dresswear.com) has a feature where the user can virtually
try on an item of clothing while on an item page.  Because the try-on
application is expensive to load, it is created on-demand in a same-origin
`<iframe>` on the page.

The main page exposes a tool for the try-on feature:

```js
try_on(item_id)
```

However, the virtual try-on can only be fulfilled in the `<iframe>`, which is
only created on demand, so the tool can't be implemented entirely in the main
page.

# Proposal

## Token Creation

We introduce a `ModelContextToolInvocation` object that is passed to a tool's
execute callback via the `ToolExecuteCallbackOptions`.  This object allows the
execute callback to access context about the current tool execution and request
tokens.

```js

dictionary ToolExecuteCallbackOptions {
  required AbortSignal signal;
  required ModelContextToolInvocation invocation;
};

[Exposed=Window, SecureContext]
interface ModelContextToolInvocation {
  // Requests a unique continuation token for this invocation.  Can be redeemed
  // to continue tool execution on another document.
  Promise<DOMString> requestToken();
}
```
Notes:

* `requestToken()` returns a DOMString to allow simple storage and transmission
  of the token.
* `requestToken()` returns a `Promise` to allow the token to be successfully
  registered with agents and bound to the correct origin before returning, which
  likely requires cross-process IPC.
* `requestToken()` rejects if the current task is not executing the callback for a tool.

## Token Redemption

Sites redeem a token by calling `resumeTool()`, which registers a request to
resume tool execution.  This can happen in a document other than the one that
generated the token.  The site is responsible for passing this token (along with
any other additional state required for the tool call to resume) to the new
document before calling `resumeTool()`.


```js
[Exposed=Window, SecureContext]
partial interface ModelContext {
  Promise<undefined> resumeTool(DOMString token, ToolExecuteCallback callback);
};
```

**This action has the following side effects:**

* If the token is accepted for redemption, then the Promise resolves
  successfully.
* The browser will call `callback` again shortly in the document that requested
  it.
* The `callback` will be passed the input object that was passed to the original
  tool call and a new `ToolExecuteCallbackOptions.`

**The following conditions apply on token redemption:**

* The token may only be redeemed once.  
* Calls to resumeTool() while the initial tool is executing are allowed, but any
  invocation of the callback will occur after the initial tool call is complete.
* Redemption fails if the document is not same-origin and part of the same
  browsing context. same browsing context group.
* If the original tool call is canceled by the caller (via AbortSignal for
  web-platform callers, or internally by a built-in agent)`,`then the token is
  revoked.
* The browser may limit the lifetime of a token, i.e. it cannot be redeemed at
  an arbitrary time later.  The initial limit will be set by looking at agent
  trajectory data, and will probably fall around 60 seconds.
* The token is implicitly canceled by the browser if the agent abandoned the
  task, received different user instructions, was blocked from accessing the
  site, etc.

If any of these conditions are not met, then `resumeTool()` rejects.

## Agent Behaviors

When the tool requests a token, this is a signal to the agent that the tool has
not completed execution.  Any output from the tool should be considered as
partial output, and the browser should expect additional output before the tool
invocation is completed.

The browser will wait until all tokens are redeemed in the chain, all tool calls
have resolved, and show the model the final output of all tool calls.  This
blocks the agent until the completion of a chain of continued tool calls.

# Examples

## Cross-document tool invocation

```js
// /billing.html
<script>
modelContext.registerTool({
   name: "checkout",
   inputSchema: {
     type: "object",
     properties: {
       billing_address: { type: "string" },
       shipping_address: { type: "string" }
     }, required: ["billing_address", "shipping_address"]
   },
   execute: async (input, options) => {
    // We'll need to continue on the next page for billing info.
    const token = await options.invocation.requestToken();
    // Handle billing_address (submit to server, etc.)
    await submitBillingAddress(input.billing_address);
    // Navigate to /shipping.html
    window.location.href = '/shipping.html?checkout_token=${token}';
  }
)};
</script>
```

```js
// /shipping.html
<script>
// Request resumption of the checkout tool.
modelContext.resumeTool(
  new URLSearchParams(window.location.search).get('token'),
  async (input, options) => {
    // We'll need to continue again on the next page for confirmation.
    const token = await options.invocation.requestToken();
    // Handle shipping_address (submit to server, etc.)
    await submitShippingAddress(input.shipping_address);
    // Navigate to /confirmation.html
    window.location.href = '/confirmation.html?token=${token}';
  }
);
</script>
```
# Alternatives Considered

*Cross-navigation Promises.* We could invent a type of Promise (or some other
script object) that survives navigation and is passed from one tool to
another.  This seems technically very complicated as that object could have
references into multiple script states and DOM trees.

*Tool Chaining.* We could have a tool return a call to another tool to be
invoked as a subsequent action by the agent.  This is a variant, but exposes
the application structure to the agent, which would need to chain multiple
tools to accomplish the intended task.  A goal is to minimize exposure of the
site structure for tools that cross document boundaries.

# Document Lifecycle Considerations

Tokens cannot be requested or redeemed on inactive documents or frames.

_Question:_ If a tool requests a token, and at a later time the tool call does
not complete (because the document becomes inactive or unloaded), then does the
token remain valid?

_Question:_ If a document requests a token then performs a cross-origin
navigation, does the token remain valid?  If so, how many navigations are
allowed before the token expires?

# Security and Privacy Considerations

The token should be unguessable.  Tokens should be created and tracked in a
trusted process to ensure access by permitted same-origin document(s) and
one-time redemption.

Its bearer will get access to the knowledge that a tool was called and the input
to that tool, which may contain private user information.  For this reason, the
token is intended to be origin-bound.  Leaking it to untrusted origins is not a
risk as it cannot be redeemed on them.

_Question:_ Do we allow tokens to be redeemed by a same-origin \<iframe\>
embedded in a cross-origin main frame?  This depends on how we view tokens
related to storage partitioning.

# Accessibility Considerations

This proposal does not create new accessibility considerations.

# Future Extensions (Out of Scope for Initial Proposal)

- Partial function calling
- Cross-origin tool calling in frames
- Document-to-worker tool calling (and vice versa)
- Document-to-server handoff

## Browsing context groups

Tools or agents may open additional browsing contexts (tabs, popup windows), and
it may be advantageous to allow tool calls to span these additional documents.
The scoping of tokens to browsing contexts can be relaxed in the future if this
does not introduce any implementation or agent compatibility issues.

## Declarative Forms

If the tool is declared through a `<form>` element, we can configure the form to
generate a token that can be redeemed in a post-submit document.  This could be
done through an additional `<form>` attribute or `<input>` type.
