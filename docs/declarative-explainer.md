# WebMCP Declarative API Explainer

> Last updated: February 11, 2026
>
> Status: Editor draft (non-normative)
>
> Audience: specification editors, implementers, framework authors, and agent/runtime developers

## Participate

- WebMCP repo: <https://github.com/webmachinelearning/webmcp>
- Declarative umbrella issue: <https://github.com/webmachinelearning/webmcp/issues/22>
- In-page consumer API issue: <https://github.com/webmachinelearning/webmcp/issues/51>
- WebIDL draft PR: <https://github.com/webmachinelearning/webmcp/pull/75>
- Declarative explainer PR: <https://github.com/webmachinelearning/webmcp/pull/76>

## Introduction

This document explains why WebMCP needs a declarative surface in addition to the existing imperative API.

This is an **explainer**. It provides context, design reasoning, and tradeoffs. It is not normative spec text.

The goal is to keep design discussion coherent while WebIDL, algorithms, and conformance tests continue to evolve.

## If You Are New to This Topic

This is the shortest way to read the proposal:

- A **tool** is a site action an agent can call directly.
- **Imperative WebMCP** means the site defines tools in JavaScript.
- **Declarative WebMCP** means the site exposes tool-like actions directly from HTML flows (especially forms).

The goal is not to replace JavaScript tools. The goal is to cover common form workflows without forcing authors to rewrite them.

## User-Facing Problem

WebMCP’s imperative API lets sites expose tool calls through JavaScript. That is necessary, but it is not enough for many common web workflows.

A large portion of high-value user actions on the web are still represented by HTML forms and submit flows. If those workflows are not expressible as first-class tools, authors are pushed into one of three undesirable outcomes:

- duplicate existing form behavior in new JavaScript tool handlers
- rely on brittle UI actuation paths
- skip exposing useful actions to agents entirely

For end users, this usually means slower and less reliable task completion. It also makes it harder to understand what the agent can do safely.

## The Web Today vs Proposed Change

### Today

- agents can act through UI simulation (click, type, navigate), which is often fragile
- sites can expose JS-defined tools, but form-native workflows still require extra plumbing or duplication

### Proposed change

- allow forms and related HTML flows to be exposed through a declarative tool surface
- keep browser-managed semantics for validation, submission, and user gating
- preserve imperative APIs for advanced, non-form, or highly custom behavior

## Continuity With Original WebMCP Goals

Declarative work extends original WebMCP goals, it does not replace them.

Still core:

- human-in-the-loop workflows
- browser-mediated control and visibility
- reduced reliance on brittle UI actuation
- practical developer adoption through reuse
- accessibility value

Still out of scope:

- replacing backend MCP integrations
- autonomous/headless-first primary target
- replacing human web interfaces

## Goals for Declarative WebMCP

- expose existing HTML workflows as tools with minimal rewrite
- preserve predictable browser-managed lifecycle semantics
- keep declarative and imperative surfaces composable
- produce behavior suitable for interoperable tests

## Non-Goals for Declarative WebMCP

- creating a second full programming model in markup
- forcing advanced orchestration into declarative primitives
- replacing imperative APIs for complex/non-form interactions
- encoding discovery strategy as a solved declarative-only concern

## Prior Art

WebMCP declarative design is informed by:

