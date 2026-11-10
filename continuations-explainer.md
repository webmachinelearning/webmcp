# Continuation Tokens

[*mark a. foltz*](mailto:mfoltz@google.com), grateful for input from Dominic
Farolino and several others.

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

However, the implementation of this tool is actually spread across two
distinct pages (`/shipping` and `/billing`). For the tool to complete, the site
needs to do a hard navigation from `/shipping` to `/billing` before finally
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
partial dictionary ToolExecuteCallbackOptions {
  ModelContextToolInvocation invocation;
};

[Exposed=Window, SecureContext]
interface ModelContextToolInvocation {
  // Requests a unique continuation token for this invocation.  Can be redeemed
  // to continue tool execution on another document.
  Promise<DOMString> requestToken();
}
```
Notes:

* `invocation` is passed to the execute callback of imperative tools registered
  with `registerTool()`.
* `requestToken()` returns a DOMString to allow simple storage and transmission
  of the token.  The token is a randomly generated, lowercase
  [RFC 9562](https://www.rfc-editor.org/rfc/rfc9562) version 4 UUID, for
  example `"6f0c8f9e-3b1a-4c2d-9e8f-0a1b2c3d4e5f"`.
* `requestToken()` returns a `Promise` to allow the token to be successfully
  registered with agents and bound to the correct origin before returning.
* A tool call can request at most one token.  `requestToken()` rejects with an
  `InvalidStateError` if:
  * the document is not
    [fully active](https://html.spec.whatwg.org/multipage/document-sequences.html#fully-active),
  * the tool Promise associated with the invocation has settled, or the tool
    call was canceled,
  * a token was already requested for the tool call, or
  * it is called from a callback passed to `resumeTool()`.

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
* The browser will then call `callback` shortly in the document that called
  `resumeTool()`.  The callback is invoked after the Promise returned by
  `resumeTool()` has resolved.
* The `callback` will be passed the input object that was passed to the original
  tool call and a new `ToolExecuteCallbackOptions`, with a new `signal` and
  `invocation`.

**The following conditions apply on token redemption:**

* The token may only be redeemed once.
* The redeeming document must be same-origin with the document that requested
  the token, and must be in the same tab.  This may be the same document, a
  different frame in the same page (for example, an `<iframe>`), or a document
  loaded by a later navigation in the tab.  Tokens can't be redeemed in other
  tabs or windows (including popups).
* The redeeming document must be fully active, and must be allowed to use the
  `"tools"` [permissions policy](https://www.w3.org/TR/permissions-policy/)
  feature.
* The token must be redeemed within a limited time after it was requested.
  Chromium currently allows 30 seconds.  There is no limit on the number of
  navigations between requesting and redeeming the token.
* `resumeTool()` may be called before the original tool call's Promise settles.
  The callback does not wait for that Promise to settle.

If any of these conditions are not met, then `resumeTool()` rejects:

* with an `InvalidStateError` if the document is not fully active,
* with a `NotAllowedError` if the document isn't allowed to use tools, and
* with an `InvalidAccessError` if the token is malformed, unknown, expired,
  already redeemed, or was requested by a different origin or in a different
  tab.

## Cancellation

The caller may cancel the tool call after a token was requested (via
`AbortSignal` for web-platform callers, or internally by a built-in agent, for
example when the agent abandons the task).

* If the token has not been redeemed, then it is **not** revoked.  It can still
  be redeemed until it expires, and the callback will run, but its result is
  discarded.

  TODO(mfoltz): Explain why cancellation does not revoke an unredeemed token.
* If the token was redeemed and the callback is executing, then the `signal`
  passed to the callback is aborted, and a `toolcancel` event is fired at
  `document.modelContext` in the redeeming document.

## Agent Behaviors

When the tool requests a token, this is a signal to the agent that the tool has
not completed execution.  The value returned by the original execute callback
(or its rejection) is ignored, and is not shown to the agent.

The browser waits until the token is redeemed and the callback passed to
`resumeTool()` settles, and shows the agent the output of that callback as the
result of the tool call.  This blocks the agent until the continued tool call
completes.  The tool call fails if:

* the callback passed to `resumeTool()` rejects,
* the document running that callback is unloaded before it settles, or
* the token expires before it is redeemed.

# Examples

## Cross-document tool invocation

```js
// /shipping.html
document.modelContext.registerTool({
  name: "checkout",
  description: "Checks out the items in the cart.",
  inputSchema: {
    type: "object",
    properties: {
      shipping_address: { type: "string" },
      billing_address: { type: "string" }
    }, required: ["shipping_address", "billing_address"]
  },
  execute: async (input, options) => {
    // We'll need to continue on the next page for billing info.
    const token = await options.invocation.requestToken();
    // Handle shipping_address (submit to server, etc.)
    await submitShippingAddress(input.shipping_address);
    // Navigate to /billing.html
    window.location.href = `/billing.html?token=${token}`;
  }
});
```

```js
// /billing.html
// Request resumption of the checkout tool.
document.modelContext.resumeTool(
  new URLSearchParams(window.location.search).get('token'),
  async (input, options) => {
    // Handle billing_address (submit to server, etc.)
    await submitBillingAddress(input.billing_address);
    const order = await placeOrder();
    // This is the result of the checkout tool call.
    return `Order ${order.id} placed.`;
  }
);
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

If the document that requested a token is unloaded before the original tool
call's Promise settles (for example, because the tool navigated it), then the
token remains valid.  This is the main use case for continuations.

If the document running the callback passed to `resumeTool()` is unloaded
before the callback settles, then the tool call fails.

If a document requests a token then performs a cross-origin navigation, the
token remains valid, and can be redeemed by a later document that is
same-origin with the document that requested it (for example, after returning
from a payment provider).  There is no limit on the number of navigations, only
on the time before the token is redeemed.

Unregistering a tool after it has requested a token does not affect the
continuation.

# Security and Privacy Considerations

The token is unguessable: it is a randomly generated version 4 UUID.  Tokens
are created and tracked in a trusted (browser) process, which enforces that
they are redeemed only once, only by same-origin documents in the same tab, and
only before they expire.

Its bearer will get access to the knowledge that a tool was called and the input
to that tool, which may contain private user information.  For this reason, the
token is intended to be origin-bound.  Leaking it to untrusted origins is not a
risk as it cannot be redeemed on them.

Tokens may be redeemed by a same-origin `<iframe>` embedded in a cross-origin
main frame, provided the `<iframe>` is allowed to use tools.  Redemption does
not check the top-level site.

# Accessibility Considerations

This proposal does not create new accessibility considerations.

# Future Extensions (Out of Scope for Initial Proposal)

- Partial function calling
- Cross-origin tool calling in frames
- Document-to-worker tool calling (and vice versa)
- Document-to-server handoff

## Chained continuations

A callback passed to `resumeTool()` can't request another token, so a tool call
can only be continued once.  Some tools span more than two documents, for
example a checkout flow across `/shipping`, `/billing` and `/confirmation`.  A
future extension could allow the callback to request a new token from its
`invocation`, to continue the tool call again on a later document.

## Browsing context groups

Tools or agents may open additional browsing contexts (tabs, popup windows), and
it may be advantageous to allow tool calls to span these additional documents.
The scoping of tokens to browsing contexts can be relaxed in the future if this
does not introduce any implementation or agent compatibility issues.

## Declarative Forms

If the tool is declared through a `<form>` element, we can configure the form to
generate a token that can be redeemed in a post-submit document.  This could be
done through an additional `<form>` attribute or `<input>` type.
