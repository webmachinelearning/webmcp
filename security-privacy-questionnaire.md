# [Self-Review Questionnaire: Security and Privacy](https://w3c.github.io/security-questionnaire/)

1.  What information does this feature expose, and for what purposes?

WebMCP exposes author-defined tool metadata and tool return values to UA. It does not expose new information about the user or their environment to origins.

Cross-origin iframes may discover these tools, only if explicitly allowed by tool authors via `exposedTo`. 

1.  Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?

The API surface itself exposes only the information necessary to implement the intended functionality. 

1.  Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?

No, the API itself does not expose PII. 

We do note that there is a novel challenge that implementors of agents calling these tools should be aware of: malicious tools can ask for a non-minimal set of PII to achieve their intended functionality, which can cause privacy leakage. See [Privacy Leakage through Over Parameterization](https://w3c.github.io/webmcp/#privacy-leakage-through-over-parameterization) for more details. WebMCP did not increase the vector of attack (compared to a malicious tool asking for more PII in a non-WebMCP context), but they should be aware that this risk exists. 

1.  How do the features in your specification deal with sensitive information?

WebMCP itself is not a source of sensitive information. Tools may wrap sensitive information, but that is not a WebMCP-specific issue. We discuss the risk of sites exposing sensitive content in tools in [Tool Implementation as Attack Targets](https://webmachinelearning.github.io/webmcp/#tool-implementation-targets).

1.  Does data exposed by your specification carry related but distinct information that may not be obvious to users?

No, the API surface itself does not expose related but distinct information.

1.  Do the features in your specification introduce state that persists across browsing sessions?

No, currently WebMCP tools are tied to the document's lifetime. There's discussions of options to help persist tools across navigation, but that is not currently specified.

1.  Do the features in your specification expose information about the underlying platform to origins?

No.

1.  Does this specification allow an origin to send data to the underlying platform?

Tool inputs and outputs flow between the page and the authorized agent, which includes built-in agent in the UA. The data is structured (JSON-serializable values conforming to declared schemas).

1.  Do features in this specification enable access to device sensors?

No.

1.  Do features in this specification enable new script execution/loading mechanisms?

No. Tool `execute` callbacks are ordinary JavaScript invoked in the registering document's existing realm.

1.  Do features in this specification allow an origin to access other devices?

No.

1.  Do features in this specification allow an origin some measure of control over a user agent's native UI?

No direct control at the moment. There is discussion of `requestUserInput` in issue [#165](https://github.com/webmachinelearning/webmcp/issues/165).

1.  What temporary identifiers do the features in this specification create or expose to the web?

None.

1.  How does this specification distinguish between behavior in first-party and third-party contexts?

Feature is gated by  Permissions Policy `tools`. The feature is allowed in the top-level document and same-origin descendants by default; Permission policy can be used to allow it in cross-origin iframes, or to disallow it in same-origin frames.

Additionally, tools themselves can specify `exposedTo` to control which origins (or `native-agents`, name to be bikeshed as per [#179](https://github.com/webmachinelearning/webmcp/pull/179)) can discover them.

1.  How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

We do not anticipate any differences.

1.  Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

[Security and Privacy Considerations](https://webmachinelearning.github.io/webmcp/#security-and-privacy-considerations).

1.  Do features in your specification enable origins to downgrade default security protections?

No.

1.  What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?

A BFCached document's registered tools remain in memory. While the document is non-fully-active, agents should not invoke its tools or deliver events to it. On restoration, registered tools become available again. 

1.  What happens when a document that uses your feature gets disconnected?

A disconnected document's tools are no longer discoverable or invokable by agents. Pending tool invocations associated with the document are abandoned.

1.  Does your spec define when and how new kinds of errors should be raised?

Yes. `registerTool()` throws `InvalidStateError` for inactive documents, duplicate names, or invalid name/description.

`NotAllowedError` when the "tools" Permissions Policy is disallowed; `SecurityError` for non-trustworthy exposedTo rigins; and `TypeError` when inputSchema serialization fails.o

These errors do not leak new information to the origins.

1.  Does your feature allow sites to learn about the user's use of assistive technology?
 
No.

1.  What should this questionnaire have asked?

None that I can think of.