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
developers to expose this kind of site functionality while still using the semantic web, we
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
form, the browser brings the submit button into focus, and the agent should then tell the user to
check the form contents, and submit it manually.

When forms with these attributes are inserted, removed, or these attributes are updated, the form
creates a new declarative WebMCP tool whose input schema is generated according to
[Input schema synthesis](#input-schema-synthesis).

### Name and description

The [`name`](https://html.spec.whatwg.org/C#attr-fe-name) attribute on form control elements
supplies the name of each "property" in the input schema generated for a declarative tool.

Since there's no pre-existing description attribute we can use, we introduce the
`toolparamdescription` attribute for form control elements, which contributes the
[description](https://json-schema.org/draft/2020-12/json-schema-validation#name-title-and-description)
of each "property" in the input schema generated for a declarative tool.

With this, the following imperative structure:

```js
window.navigator.modelContext.registerTool({
  name: "search-cars",
  description: "Perform a car make/model search",
  inputSchema: {
    type: "object",
    properties: {
      make: { type: "string", description: "The vehicle's make (e.g., BMW, Ford)" },
      model: { type: "string", description: "The vehicle's model (e.g., 330i, F-150)" },
    },
    required: ["make", "model"]
  },
  execute({make, model}, agent) { ... }
});
```

... is equivalent to the following declarative form:

```html
<form toolname="search-cars" tooldescription="Perform a car make/model search" [...]>
 <input type=text name="make" toolparamdescription="The vehicle's make (i.e., BMW, Ford)" required>
 <input type=text name="model" toolparamdescription="The vehicle's model (i.e., 330i, F-150)" required>
 <button type=submit>Search</button>
</form>
```

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

This topic is currently under debate; see https://github.com/webmachinelearning/webmcp/issues/135.

<details>
<summary>Click to read the `application/ld+json` proposal before the above issue was filed</summary>

When a form element performs a navigation, the first `<script type=application/ld+json>` tag on the
target page is used as the cross-document tool's "response" that gets sent to the model.

When no such a tag is present, probably we'll decide that the page's entire contents is sent to the
model as the response, since that's an accurate semantic representation of the result of the tool.
However, this is technically TBD at the moment.

When the form element does *NOT* perform a navigation, JavaScript can hand-craft the response to the
agent via the `SubmitEvent#respondWith()` method described below.
</details>

### Pseudo-classes

Authors might want a way to bring to the user's attention or otherwise highlight a declarative
WebMCP form that was filled out by the agent, and is waiting for the user to check the form and
submit it. (This is essentially only relevant for forms without the `toolautosubmit` attribute). To
support this, we introduce the CSS pseudo-classes `:tool-form-active` and `:tool-submit-active`.

The `:tool-form-active` pseudo-class matches `<form>` elements whose declarative tool is "running".
The exact definition of this will be clarified in the specification, but in short, a declarative
tool is considered "running" starting when the form is being filled out with agent output, until one
of the following:

 - The form is [reset](https://html.spec.whatwg.org/C#concept-form-reset) or removed from the DOM
 - The Promise returned from `SubmitEvent#respondWith()` resolves with a tool output
 - The form's `toolname` or `tooldescription` attributes are modified, added, or removed
 - The form is automatically submitted with the agent output, due to the `toolautosubmit` attribute

The `:tool-submit-active` pseudo-class matches the submit button of a `:tool-form-active` form
element.

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

We introduce these events that get fired at the `ModelContext` object when a WebMCP tool is run, and when
its invocation is canceled.

The `toolactivated` event gives the developer a hook to perform any actions, such as bringing the
form to the user's attention, once a declarative tool is filled out but before it is submitted.
(This presumes the absence of the `toolautosubmit` attribute). This event can be seen as the
JavaScript equivalent of the [`:tool-form-active` pseudo-class](#pseudo-classes).

When the agent cancels a tool call (perhaps because a user has instigated another turn of the
conversation, obviating the need for the pending tool), the `toolcanceled` event is fired. Note that
this event does not fire when the site itself cancels the tool, due to removing the form element or
changing its name or description.

Some open questions:

> [!WARNING]
> Should these events fire for imperative tool call invocations as well? Chromium
> [seems to do
> that](https://source.chromium.org/chromium/chromium/src/+/main:third_party/blink/renderer/core/script_tools/model_context.cc;l=265-274;drc=2af6413cf36d701fdaffb09188f2ab2a5be37f4f).

> [!WARNING]
> For declarative, should they be fired at `Window` or at the `<form>` that registered the tool in
> the first place, and bubble up to the document that way? See
> https://github.com/webmachinelearning/webmcp/issues/126.

## Integration with other imperative API bits

It's an open question as to whether [an
`outputSchema`](https://github.com/webmachinelearning/webmcp/issues/9) makes sense for declarative
WebMCP tools, and therefore if the `agentResponse` Promise passed to `SubmitEvent#respondWith()`
must resolve to an object conforming to such schema.

It is TBD how *declarative* WebMCP tools will be exposed to any interface that exposes a site's
tools to JavaScript. See https://github.com/webmachinelearning/webmcp/issues/51 for context. Should
a declarative WebMCP tool be able to be invoked from such an interface, should it exist in the
future? Almost certainly, yes. But details are TBD.