- [Model Context Protocol (MCP)](https://modelcontextprotocol.io/introduction)
- function-calling/schema-driven agent APIs
- prior Script Tools/WebMCP explainer and proposal work
- service-worker-like response override concepts (`respondWith` pattern)

WebMCP differs because browser lifecycle details matter directly: form semantics, navigation boundaries, frame/process transitions, and user mediation expectations.

### Additional prior art

| Prior art | Relationship to WebMCP |
|-----------|----------------------|
| [VOIX](https://svenschultze.github.io/VOIX/) ([paper](https://arxiv.org/abs/2511.11287)) | Custom `<tool>`/`<context>`/`<prop>` elements for declarative agent tools. WebMCP differs by augmenting existing forms rather than introducing custom elements. |
| [A2UI](https://a2ui.org/) (Google) | Agent-generates-UI direction; complementary to WebMCP's HTML-declares-tools-for-agents. |
| [Microformats / Microdata / RDFa](https://developer.mozilla.org/en-US/docs/Web/HTML/Guides/Microformats) | Established tradition of machine-readable structured data in HTML. WebMCP extends this from data extraction to action exposure. |
| [JSON Forms](https://jsonforms.io/) / [jsonform](https://github.com/jsonform/jsonform) | JSON Schema → HTML form (reverse direction). Validates the bidirectional form↔schema mapping. |
| [Agent Definition Language](https://www.infoq.com/news/2026/02/agent-definition-language/) (Moca) | Vendor-neutral agent definitions. ADL defines the agent; WebMCP defines the tools. |
| [WHATWG conneg analysis](https://wiki.whatwg.org/wiki/Why_not_conneg) | Documents why `Accept` header negotiation is unreliable. Informs the move away from fetch-first. |

## Proposed Approach

### High-level model

Treat declarative and imperative as complementary lanes:

- **Imperative lane**: JavaScript-defined tools for full control
- **Declarative lane**: HTML-authored tools for common workflows

The declarative lane should cover "obvious" form-driven workflows well, and hand off edge/advanced cases to imperative APIs.

### Declarative authoring direction

Current prototype direction includes form-level attributes such as:

- `toolname`
- `tooldescription`
- `toolautosubmit`

and parameter-level attributes on form-associated elements:

- `toolparamname` — override the parameter name derived from the control's `name` attribute
- `toolparamdescription` — provide a human-readable description for schema synthesis

These names are prototype names and may still change during standardization.
Issue [#22](https://github.com/webmachinelearning/webmcp/issues/22) continues to discuss alternatives, including hyphenated attribute spellings and broader element coverage.

#### Human-in-the-loop: `toolautosubmit`

The `toolautosubmit` attribute is a boolean attribute on the form:

- **Present**: the agent can auto-submit the form after filling controls. The browser submits without requiring explicit user confirmation.
- **Absent**: the browser focuses the submit button after the agent fills controls, and the agent tells the user to review and submit manually.

This is less granular than per-field elicitation (as explored in earlier `tool-elicit` designs), but it matches what Chromium is currently prototyping and provides a clear opt-in for auto-submission.

Example (non-normative):

```html
<form
  action="/flights/search"
  method="get"
  toolname="search_flights"
  tooldescription="Search flights by route/date"
  toolautosubmit
>
  <label>
    Origin
    <input name="origin" required toolparamdescription="IATA code or city" />
  </label>

  <label>
    Destination
    <input name="destination" required />
  </label>

  <label>
    Date
    <input type="date" name="date" required />
  </label>

  <button type="submit">Search</button>
</form>
```

### Invocation and completion model

Current direction for agent-invoked declarative execution is:

1. discover declarative tool
2. synthesize input schema from form controls
3. validate/fill controls from invocation arguments
4. autosubmit or pause for user submit
5. return result via one of two completion paths

Completion paths:

- same-document override via `SubmitEvent.respondWith(...)` (stay on current page)
- cross-document extraction from structured data in navigated document (after navigation to a new page)

Recent [#22](https://github.com/webmachinelearning/webmcp/issues/22) discussion also moved away from a bespoke agent-only `Accept: application/json` submission path in the current prototype direction. That older fetch-first model remains design input, not a resolved standard.

Same-document example (non-normative):

```js
document.querySelector("form[toolname='search_flights']")?.addEventListener("submit", (event) => {
  if (!event.agentInvoked) {
    return;
  }

  event.preventDefault();
  event.respondWith(
    Promise.resolve({
      content: [{ type: "text", text: "Search submitted and processed." }],
    })
  );
});
```

#### `SubmitEvent` WebIDL additions

The Chromium prototype extends `SubmitEvent` with two additions:

```webidl
partial interface SubmitEvent {
  readonly attribute boolean agentInvoked;
  undefined respondWith(Promise<any> agentResponse);
};
```

- `agentInvoked` — `true` when the form submission was triggered by an agent tool invocation, `false` for normal user submissions.
- `respondWith(...)` — allows the submit handler to intercept the form submission and provide a structured result directly to the agent, similar to the `FetchEvent.respondWith()` pattern in Service Workers.

The `toolactivated` and `toolcanceled` events are noted as future direction in Chromium's design documents but are not yet specified.

#### Processing model: cancellation on reset or definition change

If a declarative tool invocation is in-flight and either:

- the form is reset, or
- the tool declaration changes (e.g., `toolname` attribute is removed or modified)

then the in-flight invocation is cancelled and the agent is notified. This prevents stale or inconsistent results from being delivered after the form's tool identity has changed.

#### Why structured invocation differs from user action

Agent tool invocation is intentionally different from replaying user actions:

- **Schema validation**: arguments are validated against the synthesized schema before filling controls
- **Structured results**: `respondWith(...)` returns data directly to the agent without page scraping
- **Navigation control**: avoids unintended navigation, redirects, and submission side effects
- **Programmatic override**: same-document handlers can respond without a network round-trip

### Schema synthesis direction

Implementation evidence currently shows mapping work for:

- text-like controls and `textarea`
- date/time-specific handling
- number/range with min/max/step derivation
- `select`/radio/checkbox group synthesis (`oneOf`/`enum` variants)
- parameter-description fallback order (explicit attribute, label text, ARIA)

This is no longer hypothetical, but it is not final normative behavior.

### Output and response transport direction

- Imperative `outputSchema` direction is resolved at WG level.
- Declarative `outputSchema` association remains open.
- Cross-document extraction currently uses `<script type="application/ld+json">` in prototype code paths.
- Fallback behavior when no suitable structured data exists on the target page remains open ([#22](https://github.com/webmachinelearning/webmcp/issues/22), [#9](https://github.com/webmachinelearning/webmcp/issues/9)).

The two-path model (`respondWith()` + JSON-LD extraction) avoids reliance on `Accept: application/json` content negotiation, which is [unreliable on the web](https://wiki.whatwg.org/wiki/Why_not_conneg).

### Proposal variants in current discussions

The group has moved toward the Chromium prototype direction as the primary design path:

- **Current direction**: `toolname`/`tooldescription`/`toolautosubmit` attributes, `SubmitEvent.agentInvoked`, `respondWith(...)`, and `<script type="application/ld+json">` extraction for cross-document results.
- **Historical context**: older fetch-first drafts with hyphenated `tool-*` attributes and agent-specific `Accept: application/json` conventions remain design input, but they are not the current direction.

Final naming and transport semantics are not yet formally resolved at WG level, but implementation work is proceeding on the current direction.

## Practical Use Cases

### Use case 1: Form-native search tools

A site with mature form UX can expose existing behavior without duplicating validation and submission logic in separate tool handlers.

### Use case 2: Human-confirmed consequential actions

A form can be machine-prepared by an agent and user-confirmed when autosubmit is absent, matching the human-in-the-loop goal.

### Use case 3: Multi-page workflows

Cross-document declarative flows remain agent-callable when result extraction is well-defined and robustly specified.

## Accessibility Parallel: Structured Affordances for Agents

An important parallel is accessibility: the web became far more usable when authors exposed semantics instead of only pixels and click paths.

For assistive technologies, that semantic layer is accessibility metadata and accessible interaction structure. For agents, WebMCP aims to provide a similar structured layer for actions.

This framing is useful because it emphasizes:

- **semantic intent over brittle mechanics**: "submit search" instead of "click here, then type there"
- **predictability**: structured inputs/outputs are easier to reason about than arbitrary UI state
- **user control**: human confirmation points can be explicit in the execution model

It is not a perfect one-to-one mapping with accessibility APIs, but it is a strong design analogy for why structured tool surfaces improve robustness and inclusion.

Concrete analogy:

- For assistive technology, semantic headings and labels are better than raw coordinates.
- For agents, a declared tool with a schema is better than replaying brittle click/type sequences.

## Alternatives Considered

### Imperative-only

Pros:

- one explicit model
- maximal expressiveness

Cons:

- high friction for form-heavy adoption
- logic duplication risk
- poorer path for straightforward HTML-first actions

### Manifest-only declarative discovery

Pros:

- pre-navigation discoverability potential

Cons:

- definition/execution drift risk
- still requires runtime execution semantics
- does not itself solve form lifecycle behavior

### Declarative-only

Pros:

- simple for basic form workflows

Cons:

- insufficient for advanced orchestration/non-form cases
- risks overloading declarative vocabulary with imperative concerns

## State of Play (Resolved vs Prototyped vs Open)

### Resolved direction

- `navigator.modelContext` root naming resolution (October 2, 2025)
- pursue declarative + imperative together (January 22, 2026)
- use `AbortSignal` for cancellation signaling (January 22, 2026)
- add imperative `outputSchema`, investigate declarative association (February 5, 2026)

Minutes:

- <https://www.w3.org/2025/10/02-webmachinelearning-minutes.html>
- <https://www.w3.org/2026/01/22-webmachinelearning-minutes.html>
- <https://www.w3.org/2026/02/05-webmachinelearning-minutes.html>

Chromium prototype sources (schema synthesis, form integration, `SubmitEvent` additions, activation/cancel eventing) are listed under [Chromium tracking/source](#chromium-trackingsource) in References.

### Open and blocking

Open questions now directly block interop and conformance. They are no longer peripheral design polish.

## Open Questions

The umbrella issue [#22](https://github.com/webmachinelearning/webmcp/issues/22) tracks unresolved declarative semantics: attribute naming, element coverage, transport behavior, and schema edge cases. Key blocking areas include:

- **Consumer API**: the public list/execute surface is not finalized ([#51](https://github.com/webmachinelearning/webmcp/issues/51), [#74](https://github.com/webmachinelearning/webmcp/issues/74))
- **Cross-origin and iframe delegation**: policy for tools declared in iframes ([#57](https://github.com/webmachinelearning/webmcp/issues/57))
- **Concurrency and cancellation**: concurrent execution, abort signaling, dynamic registration ordering ([#47](https://github.com/webmachinelearning/webmcp/issues/47), [#48](https://github.com/webmachinelearning/webmcp/issues/48))
- **Output schema association**: declarative `outputSchema` remains open ([#9](https://github.com/webmachinelearning/webmcp/issues/9))

**Declarative context** (future direction): [PR #26](https://github.com/webmachinelearning/webmcp/pull/26) discussion raised the possibility of declarative context elements (not just tools) — for example, a `<context>` element providing agent-only text without affecting visual rendering, as explored by [VOIX](https://svenschultze.github.io/VOIX/).

## Accessibility, Internationalization, Privacy, and Security Considerations

This section is intentionally brief and points to the canonical security/privacy write-up, with declarative-specific emphasis.

- Security/privacy master doc: <https://github.com/webmachinelearning/webmcp/blob/main/docs/security-privacy-considerations.md>
- Accessibility tracking: [#65](https://github.com/webmachinelearning/webmcp/issues/65)

Declarative-specific concerns to preserve in design:

- metadata/tool-description injection risk
- mismatch between declared and actual side effects
- over-parameterization and private attribute leakage
- user-consent UX for agent-triggered form flows

Internationalization considerations remain open. They should be addressed as schema/title/description derivation is formalized.

Accessibility-specific direction should include:

- preserving clear user visibility when agent-triggered actions are prepared or submitted
- keeping authored semantics understandable across both assistive tools and agentic tools
- avoiding designs that incentivize hidden or misleading action descriptions

The group decided against reusing ARIA attributes for agent tool metadata, as ARIA and agent descriptions have different optimization targets. WebMCP uses dedicated `tool*` attributes to avoid conflicts of interest.

## Stakeholder Feedback / Opposition

From ChromeStatus feature metadata snapshot for feature `5117755740913664` on February 10, 2026:

- Chrome: Proposed
- Firefox: No signal
- Safari: No signal
- Web developers: No signals

Source:

- <https://chromestatus.com/feature/5117755740913664>

Also active:

- explicit extension-community integration thread [#74](https://github.com/webmachinelearning/webmcp/issues/74)
- active in-page consumer API thread [#51](https://github.com/webmachinelearning/webmcp/issues/51)

## References and Acknowledgements

### Core docs

- Repository: <https://github.com/webmachinelearning/webmcp>
- Rendered spec: <https://webmachinelearning.github.io/webmcp/>
- Spec source: <https://github.com/webmachinelearning/webmcp/blob/main/index.bs>
- Proposal: <https://github.com/webmachinelearning/webmcp/blob/main/docs/proposal.md>
- Security/privacy considerations: <https://github.com/webmachinelearning/webmcp/blob/main/docs/security-privacy-considerations.md>

### W3C minutes

- <https://www.w3.org/2025/09/18-webmachinelearning-minutes.html>
- <https://www.w3.org/2025/10/02-webmachinelearning-minutes.html>
- <https://www.w3.org/2025/10/16-webmachinelearning-minutes.html>
- <https://www.w3.org/2026/01/22-webmachinelearning-minutes.html>
- <https://www.w3.org/2026/02/05-webmachinelearning-minutes.html>

### Chromium tracking/source

- ChromeStatus feature: <https://chromestatus.com/feature/5117755740913664>
- Blink bug: <https://crbug.com/445637567>
- Blink intent thread: <https://groups.google.com/a/chromium.org/d/msgid/blink-dev/CANMmsAtRdyRw1WtO5va0K%3D_adYH-FRh01xvw5%2BosSd_DAq%3D%3DUQ%40mail.gmail.com>
- `ModelContext` IDL: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/script_tools/model_context.idl>
- `ModelContextTesting` IDL: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/script_tools/model_context_testing.idl>
- Form integration: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/html/forms/html_form_element.cc>
- Schema synthesis: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/html/forms/form_mcp_schema.cc>
- Execution/result handling: <https://chromium.googlesource.com/chromium/src/+/refs/heads/main/third_party/blink/renderer/core/script_tools/model_context.cc>
