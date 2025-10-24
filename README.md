# WebMCP 🧪

_Enabling web apps to provide JavaScript-based tools that can be accessed by AI agents and assistive technologies to create collaborative, human-in-the-loop workflows._

> First published August 13, 2025
>
> Brandon Walderman <code>&lt;brwalder@microsoft.com&gt;</code><br>
> Leo Lee <code>&lt;leo.lee@microsoft.com&gt;</code><br>
> Andrew Nolan <code>&lt;annolan@microsoft.com&gt;</code><br>
> David Bokan <code>&lt;bokan@google.com&gt;</code><br>
> Khushal Sagar <code>&lt;khushalsagar@google.com&gt;</code><br>
> Hannah Van Opstal <code>&lt;hvanopstal@google.com&gt;</code>

## TL;DR

We propose a new JavaScript interface that allows web developers to expose their web application functionality as "tools" - JavaScript functions with natural language descriptions and structured schemas that can be invoked by AI agents, browser assistants, and assistive technologies. Web pages that use WebMCP can be thought of as [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction) servers that implement tools in client-side script instead of on the backend. WebMCP enables collaborative workflows where users and agents work together within the same web interface, leveraging existing application logic while maintaining shared context and user control.

For the technical details of the proposal, code examples, API shape, etc. see [proposal.md](./docs/proposal.md).

## Terminology Used

###### Agent
An autonomous assistant that can understand a user's goals and take actions on the user's behalf to achieve them. Today,
these are typically implemented by large language model (LLM) based AI platforms, interacting with users via text-based
chat interfaces.

###### Browser's Agent
An autonomous assistant as described above but provided by or through the browser. This could be an agent built directly
into the browser or hosted by it, for example, via an extension or plug-in.

###### AI Platform
Providers of agentic assistants such as OpenAI's ChatGPT, Anthropic's Claude, or Google's Gemini.

###### Backend Integration
A form of API integration between an AI platform and a third-party service in which the AI platform can talk directly to
the service's backend servers without a UI or running code in the client. For example, the AI platform communicating with
an MCP server provided by the service.

###### Actuation
An agent interacting with a web page by simulating user input such as clicking, scrolling, typing, etc.

## Background and Motivation

The web platform's ubiquity and popularity have made it the world's gateway to information and capabilities. Its ability to support complex, interactive applications beyond static content, has empowered developers to build rich user experiences and applications. These user experiences rely on visual layouts, mouse and touch interactions, and visual cues to communicate functionality and state.

As AI agents become more prevalent, the potential for even greater user value is within reach. AI platforms such as Copilot, ChatGPT, Claude, and Gemini are increasingly able to interact with external services to perform actions such as checking local weather, finding flight and hotel information, and providing driving directions. These functions are provided by external services that extend the AI model’s capabilities. These extensions, or “tools”, can be used by an AI to provide domain-specific functionality that the AI cannot achieve on its own. Existing tools integrate with each AI platform via bespoke “integrations” - each service registers itself with the chosen platform(s) and the platform communicates with the service via an API (MCP, OpenAPI, etc). In this document, we call this style of tool a “backend integration”; users make use of the tools/services by chatting with an AI, the AI platform communicates with the service on the user's behalf.

Much of the challenges faced by assistive technologies also apply to AI agents that struggle to navigate existing human-first interfaces when agent-first "tools" are not available. Even when agents succeed, simple operations often require multiple steps and can be slow or unreliable.

The web needs web developer involvement to thrive. What if web developers could easily provide their site's capabilities to the agentic web to engage with their users? We propose WebMCP, a JavaScript API that allows developers to define tools for their webpage. These tools allow for code reuse with frontend code, maintain a single interface for users and agents, and simplify auth and state where users and agents are interacting in the same user interface. Such an API would also be a boon for accessibility tools, enabling them to offer users higher-level actions to perform on a page. This would mark a significant step forward in making the web more inclusive and actionable for everyone.

AI agents can integrate in the backend via protocols like MCP in order to fulfill a user's task. For a web developer to expose their site's functionality this way, they need to write a server, usually in Python or NodeJS, instead of frontend JS which may be more familiar.

