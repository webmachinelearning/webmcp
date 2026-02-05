# WebMCP declarative API

See discussion in https://github.com/webmachinelearning/webmcp/issues/22 that led to the creation of
this proposal.

## Motivation

WebMCP lets developers expose intricate functionality backed by a website's JavaScript functions to
an agent as "tools", effectively turning the site into an "MCP server". Agents can see the list of
tools a site offers paired with natural language descriptions of what the tools do, and invoke them
with structured data.

With WebMCP, agents can perform complex actions like booking a flight or reserving a table by
hooking into a site's own code designed to perform those actions, instead of the agent having to
figure it out manually through a brittle series of screen shots, scrolls, and out-of-date screen
reads.

However, not all site functionality is exposed via JavaScript functions, and features that *are*
take some effort to rewrite with an agent invoker in mind. Much of a site's functionality is
provided via semantic HTML elements like `<form>`, and its various inputs. To **make it easier** for
developers to expose this kind of site functionality while still using thte semantic web, we
propose:

1. New attributes that augment `<form>`s and [form-associated
   elements](https://html.spec.whatwg.org/#form-associated-element), that expose these as WebMCP
   tools to agents.
2. Algorithms that deterministically "compile" a form and its associated inputs down to a WebMCP
   "input schema", so that the agent knows how to fill out the form and submit it.
3. Two ways of getting a form response back to the agent that invoked the form tool:
    1. `SubmitEvent#respondWith()`, which lets JavaScript on the page override the default form
       action, and pipe a response back to the agent without navigating the page.
    2. Extracting `<script type="application/json-ld">` tags on the page that the form navigated to,
       and using that structured data as a response to the form.

## Form attributes

```html
<form
  toolname="Search flights"
  tooldescription="This form searches flights and displays [...]"
  toolautosubmit>
```

The `toolname` attribute is analogous to the imperative API's
[`ModelContextTool#name`](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttool-name),
while `tooldescription` is analogous to
[`ModelContextTool#description`](https://webmachinelearning.github.io/webmcp/#dom-modelcontexttool-description).

The `toolautosubmit` [boolean attribute](https://html.spec.whatwg.org/C#boolean-attribute), lets the
agent submit the form on the user's behalf after filling it out, without requiring the user to check
it manually before submitting. If this attribute is missing when the agent finishes filling out the
form, the browser brings the submit attribute into focus, and the agent should then tell the user to
check the form contents, and submit it manually.

When forms with these attributes are inserted, removed, or these attributes are updated, the form
creates a new declarative WebMCP tool whose input schema is generated according to
[Input schema synthesis](#input-schema-synthesis).

TODO(domfarolino): Describe the `toolparamname` and `toolparamdescription` attributes, and how they
are processed on form-associated elements.

## Processing model

### Changes to form reset

When a form is [reset](https://html.spec.whatwg.org/C#concept-form-reset) **OR** its tool
declaration changes (as a result of `toolname` attribute modifications, for example), then any
in-flight invocation of the tool will be cancelled, and the agent will be notified of this
cancellation.

### Input schema synthesis

TODO: The exact algorithms reducing a form, its form-associated elements, and *their* attributes
like [`step`](https://html.spec.whatwg.org/C#the-step-attribute) and
[`min`](https://html.spec.whatwg.org/C#attr-input-min) is TBD. We need to concretely specify how
various form-associated elements like `<input>` and `<select>` reduce to a JSON Schema that includes
`anyOf`, `oneOf`, and `maximum`/`mininum` declarations.

Chromium is implementing a loose version of this and will conduct testing/trials to see if what
we've come up with should be supported by the community as a general approach.

### Getting the form response to the agent

TODO: Mention application/json-ld responses, and so on.

### Events

**Additions to `SubmitEvent`**

The `SubmitEvent` interface gets two new members, `agentInvoked` to let `submit` event handler react
to agent-invoked form submissions, and the `respondWith()` method.

This method takes a `Promise<any>` that resolves to the response that the agent will consume. This
method is used to override the default behavior of the form submission; the form's `action` will NOT
navigate, and the `preventDefault()` must be called before this method is called.

```js
[Exposed=Window]
interface SubmitEvent : Event {
  // ...
  readonly attribute boolean agentInvoked;
  undefined respondWith(Promise<any> agentResponse);
};
```

**`toolactivated` and `toolcanceled` events

TODO: Fill this out.

## Integration with other imperative API bits

It's an open question as to whether [an
`outputSchema`](https://github.com/webmachinelearning/webmcp/issues/9) makes sense for declarative
WebMCP tools, and therefore if the `agentResponse` Promise passed to `SubmitEvent#respondWith()`
must resolve to an object conforming to such schema.

It is TBD how *declarative* WebMCP tools will be exposed to any interface that exposes a site's
tools to JavaScript. See https://github.com/webmachinelearning/webmcp/issues/51 for context. Should
a declarative WebMCP tool be able to be invoked from such an interface, should it exist in the
future?