There are several advantages to using the web to connect agents to services:

* **Businesses near-universally already offer their services via the web.**
    
    WebMCP allows them to leverage their existing business logic and UI, providing a quick, simple, and incremental
    way to integrate with agents. They don't have to re-architect their product to fit the API shape of a given agent.
    This is especially true when the logic is already heavily client-side.


* **Enables visually rich, cooperative interplay between a user, web page, and agent with shared context.**

    Users often start with a vague goal which is refined over time. Consider a user browsing for a high-value purchase.
    The user may prefer to start their journey on a specific page, ask their agent to perform some of the more tedious
    actions ("find me some options for a dress that's appropriate for a summer wedding, preferably red or orange, short
    or no sleeves and no embellishments"), and then take back over to browse among the agent-selected options.

* **Allows authors to serve humans and agents from one source**

    The human-use web is not going away. Integrating agents into it prevents fragmentation of their service and allows
    them to keep ownership of their interface, branding and connection with their users.

WebMCP is a proposal for a web API that enables web pages to provide agent-specific paths in their UI. With WebMCP, agent-service interaction takes place _via app-controlled UI_, providing a shared context available to app, agent, and user. In contrast to backend integrations, WebMCP tools are available to an agent only once it has loaded a page and they execute on the client. Page content and actuation remain available to the agent (and the user) but the agent also has access to tools which it can use to achieve its goal more directly.

![A diagram showing an agent communicating with a third-party service via WebMCP running in a live web page](./content/explainer_webmcp.png)

In contrast, in a backend integration, the agent-service interaction takes place directly, without an associated UI. If
a UI is required it must be provided by the agent itself or somehow connected to an existing UI manually:

![A diagram showing an agent communicating with a third-party service directl via MCP](./content/explainer_mcp.png)

## Goals

- **Enable human-in-the-loop workflows**: Support cooperative scenarios where users work directly through delegating tasks to AI agents or assistive technologies while maintaining visibility and control over the web page(s).
- **Simplify AI agent integration**: Enable AI agents to be more reliable and helpful by interacting with web sites through well-defined JavaScript tools instead of through UI actuation.
- **Minimize developer burden**: Any task that a user can accomplish through a page's UI can be made into a tool by re-using much of the page's existing JavaScript code.
- **Improve accessibility**: Provide a standardized way for assistive technologies to access web application functionality beyond what's available through traditional accessibility trees which are not widely implemented.

## Non-Goals

- **Headless browsing scenarios**: While it may be possible to use this API for headless or server-to-server interactions where no human is present to observe progress, this is not a current goal. Headless scenarios create many questions like the launching of browsers and profile considerations.
- **Autonomous agent workflows**: The API is not intended for fully autonomous agents operating without human oversight, or where a browser UI is not required. This task is likely better suited to existing protocols like [A2A](https://a2aproject.github.io/A2A/latest/).
- **Replacement of backend integrations**: WebMCP works with existing protocols like MCP and is not a replacement of existing protocols.
- **Replace human interfaces**: The human web interface remains primary; agent tools augment rather than replace user interaction.
- **Enable / influence discoverability of sites to agents**

## Use Cases

The use cases for WebMCP are ones in which the user is collaborating with the agent, rather than completely
delegating their goal to it. They can also be helpful where interfaces are highly specific or complicated.

### Example - Creative

_Jen wants to create an invitation to her upcoming yard sale so she uses her browser to navigate to
`http://easely.example`, her favorite graphic design platform. However, she's rather new to it and sometimes struggles
to find all the functionality needed for her task in the app's extensive menus. She creates a "yard sale flyer" design
and opens up a "templates" panel to look for a premade design she likes. There's so many templates and she's not sure
which to choose from so she asks her browser agent for help._

**Jen**: Show me templates that are spring themed and that prominently feature the date and time. They should be on a
white background so I don't have to print in color.

_The current document has registered a WebMCP tool that the agent notices may be relevant to this query:_

```js
/**
 * Filters the list of templates based on a description.
 *
 * description - A visual description of the types of templates to show, in natural language (English).
 */
 filterTemplates(description)
```

_The agent invokes the tool: `filterTemplate("spring themed, date and time displayed prominently, white background")`.
The UI updates to show a filtered list matching this description._

**Agent**: Ok, the remaining templates should now match your description.

_Jen picks a template and gets to work._

_The agent notices a new tool was registered when the design was loaded:_

```js
/**
 * Makes changes to the current design based on instructions. Possible actions include modifications to text
 * and font; insertion, deletion, transformation of images; placement and scale of elements. The instructions
 * should be limited a single task. Here are some examples:

 * editDesign("Change the title's font color to red");
 * editDesign("Rotate each picture in the background a bit to give the design a less symmetrical feel");
 * editDesign("Add a text field at the bottom of the design that reads 'example text'");
 *
 * instructions - A description of how the design should be changed, in natural language (English).
 */
 editDesign(instructions)
```

_With all the context of Jen's prompts, page state, and this editDesign tool, the agent is able to make helpful
suggestions on next steps:_

**Agent**: Would you like me to make the time/date font larger?

**Jen**: Sure. Could you also swap out the clipart for something more yard-sale themed?

**Agent**: Sure, let me do that for you.

**Jen**: Please fill in the time and place using my home address. The time should be in my e-mail in a message from my
husband.

**Agent**: Ok, I've found it - I'll fill in the flyer with Aug 5-8, 2025 from 10am-3pm | 123 Queen Street West.

_Jen is almost happy with the current design but think the heading could be better_

**Jen**: Help me come up with a more attention grabbing headline for the call to action and title.

**Agent**: Of course! Here are some more attention-grabbing headlines for your yard sale flyer, broken down by title and
call to action:

To Create Excitement:
 * Yard Sale Extravaganza!
 * The Ultimate Clear-Out Sale
 * Mega Garage & Yard Sale

...

**Jen**: Lets use "Yard Sale Extravaganza!" as the title. Create copies of this page with each of the call to action
suggestions.

_The agent takes this action using a sequence of tool calls which might look something like:_

* `EditDesign("Change the title text to 'Yard Sale Extravaganza!'")`
* `EditDesign("Change the call-to-action text to 'The hunt is on!'")`
* `AddPage("DUPLICATE")`
* `EditDesign("Change the call-to-action text to 'Ready, set, shop!'")`
* `AddPage("DUPLICATE")`
* `EditDesign("Change the call-to-action text to 'Come for the bargains, stay for the cookies'")`
  
_Jen now has 3 versions of the same yard sale flyer. Easely implements these WebMCP tools using AI-based techniques on
their backend to allow a natural language interface. Additionally, the UI presents these changes to Jen as an easily
reversible batch of "uncommitted" changes, allowing her to easily review the agent's actions and make changes or undo as
necessary. While the site could also implement a chat interface to expose this functionality with their own agent, the
browser's agent provides a seamless journey by using tools across multiple sites/services. For example, pulling up 
information from the user's email service._

**Agent**: Done! I've created three variations of the original design, each with a unique call to action. 

_Jen is now happy with these flyers. Normally she'd print to PDF and then take the file to a print shop. However, Easely
has a new print service that Jen doesn't know about and doesn't notice in the UI. However, the agent knows the page has
an `orderPrints` tool:_

```js
/**
 * Orders the current design for printing and shiping to the user.
 *
 * copies - A number between 0 and 1000 indicating how many copies of the design to print. Required.
 * page_size - The paper type to use. Available options are [Legal, Letter, A4, A5]. Default is "Letter".
 * page_finish - What kind of paper finish to use. Available options are [Regular, Glossy Photo, Matte Photo].
 *               Default is "Regular"
 */
orderPrints(copies, page_size, page_finish);
```

_The agent understands the user's intent and so surfaces a small chip in it's UI:_

**Agent**: `<Done? Click to Order Prints>`

_Jen is delighted she saved a trip to the store and clicks the button_.

**Agent**: How many copies would you like? I'll request 8.5x11 sized regular paper but there are other options available.

**Jen**: Please print 10 copies.

**Agent**: Done! The order is ready for your review.

_The site navigates to the checkout page where Jen can review the order and click the checkout button to confirm._


### Example - Shopping

Shopping often includes exploration, browsing, and reviewing. The agent can help and remove the tedious parts but the
user wants to be involved.

_Maya has a few favorite clothing vendors but doesn't get formal clothes very often so she opens her AI assistant_

**Maya**: Suggest a few formal or semi-formal dress places where I can shop for my friend’s wedding. Please make sure
they are ecofriendly and not too expensive.

**Agent**: Sure, here's a few suggestions

1. Elara - uses innovative recycled and plant-based fabrics to minimize waste for a modern, chic wardrobe that allows
  you to embrace the latest trends, responsibly. Price Range: Dresses typically \$90 - \$220 CAD

2. Linden Harbour - creates wonderfully crafted womenswear designed to last a lifetime. Carries everything from classic
shirt dresses to elegant knitwear, meticulously made from the finest organic cotton, linen, and responsibly sourced
wool. Price Range: Dresses typically \$250 - \$550 CAD

3. Wildebloom - Flowing dresses brought to life through artisanal collaborations, using natural plant dyes and panels of
  upcycled vintage textiles.  Price Range: Dresses typically \$180 - \$450 CAD

**Maya**: Lets take a look at Wildebloom.

_The agent now opens Maya's web browser, which it is integrated with. i.e. the agent can observe and control the
browser. It navigates to the dresses page on `http://wildebloom.example/shop`_

**Agent**: Ok, here are the dresses that Wildebloom carries.

_Maya is immediately overwhelmed. There are so many options! Moreover, when she looks at filters she sees they're
quite limited with only colour and size as options._

**Maya**: Show me only dresses available in my size, and also show only the ones that would be appropriate for a
cocktail-attire wedding.

_The agent notices the dresses page registers several tools:_

```js
/*
 * Returns an array of product listings containing an id, detailed description, price, and photo of each
 * product
 *
 * size - optional - a number between 2 and 14 to filter the results by EU dress size
 * size - optional - a color from [Red, Blue, Green, Yellow, Black, White] to filter dresses by
 */
getDresses(size, color)

/*
 * Displays the given products to the user
 *
 * product_ids - An array of numbers each of which is a product id returned from getDresses
 */
showDresses(product_ids)
```

_The agent calls `getDresses(6)` and receives a JSON object:_

```json
{
    "products": [
        {
            "id": 1021,
            "description": "A short sleeve long dress with full length button placket...",
            "price": "€180",
            "image": "img_1024.png"
        },
        {
            "id": 4320,
            "description": "A straight midi dress in organic cotton...",
            "price": "€140",
            "image": "img_4320.png"
        },
        ...
    ]
}
```

> [!Note]
> How to pass images and other non-textual data is something we should improve (See [Issue #10](https://github.com/webmachinelearning/webmcp/issues/10))

_The agent can now process this list, fetching each image, and using the user's criteria to filter the list. When
completed it makes another call, this time to `showDresses([4320, 8492, 5532, ...])`. This call updates the UI on the
page to show only the requested dresses._

_This is still too many dresses so Maya finds an old photo of herself in a summer dress that she really likes and shares
it with her agent._

**Maya**: Are there any dresses similar to the dress worn in this photo? Try to match the colour and style, but continue
to show me dresses appropriate for cocktail-attire.

_The agent uses this image to identify several new parameters including: the colour, the fit, and the neckline and
narrows down the list to just a few dresses. Maya finds and clicks on a dress she likes._

_Notice, the user did not give their size, but the agent knows this from personalization and may even translate the stored
size into EU units to use it with this site._

### Example - Code Review

Some services are very domain specific and/or provide a lot of functionality. A real world example is the Chromium code
review tool: Gerrit. See [CL#5142508](https://crrev.com/c/5142508). Gerrit has many features but they're not obvious just by
looking at the UI (you can press the '?' key to show a shortcut guide). In order to add a comment to a line, the user
must know to press the 'c' key. The user can suggest edits but has to open a comment to do so. Results from test runs
are available but are hidden in a generically-named "Checks" tab.

Agents are typically trained on everyday usage so may do a poor job on more specialized, complex interfaces. However,
such sites could provide the agent with tools which serve as both a shortcut and a user manual for the agent.

_John is a software developer and opens a code review sent from his colleague. He notices there's two red bots
indicating test failures on this patch._

**John**: Why are the Mac and Android bots failing?

_The site includes some relevant tools:_

```js
/**
 * Returns the status of each bot run in a try run job. Use the returned bot_name in getTryRunFailureSnippet
 * to get more details about a run.
 */
getTryRunStatuses();

/**
 * If a bot failed, returns the TAIL snippet of the log containing the error message describing the failure.
 *
 * bot_name - The name of the bot, as returned from getTryRunStatuses, to get failure snippet from
 */
getTryRunFailureSnippet(bot_name)
```

_The agent calls `getTryRunStatuses()` to find the names of the failing bots. It returns:_

```json
[
    {
        "bot_name": "mac-x86-rel",
        "status": "PASSED"
    },
    {
        "bot_name": "mac-x64-rel",
        "status": "FAIL"
    },
    {
        "bot_name": "android-15-rel",
        "status": "PASSED"
    },
    ...
]
```

**Agent**: I see that the Mac x86 Android 10 bots are failing. Let me get more details...

_The agent now calls `getTryRunFailureSnippet` on each of the failing bots and receives a snippet of the failing log of
each._

**Agent**: The Mac bot is failing to start with an "Out of Space" error. This is probaby just an ifrastructure issue.

The Android bot is failing while linking with a missing symbol `gfx::DisplayCompositor`.

_John realizes his colleague forgot to add the Android implementation of this class to the build file._

**John**: I think BUILD.gn is missing `display_compositor_android.cc`, please add a suggested edit adding it in the
Android section.

_The agent has access to BUILD.gn from the main UI as well as in DOM. It notices it also has access to a relevant tool
which is uses to add the change:_

```js
/*
 * Adds a suggested edit to the review
 *
 * filename - the name of the file in which to make the edit
 * patch - the edit diff in unidiff format
 */
addSuggestedEdit(filename, patch)
```

**Agent**: Ok, I've added `display_compositor_android.cc` to the BUILD.gn file.

_The UI displays the suggested diff with an option for the user to accept, modify, or reject the change. John accepts
the change._

_Reading the rest of the review, John notices a small issue repeated across multiple files._

**John**: Add a polite comment to the review that we should use "PointF" rather than "Point" for input coordinates since
the latter can cause unintended rounding. Then add suggested edits changing all instances where Point was added to
PointF.

_The agent automates the repetitive task of making all the simple changes. The UI provides John with a visual way to
quickly review the agent's actions and accept/modify/reject them._

## Assumptions

* For many sites wanting to integrate with agents quickly - augmenting their existing UI with WebMCP tools will be
  easier vs. backend integration
* Agents will perform quicker and more successfully with specific tools compared to using a human interface.
* Users might use an agent for a direct action query (e.g. “create a 30 minute meeting with Pat at 3:00pm”), complex
  cross-site queries (e.g. “Find the 5 highest rated restaurants in Toronto, pin them in my Map, and book a table at
  each one over the next 5 weeks”) and everything in between.

## Prior Art

### Model Context Protocol (MCP)

MCP is a protocol for applications to interface with an AI model. Developed by Anthropic, MCP is supported by Claude
Desktop and Open AI's Agents SDK as well as a growing ecosystem of clients and servers. 

In MCP, an application can expose tools, resources, and more to an AI-enabled application by implementing an MCP server.
The server can be implemented in various languages, as long as it conforms to the protocol. For example, here’s an
implementation of a tool using the Python SDK from the MCP quickstart guide:

```python
@mcp.tool()
async def get_alerts(state: str) -> str:
    """Get weather alerts for a US state.

    Args:
        state: Two-letter US state code (e.g. CA, NY)
    """
    url = f"{NWS_API_BASE}/alerts/active/area/{state}"
    data = await make_nws_request(url)

    if not data or "features" not in data:
        return "Unable to fetch alerts or no alerts found."

    if not data["features"]:
        return "No active alerts for this state."

    alerts = [format_alert(feature) for feature in data["features"]]
    return "\n---\n".join(alerts)
```

A client application implements a matching MCP client which takes a user’s query, communicates with one or more MCP
servers to enumerate their capabilities, and constructs a prompt to the AI platform, passing along any server-provided
tools or data.

The MCP protocol defines how this client-server communication happens. For example, a client can ask the server to list
all tools which might return a response like this:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "tools": [
      {
        "name": "get_weather",
        "description": "Get current weather information for a location",
        "inputSchema": {
          "type": "object",
          "properties": {
            "location": {
              "type": "string",
              "description": "City name or zip code"
            }
          },
          "required": ["location"]
        }
      }
    ],
    "nextCursor": "next-page-cursor"
  }
}
```

Unlike OpenAPI, MCP is transport-agnostic. It comes with two built in transports: stdio which uses the systems standard
input/output, well suited for local communication between apps, and Server-Sent Events (SSE) which uses HTTP commands
for remote execution.

### WebMCP (MCP-B)

[MCP-B](https://mcp-b.ai/), or Model Context Protocol for the Browser, is an open source project found on GitHub [here](https://github.com/MiguelsPizza/WebMCP) and has much the same motivation and solution as described in this proposal. MCP-B's underlying protocol, also named WebMCP, extends MCP with tab transports that allow in-page communication between a website's MCP server and any client in the same tab. It also extends MCP with extension transports that use Chromium's runtime messaging to make a website's MCP server available to other extension components within the browser (background, sidebar, popup), and to other external MCP clients running on the same machine. MCP-B enables tools from different sites to work together, and for sites to cache tools so that they are discoverable even if the browser isn't currently navigated to the site.

### OpenAPI

OpenAPI is a standard for describing HTTP based APIs. Here’s an example in YAML (from the ChatGPT Actions guide):

```yaml
openapi: 3.1.0
info:
  title: NWS Weather API
  description: Access to weather data including forecasts, alerts, and observations.
  version: 1.0.0
servers:
  - url: https://api.weather.gov
    description: Main API Server
paths:
  /points/{latitude},{longitude}:
    get:
      operationId: getPointData
      summary: Get forecast grid endpoints for a specific location
      parameters:
        - name: latitude
          in: path
          required: true
          schema:
            type: number
            format: float
          description: Latitude of the point
        - name: longitude
          in: path
          required: true
          schema:
            type: number
            format: float
          description: Longitude of the point
      responses:
        '200':
          description: Successfully retrieved grid endpoints
          content:
            application/json:
              schema:
                type: object
                properties:
                  properties:
                    type: object
                    properties:
                      forecast:
                        type: string
                        format: uri
                      forecastHourly:
                        type: string
                        format: uri
                      forecastGridData:
                        type: string
                        format: uri
```

A subset of the OpenAPI specification is used for function-calling / tool use for various AI platforms, such as ChatGPT
Actions and Gemini Function Calling. A user or developer on the AI platform would provide the platform with the OpenAPI
schema for an API they wish to provide as a “tool”. The AI is trained to understand this schema and is able to select
the tool and output a “call” to it, providing the correct arguments. Typically, some code external to the AI itself
would be responsible for making the API call and passing the returned result back to the AI’s conversation context to
reply to the user’s query.

### Agent2Agent Protocol

The Agent2Agent Protocol is another protocol for communication between agents. While similar in structure to MCP (client
/ server concepts that communicate via JSON-RPC), A2A attempts to solve a different problem. MCP (and OpenAPI) are
generally about exposing traditional capabilities to AI models (i.e. “tools”), A2A is a protocol for connecting AI
agents to each other.  It provides some additional features to make common tasks in this scenario more streamlined, such
as: capability advertisement, long running and multi-turn interactions, and multimodal input/output.

## Open topics

### Security considerations

There are security considerations that will need to be accounted for, especially if the WebMCP API is used by semi-autonomous systems like LLM-based agents. Engagement from the community is welcome.

### Model poisoning

Explorations should be made on the potential implications of allowing web developers to create tools in their front-end code for use in AI agents and LLMs. For example, vulnerabilities like being able to access content the user would not typically be able to see will need to be investigated.

### Cross-Origin Isolation

Client applications would have access to many different web sites that expose tools. Consider an LLM-based agent. It is possible and even likely that data output from one application's tools could find its way into the input parameters for a second application's tool. There are legitimate reasons for the user to want to send data across origins to achieve complex tasks. Care should be taken to indicate to the user which web applications are being invoked and with what data so that the user can intervene.

### Permissions

A trust boundary is crossed both when a web site first registers tools via WebMCP, and when a new client agent wants to use these tools. When a web site registers tools, it exposes information about itself and the services it provides to the host environment (i.e. the browser). When agents send tool calls, the site receives untrusted input in the parameters and the outputs in turn may contain sensitive user information. The browser should prompt the user at both points to grant permission and also provide a means to see what information is being sent to and from the site when a tool is called. To streamline workflows, browsers may give users the choice to always allow tool calls for a specific web app and client app pair.

### Model Context Protocol (MCP) without WebMCP

MCP has quickly garnered wide interest from the developer community, with hundreds of MCP servers being created. WebMCP is designed to work well with MCP, so that developers can reuse many of the MCP topics with their front-end website using JavaScript. We originally planned to propose an explainer very tightly aligned with MCP, providing all the same concepts supported by MCP at the time of writing, including tools, resources, and prompts. Since MCP is still actively changing, matching its exact capabilities would be an ongoing effort. Aligning the WebMCP API tightly with MCP would also make it more difficult to tailor WebMCP for non-LLM scenarios like OS and accessibility assistant integrations. Keeping the WebMCP API as agnostic as possible increases the chance of it being useful to a broader range of potential clients.

We expect some web developers will continue to prefer standalone MCP instead of WebMCP if they want to have an always-on MCP server running that does not require page navigation in a full browser process. For example, server-to-server scenarios such as fully autonomous agents will likely benefit more from MCP servers. WebMCP is best suited for local browser workflows with a human in the loop.

The WebMCP API still maps nicely to MCP, and exposing WebMCP tools to external applications via an MCP server is still a useful scenario that a browser implementation may wish to enable.

### Existing web automation techniques (DOM, accessibility tree)

One of the scenarios we want to enable is making the web more accessible to general-purpose AI-based agents. In the absence of alternatives like MCP servers to accomplish their goals, these general-purpose agents often rely on observing the browser state through a combination of screenshots, and DOM and accessibility tree snapshots, and then interact with the page by simulating human user input. We believe that WebMCP will give these tools an alternative means to interact with the web that give the web developer more control over whether and how an AI-based agent interacts with their site.

The proposed API will not conflict with these existing automation techniques. If an agent or assistive tool finds that the task it is trying to accomplish is not achievable through the WebMCP tools that the page provides, then it can fall back to general-purpose browser automation to try and accomplish its task.

## Future explorations

### Progressive web apps (PWA)

PWAs should also be able to use the WebMCP API as described in this proposal. There are potential advantages to installing a site as a PWA. In the current proposal, tools are only discoverable once a page has been navigated to and only persist for the lifetime of the page. A PWA with an app manifest could declare tools that are available "offline", that is, even when the PWA is not currently running. The host system would then be able to launch the PWA and navigate to the appropriate page when a tool call is requested.

### Background model context providers

Some tools that a web app may want to provide for agents and assistive technologies may not require any web UI. For example, a web developer building a "To Do" application may want to expose a tool that adds an item to the user's todo list without showing a browser window. The web developer may be content to just show a notification that the todo item was added.

For scenarios like this, it may be helpful to combine tool call handling with something like the ['launch'](https://github.com/WICG/web-app-launch/blob/main/sw_launch_event.md) event. A client application might attach a tool call to a "launch" request which is handled entirely in a service worker without spawning a browser window.

## Acknowledgments

Many thanks to [Alex Nahas](https://github.com/MiguelsPizza) and [Jason McGhee](https://github.com/jasonjmcghee/) for sharing related [implementation](https://github.com/MiguelsPizza/WebMCP) [experience](https://github.com/jasonjmcghee/WebMCP).
