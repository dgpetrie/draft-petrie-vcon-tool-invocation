---
title: "The JSON vCon - Tool Invocation Extension"
abbrev: "vCon Tool Invocation Extension"
category: std

docname: draft-petrie-vcon-tool-invocation-latest

submissiontype: IETF  # also: "independent", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: "Applications and Real-Time"
workgroup: "Virtualized Conversations"
keyword:
 - vcon
 - conversation
 - AI agent
 - tool invocation
 - MCP
 - agentproto
 - AUDIT
venue:
  group: "Virtualized Conversations"
  type: "Working Group"
  mail: "vcon@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/vcon/"
  github: "dgpetrie/draft-petrie-vcon-tool-invocation"
  latest: "https://dgpetrie.github.io/draft-petrie-vcon-tool-invocation/draft-petrie-vcon-tool-invocation.html"

author:
 -
    fullname: Daniel G Petrie
    organization: SIPez LLC
    email: dan.ietf@sipez.com

normative:

  JSON: RFC8259

  VCON-CORE: I-D.draft-ietf-vcon-vcon-core

informative:

  CDDL: RFC8610

  ROADMAP: I-D.draft-petrie-vcon-agent-audit-roadmap

  AUDIT-ARCH: I-D.draft-kuehlewind-audit-architecture-01

  MCP:
    target: https://modelcontextprotocol.io/specification/
    title: "Model Context Protocol Specification"

  MCP-2025-11-25:
    target: https://modelcontextprotocol.io/specification/2025-11-25
    title: "Model Context Protocol Specification, Revision 2025-11-25"

  MCP-2026-07-28:
    target: https://modelcontextprotocol.io/specification/2026-07-28
    title: "Model Context Protocol Specification, Revision 2026-07-28"


--- abstract

A vCon is a container for data and information relating to a conversation.
This document defines an extension to the JSON vCon schema for recording the policy decisions of AI agents and their invocation of tools.
Before an agent performs an operation, the agent or its host uses policy to decide whether the operation is allowed, disallowed or requires permission.
This document defines an Event Object type for those decisions, and Event Object types for each step of a tool invocation.
It also outlines requirements for recording requests for permission and direction, which need to be defined in a future document.

--- middle

# Introduction

This document adds Event Object types for recording the policy decisions of AI agents and their invocation of tools to the JSON [JSON] vCon schema defined in [VCON-CORE].
The general framework and requirements for defining an extension to the JSON vCon schema are defined in [VCON-CORE].

AI agents converse with people and other agents, and perform operations on their behalf (e.g. invoking tools).
The conversation between an agent and a person or another agent is captured with Dialog Objects in [VCON-CORE].
The use of a tool by an agent is not dialog.
It is a request to the tool and one or more responses from it, which are captured in this document as Event Objects.
This fills the gap for tool invocations and results identified in [ROADMAP], and covers the agent view of an Action Record in [AUDIT-ARCH].

Before an agent performs an operation (e.g. invoking a tool), the agent or its host decides by policy whether the operation may be performed.
The decision is that no permission is needed, that permission is required or that the operation is disallowed.
No one is asked, so there is no conversation, only the decision and what it applies to.

The policy decision is defined in this document because the invocation of a tool is the most common operation it applies to, and the decision is part of the life cycle of a tool invocation.
Its outcome determines whether the tool is invoked, and it identifies the tool by the same parameters as the invocation.
Without it, a use of a tool which was disallowed, or for which permission was not granted, would leave no record.
The decision is defined so that it can also apply to other operations (e.g. delegating to another agent).

This document defines a state model for the policy decision (see [](#policy-model)) and a state model for a tool invocation (see [](#state-model)), with an Event Object type for each transition in them.
The Event Object types are not specific to any one protocol.
The Model Context Protocol [MCP] is used as the first example, and a mapping to it is given in [](#mapping-to-mcp).
The word "invocation" is used rather than "call", as "call" in vCon commonly implies a telephone call.

Requests for permission or direction made to the person or agent on whose behalf the agent is acting, and their responses, are not defined in this document.
They are part of the conversation, and occur outside tool invocations as well as within them (e.g. a clarifying choice put to a person, approval to delegate to another agent, and in contact centers, consent to record and IVR menu choices).
Event Object types for them need to be defined in a future document.
The requirements identified so far are outlined in [](#directives).

The following are also not in the scope of this document:

* The definition of tools and of the tools available to an agent.
  This document assumes that tools are defined elsewhere and only records their use.
* Making or enforcing the policy decision.
  Only the decision is recorded.
* The record of the invocation kept by the tool or its provider.
  This is another observation point, which may be captured in the same vCon, in a separate vCon or in a log (see [ROADMAP]).
* The protocol used to invoke the tool.
  Only the data exchanged and the outcome of the invocation are captured.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

See section 2.1 of [VCON-CORE] for terminology used in this document.

Agent:
: A party that acts on behalf of a person or another agent and that performs operations, such as invoking tools.
  TODO: align with the definition in [ROADMAP].

Host:
: The software in which an agent runs and through which it sends requests to tools.
  The host may itself provide tools, and is then a server.

Server:
: The provider of a set of tools (e.g. an [MCP] server, or the host).

Tool:
: A named function or service which an agent may invoke.
  A tool is not a party in the vCon.

Operation:
: An action which an agent performs, and to which a policy decision applies (e.g. the invocation of a tool).

Policy Decision:
: The decision made by the agent or its host, by policy, whether an operation may be performed.

Invocation:
: The use of a tool by an agent, which is one kind of operation, consisting of one or more attempts.

Attempt:
: A single request sent to a tool for an invocation, and the responses to that request.

# Extension Name

The extension name for this document is "tool-invocation".

A vCon that contains any of the Event Object types defined in this document SHOULD include "tool-invocation" in the extensions parameter.
"tool-invocation" is not required in the critical parameter (see [](#compatibility-considerations)).

TODO: revisit whether "tool-invocation" is required in the critical parameter, pending [](#incompatibility-considerations).

# Policy Decision State Model {#policy-model}

Before an operation is performed, the agent or its host makes a policy decision whether it may be performed.
When the decision is made for that operation alone, it is recorded as a "policy-decision" type Event Object (see [](#policy-decisions)).
If the operation is allowed, or permission is required and is granted, the operation is performed.
For a tool invocation, the next Event Object is then the "tool-invocation" type Event Object (see [](#state-model)).
If the operation is disallowed, or permission is required and is not granted, the operation is not performed.

The state before an operation is set by the policy decision which applies to it: a "policy-decision" type Event Object for that operation alone, or a standing decision in effect for it (see [](#policy-decisions)).
An operation covered by a standing decision has no "policy-decision" type Event Object of its own.
When no policy decision for an operation was observed (e.g. by an observer at the tool), the vCon contains no "policy-decision" type Event Object for it.

The policy decision follows the state model shown in [](#fig-policy-states).

~~~
{::include policy-decision-states.ans}
~~~
{: #fig-policy-states title="Policy Decision States"}

The states are:

* Allowed - the policy decision is that no permission is needed
* Permission Required - the policy decision is that permission is required before the operation is performed
* Disallowed - the policy decision is that the operation is disallowed, and it is not performed

The following table defines the transitions.

| From State | Event Object | To State |
| --- | --- | --- |
| (before the operation) | policy-decision, decision "allowed" | Allowed |
| (before the operation) | policy-decision, decision "permission-required" | Permission Required |
| (before the operation) | policy-decision, decision "disallowed" | Disallowed |
| Allowed | the operation (e.g. tool-invocation) | (operation is performed) |
| Permission Required, when permission is granted | the operation (e.g. tool-invocation) | (operation is performed) |
{: title="Policy Decision State Transitions"}

The request for permission and the response are not defined in this document (see [](#directives)).
When permission is not granted, the operation is not performed, and for a tool invocation no "tool-invocation" type Event Object follows.
The Event Objects of this document alone do not distinguish permission refused, from no response, from a response which was not observed.

## Policy Decisions {#policy-decisions}

The policy decides whether a decision applies to one operation only or is standing.

A decision for one operation applies only to that operation, and is recorded before the operation is performed.
It is recorded as a "policy-decision" type Event Object with the policy_life parameter "once" and the operation_id of the operation.
Depending upon the policy, a decision for one operation may be made before any attempt of it, and is then recorded before that attempt with its attempt number.

A standing decision applies to all later operations of the same kind and target.
For a tool invocation, the target is a tool, or all of the tools of a server when no tool is named.
It is recorded as a "policy-decision" type Event Object with the policy_life parameter "standing" and no operation_id.
Operations covered by a standing decision have no "policy-decision" type Event Object of their own.
A standing decision applies until a later "policy-decision" type Event Object for the same operation and target replaces it, or until the end of the vCon.
When a host starts a new vCon while standing decisions from an earlier vCon are still in effect, it SHOULD record a "policy-decision" type Event Object for each of them at the start of the new vCon.

TODO: how a standing decision for one tool and a standing decision for all of the tools of the same server interact (e.g. which applies when both are in effect).

# Tool Invocation State Model {#state-model}

A tool invocation starts with a "tool-invocation" type Event Object, after any policy decision which applies to it (see [](#policy-model)).
The invocation, and each attempt of it, follows the state model shown in [](#fig-states).
Each transition is recorded as an Event Object of the type that labels it.

~~~
{::include tool-invocation-states.ans}
~~~
{: #fig-states title="Tool Invocation States"}

The states are:

* Running - the request has been sent to the tool and it is performing the operation
* Pending - the tool has accepted the request and will complete it later
* Input Required - the tool has asked the agent for input before it continues
* Timed Out - the agent has stopped waiting for a response
* Cancel Requested - the agent has asked the tool to stop
* Completed, Failed, Errored and Denied - the attempt has ended

Running, Pending and Input Required are the Working states.
The Working states, Timed Out and Cancel Requested are the Active states.
Timed Out and Cancel Requested are not final.
A tool may still respond after the agent has timed out or asked it to stop, and that response is recorded.

The following table defines the transitions.

| From State | Event Object Type | To State |
| --- | --- | --- |
| (start of an attempt) | tool-invocation | Running |
| Running | tool-pending | Pending |
| Running, Pending, Timed Out, Cancel Requested | tool-progress | (unchanged) |
| Running, Pending | tool-input | Input Required |
| Input Required | tool-input-response | Running or Pending, the state from which the input was requested |
| Running, Pending, Input Required | tool-timeout | Timed Out |
| Running, Pending, Input Required, Timed Out | tool-cancel | Cancel Requested |
| any Active state | tool-result | Completed |
| any Active state | tool-failure | Failed |
| any Active state | tool-error | Errored |
| any Active state | tool-denied | Denied |
{: title="Tool Invocation State Transitions"}

TODO: should a "tool-input-response" type Event Object return to the state from which the input was requested, as above, or always to Running?

A vCon records what was observed.
An Event Object which does not follow the state model (e.g. because an earlier Event Object was not observed) is still recorded.

## Attempts {#attempts}

An invocation may take more than one attempt.
Each attempt starts with a "tool-invocation" type Event Object and follows the state model above.
All attempts of an invocation have the same operation_id, and each later attempt has the next attempt number.
An agent may retry after any ended attempt, for example after "tool-denied" when it has obtained more authorization, or after "tool-error".
Every attempt SHOULD be recorded, so that the escalation leading to the final attempt can be reviewed.

The Event Objects of an attempt carry its attempt number, so that a late response is associated with the attempt to which it responds.

# Parameter Definitions

## Event Object Types and Parameters {#event-types}

The following types of events are defined in this document:

* "policy-decision" - the agent or its host made a policy decision whether an operation may be performed

and the following, collectively referred to as the tool event types:

* "tool-invocation" - the agent sent a request to the tool
* "tool-pending" - the tool accepted the request and will complete it later
* "tool-progress" - the tool reported progress or partial output
* "tool-input" - the tool asked the agent for input before it continues
* "tool-input-response" - the agent sent the input requested by the tool
* "tool-result" - the tool completed the operation and returned its output
* "tool-failure" - the tool performed or attempted the operation and reported an error
* "tool-error" - the request failed at the protocol level (e.g. an unknown tool or invalid arguments)
* "tool-denied" - the tool refused to perform the operation
* "tool-timeout" - the agent stopped waiting for a response from the tool
* "tool-cancel" - the agent asked the tool to stop

The "policy-decision" type Event Object records the decision of the policy of the agent or its host, before the operation is performed (see [](#policy-model)).
It is not a response from a tool.

The "tool-denied" type Event Object records a refusal by the tool, after the request was sent to it.
It is used only when the response from the tool is identified by the protocol as a denial (e.g. for [MCP], an HTTP 401 or 403 response).
A refusal which the protocol does not distinguish from other errors is recorded as a "tool-failure" type Event Object, with the refusal in its content.

The "tool-invocation" type Event Object records the request actually sent to the tool.

In the Event Object types defined in this document:

* the sessions parameter references the Session Object(s) in which the agent performed, or decided whether to perform, the operation
* the party parameter references the agent, including in the Event Objects for policy decisions and for responses from the tool
* the tool is identified by the server and tool parameters in the "tool-invocation" type Event Object and in a "policy-decision" type Event Object for the "tool-invocation" operation, and the operation by the operation_id parameter in the other tool event types

TODO: the dialog parameter in [VCON-CORE]; decide whether it applies to the Event Object types defined in this document (e.g. a "tool-invocation" which an agent made in response to a Dialog Object).

### Event Object Parameter Applicability by Type

The following tables summarize which Event Object parameters apply to each Event Object type defined in this document.
The parameter definitions in the subsections of this document are definitive; these tables are provided as a summary.
The symbols used in the tables are: M the parameter MUST be present, S the parameter SHOULD be present, O the parameter is optional, X the parameter MUST NOT be present.
"Content" is the body, encoding and mediatype parameters or the url, content_hash and mediatype parameters (see [](#event-content)).

The following table applies to the "policy-decision" type Event Object for the "tool-invocation" operation.

| Parameter | policy-decision, policy_life "once" | policy-decision, policy_life "standing" |
| --- | --- | --- |
| type | M | M |
| time | M | M |
| sessions | M | M |
| party | M | M |
| decision | M | M |
| policy_life | M | M |
| reason | O | O |
| operation | M | M |
| operation_id | M | X |
| attempt | M | X |
| server | M | M |
| tool | M | O |
| content | X | X |
| attachment | X | X |
{: title="Policy Decision Event Parameters"}

| Parameter | tool-invocation | tool-pending | tool-progress | tool-input | tool-input-response | tool-result |
| --- | --- | --- | --- | --- | --- | --- |
| type | M | M | M | M | M | M |
| time | M | M | M | M | M | M |
| sessions | M | M | M | M | M | M |
| party | M | M | M | M | M | M |
| operation_id | M | M | M | M | M | M |
| attempt | M | M | M | M | M | M |
| server | M | X | X | X | X | X |
| tool | M | X | X | X | X | X |
| content | S | O | S | S | S | S |
| attachment | O | O | O | O | O | O |
{: title="Tool Event Parameters, Part 1"}

| Parameter | tool-failure | tool-error | tool-denied | tool-timeout | tool-cancel |
| --- | --- | --- | --- | --- | --- |
| type | M | M | M | M | M |
| time | M | M | M | M | M |
| sessions | M | M | M | M | M |
| party | M | M | M | M | M |
| operation_id | M | M | M | M | M |
| attempt | M | M | M | M | M |
| server | X | X | X | X | X |
| tool | X | X | X | X | X |
| content | S | S | S | O | O |
| attachment | O | O | O | O | O |
{: title="Tool Event Parameters, Part 2"}

The decision, policy_life, reason and operation parameters MUST NOT be present in the tool event types.

The content of each Event Object type is:

* "policy-decision" - none, the arguments of an invocation are recorded only in its "tool-invocation" type Event Object
* "tool-invocation" - the arguments sent to the tool
* "tool-pending" - any reference to the pending operation returned by the tool (e.g. a task handle)
* "tool-progress" - the progress report or the partial output
* "tool-input" - the request for input
* "tool-input-response" - the input sent to the tool
* "tool-result" - the output of the tool
* "tool-failure" - the error reported by the tool
* "tool-error" - the protocol error
* "tool-denied" - the reason given by the tool for the denial (e.g. for [MCP], the required scopes and the error description)
* "tool-timeout" - any detail of the timeout (e.g. how long the agent waited)
* "tool-cancel" - any reason for the cancellation sent to the tool

As the arguments are recorded only in the "tool-invocation" type Event Object, the arguments of an invocation which is disallowed, or for which permission is not granted, are not recorded.

TODO: whether "tool-input-response" carries the outcome of the response (e.g. accept, decline or cancel in [MCP]) as a parameter of its own, rather than only in its content.

### operation_id {#event-operation-id}

The operation_id parameter identifies the operation to which this event relates.

* operation_id: "String"

The operation_id parameter MUST be present in the Event Object types defined in this document, other than "policy-decision" type Event Objects with the policy_life "standing", in which it MUST NOT be present.
All of the Event Objects for an operation, across all of its attempts, MUST have the same operation_id.
The operation_id MUST be unique among the operations in the vCon.

The operation_id is assigned by the agent or its host, and is not the request identifier of the protocol used to perform the operation (e.g. to invoke the tool).
The protocol request identifier may change between attempts or when a request is reissued.
When a policy decision is made for an operation alone, the operation_id is assigned when the decision is made, before the operation is performed.
It can then also be referenced by the permission request and response Event Objects to be defined in a future document (see [](#directives)).

### attempt {#event-attempt}

The attempt parameter indicates the attempt of the operation to which this event relates.

* attempt: "UnsignedInt"

The attempt parameter MUST be present in the Event Object types defined in this document, other than "policy-decision" type Event Objects with the policy_life "standing", in which it MUST NOT be present.
In a "policy-decision" type Event Object, it is the attempt which the decision precedes.
The first attempt of an operation is 1, and each later attempt is one more than the attempt before it (see [](#attempts)).

### decision {#event-decision}

The decision parameter indicates the outcome of a policy decision.

* decision: "String"

The decision parameter MUST be present in "policy-decision" type Event Objects.
The value MUST be one of:

* "allowed" - no permission is needed to perform the operation
* "permission-required" - permission is required before the operation is performed
* "disallowed" - the operation is disallowed

### policy_life {#event-policy-life}

The policy_life parameter indicates how long a policy decision applies (see [](#policy-decisions)).

* policy_life: "String"

The policy_life parameter MUST be present in "policy-decision" type Event Objects.
The value MUST be one of:

* "once" - the decision applies to one operation only
* "standing" - the decision applies to later operations until it is replaced or the vCon ends

### reason {#event-reason}

The reason parameter gives the reason for a policy decision (e.g. why an operation was disallowed).

* reason: "String" (optional)

The reason parameter MAY be present in "policy-decision" type Event Objects.

### operation {#event-operation}

The operation parameter indicates the kind of operation to which a policy decision applies.

* operation: "String"

The operation parameter MUST be present in "policy-decision" type Event Objects.
This document defines the value "tool-invocation", for the invocation of a tool.
For the "tool-invocation" operation, the server and tool parameters identify the tool (see [](#event-server) and [](#event-tool)).

TODO: other operations (e.g. access to a resource, delegation to another agent, sharing data with another agent or service, and in contact centers, recording), the parameters which identify them, and whether operation names need a registry.

### server {#event-server}

The server parameter identifies the server which provides the tool.

* server: "String"

The server parameter MUST be present in "tool-invocation" type Event Objects and in "policy-decision" type Event Objects for the "tool-invocation" operation.
It MUST NOT be present in the other tool event types.
The value is one of:

* the URL of the server, when it has one (e.g. an [MCP] server reached over HTTP)
* the name of the server, when it has no URL (e.g. for [MCP], the name in serverInfo of a server reached over stdio)
* "!host", when the tool is provided by the host

A server name MUST NOT start with "!", which is reserved for values defined by this document or a vCon extension.
The "!" character is not allowed in DNS host names or IPv6 addresses, and cannot start a URL.

A server name is not guaranteed to be unique (e.g. the name in [MCP] serverInfo), so the URL SHOULD be used when there is one.

### tool {#event-tool}

The tool parameter identifies the tool that was invoked, or to which a policy decision applies.

* tool: "String"

The tool parameter MUST be present in "tool-invocation" type Event Objects, and in "policy-decision" type Event Objects for the "tool-invocation" operation with the policy_life "once".
It MAY be present in "policy-decision" type Event Objects for the "tool-invocation" operation with the policy_life "standing", and when absent, the decision applies to all of the tools of the server.
It MUST NOT be present in the other tool event types.
The value is the name of the tool as known to the server (e.g. for [MCP], the name of the tool in the tools/call request).
A tool name is unique within a server, so the server and tool parameters together identify the tool.

TODO: whether a standing "policy-decision" type Event Object may name a list of tools of a server, rather than one tool or all of them.

### Event Content {#event-content}

The content of a tool event type is included inline or referenced externally, in the same way as Dialog Object content in [VCON-CORE].

For inline included content:

* body: "*" (optional in a redacted vCon)
* encoding: "String"

Alternatively, for externally referenced content:

* url: "String" (optional in a redacted vCon)
* content_hash: "ContentHash" \| "ContentHash\[\]"

The media type of the content:

* mediatype: "String" (required for inline content)

The body and url parameters MUST NOT both be present in the same Event Object.
The mediatype parameter MUST be present for inline content, as for other inline content in [VCON-CORE] (e.g. "application/json" for the arguments of an [MCP] tool invocation).
It is not required when the content is absent.

### attachment {#event-attachment}

The attachment parameter references files which are passed to or returned by the tool.

* attachment: "UnsignedInt" \| "UnsignedInt\[\]" (optional)

The value of the attachment parameter is the index, or an array of indices, into the attachments Object array to the Attachment Objects to which this event relates.

# Requirements for Permission and Directive Events {#directives}

This section is informative.

When a policy decision requires permission, or an agent needs direction, the agent asks the person or agent on whose behalf it is acting, and they respond.
These requests and responses are part of the conversation.
They occur outside tool invocations as well as within them, and also in contact centers (e.g. consent to record and IVR menu choices).
For this reason they are not defined in this document.
Event Object types for them need to be defined in a future document.
The following requirements have been identified so far.

* PD-1: permission requests and responses, with responses beyond yes or no (e.g. allow once, allow for the Session, always allow or a choice from a menu)
* PD-2: a request and its response refer to the policy decision and the operation they apply to (e.g. by operation_id)
* PD-3: a response given by a button press (e.g. a "keydown" type Event Object) is linked to the request it answers
* PD-4: a question from a tool ("tool-input") which is passed on to the person or a parent agent, and its answer, are correlated with the "tool-input"
* PD-5: step-up consent after a "tool-denied" (e.g. an [MCP] HTTP 403 with insufficient scope) is correlated with the retried attempt
* PD-6: directive requests which are not about an operation (e.g. a clarifying choice put to a person)
* PD-7: a subagent asking its parent agent, rather than a person
* PD-8: directives no one asked for (e.g. stop, redirect, change of mode, and grant or revocation of a permission)
* PD-9: requests which end other than with an answer (withdrawn, timed out or expired, and declined versus dismissed)
* PD-10: requests passed along a chain of agents, and the answer passed back, across Sessions
* PD-11: who may answer a request, including more than one approver
* PD-12: answers given out of band (e.g. [MCP] URL mode elicitation), where only the outcome is observed
* PD-13: the kind of answer (e.g. free text, a form with a schema, or a single or multiple choice)
* PD-14: a response or an unprompted grant which changes the policy (e.g. always allow) is linked to the standing "policy-decision" type Event Object that follows it

In the absence of that document, a request and its response may be recorded with the existing Objects of [VCON-CORE]:

* the question and the answer as Dialog Objects
* a button press as a "keydown" type Event Object

These do not record the link to the policy decision and the operation, which is why Event Object types for requests and responses need to be defined.

# Compatibility Considerations

The Event Object types defined in this document are new values of the Event Object type parameter.
An implementation that does not recognize this extension ignores Event Objects with a type which it does not recognize, as described in [VCON-CORE].
It can still process the rest of the vCon correctly, as the Dialog, Attachment and other Objects do not depend on these Event Object types.

The parameters defined in this document apply only to the Event Object types defined in this document, and so are ignored with them.

# Incompatibility Considerations

This extension introduces no incompatibilities in the processing of the conversation itself.

TODO: the content of the tool event types may contain data that must be redacted (see [](#privacy-and-integrity-considerations)).
An implementation that redacts a vCon but does not recognize this extension might copy the tool event types unchanged into the redacted vCon.
Decide whether "tool-invocation" is therefore required in the critical parameter when tool event types with content are present, or whether [VCON-CORE] already requires unrecognized Objects to be removed when redacting.

# Privacy and Integrity Considerations

The content of the tool event types can contain personally identifiable information, party identifying data, the contents of files, credentials and other secrets, and data belonging to third parties.
The arguments of a "tool-invocation" and the output of a "tool-result" are the most likely to contain such data.
The content of the tool event types is subject to redaction in the same way as Dialog Object content in [VCON-CORE].
The body or url parameter MAY be absent in a redacted vCon.

The tool and server parameters can themselves reveal information about the conversation (e.g. the name of a tool which accesses a medical record).
The reason parameter of a "policy-decision" type Event Object can reveal the policy of the host, and may contain data from the conversation.

Externally referenced content MUST have a content_hash, as for other externally referenced files in [VCON-CORE].

Credentials (e.g. access tokens) used to invoke a tool MUST NOT be recorded in the Event Object types defined in this document.
This includes the content of a "tool-denied" type Event Object.

# Security Considerations

The security considerations of [VCON-CORE] apply.

* The output of a tool may contain instructions intended to manipulate an agent (e.g. prompt injection).
  Such content is recorded as it was received, and implementations processing a vCon MUST NOT act on instructions in the content of the tool event types.
* TODO: other security considerations.

# IANA Considerations

## vCon JSON Registry Additions

### vCon Extensions Names Registry

The following extension name is added to the vCon Extensions Names Registry.

| Extension Name | Extension Description | Change Controller | Specification Document(s) |
| --- | --- | --- | --- |
| tool-invocation | policy decision and tool invocation Event Object types | IESG | [](#extension-name) RFC XXXX |

### Event Type Name Registry

The following defines additional values for the Event Type Name Registry.

| Event Type Name | Event Type Description | Change Controller | Specification Document(s) |
| --- | --- | --- | --- |
| policy-decision | operation policy decision event | IESG | [](#event-types) RFC XXXX |
| tool-invocation | tool invocation request sent event | IESG | [](#event-types) RFC XXXX |
| tool-pending | tool invocation pending event | IESG | [](#event-types) RFC XXXX |
| tool-progress | tool invocation progress event | IESG | [](#event-types) RFC XXXX |
| tool-input | tool input request event | IESG | [](#event-types) RFC XXXX |
| tool-input-response | tool input response event | IESG | [](#event-types) RFC XXXX |
| tool-result | tool invocation result event | IESG | [](#event-types) RFC XXXX |
| tool-failure | tool reported failure event | IESG | [](#event-types) RFC XXXX |
| tool-error | tool invocation protocol error event | IESG | [](#event-types) RFC XXXX |
| tool-denied | tool denied invocation event | IESG | [](#event-types) RFC XXXX |
| tool-timeout | tool invocation timeout event | IESG | [](#event-types) RFC XXXX |
| tool-cancel | tool invocation cancel event | IESG | [](#event-types) RFC XXXX |

### Event Object Parameter Names Registry

The following defines additional values for the Event Object Parameter Names Registry.

| Parameter Name | Parameter Description | Change Controller | Specification Document(s) |
| --- | --- | --- | --- |
| operation_id | operation identifier | IESG | [](#event-operation-id) RFC XXXX |
| attempt | operation attempt number | IESG | [](#event-attempt) RFC XXXX |
| decision | policy decision outcome | IESG | [](#event-decision) RFC XXXX |
| policy_life | policy decision life | IESG | [](#event-policy-life) RFC XXXX |
| reason | policy decision reason | IESG | [](#event-reason) RFC XXXX |
| operation | kind of operation of a policy decision | IESG | [](#event-operation) RFC XXXX |
| server | tool server URL or name | IESG | [](#event-server) RFC XXXX |
| tool | tool name | IESG | [](#event-tool) RFC XXXX |
| body | inline tool event content | IESG | [](#event-content) RFC XXXX |
| encoding | inline tool event content encoding | IESG | [](#event-content) RFC XXXX |
| url | externally referenced tool event content | IESG | [](#event-content) RFC XXXX |
| content_hash | externally referenced tool event content hash | IESG | [](#event-content) RFC XXXX |
| mediatype | tool event content media type | IESG | [](#event-content) RFC XXXX |
| attachment | tool event attachment index or indices | IESG | [](#event-attachment) RFC XXXX |


--- back

# Mapping to MCP {#mapping-to-mcp}

This appendix describes how the Event Object types defined in this document map to the messages of [MCP].
The mapping is informative.
It is given for two revisions, [MCP-2025-11-25] and [MCP-2026-07-28], as the later revision changes how input, pending operations and retries are handled.

| Event Object Type | MCP Revision 2025-11-25 | MCP Revision 2026-07-28 |
| --- | --- | --- |
| policy-decision | no message; the host applies its policy | no message; the host applies its policy |
| tool-invocation | tools/call request | tools/call request |
| tool-pending | task-augmented response to tools/call | task handle returned by the tasks extension (io.modelcontextprotocol/tasks) |
| tool-progress | notifications/progress | notifications/progress on the response stream of the request |
| tool-input | elicitation/create request from the server; for a task, task status input_required | result with resultType "input_required"; for a task, task status input_required |
| tool-input-response | response to elicitation/create | inputResponses in a retry of the original request; for a task, inputResponses in tasks/update |
| tool-result | tools/call result with isError absent or false | tools/call result with isError absent or false |
| tool-failure | tools/call result with isError true | tools/call result with isError true |
| tool-error | JSON-RPC error response | JSON-RPC error response |
| tool-denied | HTTP 401 or 403 response (HTTP transports only) | HTTP 401 or 403 response (HTTP transports only) |
| tool-timeout | no message; the client stops waiting | no message; the client stops waiting |
| tool-cancel | notifications/cancelled | notifications/cancelled; for a task, tasks/cancel |
{: title="Mapping of Event Object Types to MCP"}

Notes:

* The operation_id is not the JSON-RPC request id, which changes when a request is reissued.
* The server parameter is the URL of a server reached over HTTP, or the name in serverInfo of a server reached over stdio.
  [MCP] tool names are unique within a server, and [MCP] says the serverInfo name is not guaranteed to be unique.
* In [MCP-2026-07-28], input is supplied by retrying the original request.
  TODO: decide whether that retry is a new attempt, with a "tool-invocation" type Event Object, or only a "tool-input-response" type Event Object in the same attempt.
* A tool reached over the stdio transport has no HTTP response, so a refusal by it is reported as a tool execution error and recorded as "tool-failure".
* After an HTTP 403 with insufficient scope, a client acting for a user may obtain a token with more scope and retry the request.
  The retry is a new attempt of the same invocation (see [](#attempts)).
  The consent of the user is a permission request (see PD-5 in [](#directives)).
* In [MCP-2026-07-28], notifications/cancelled MUST NOT be used to cancel a task, which is cancelled with tasks/cancel.
  Cancellation of a task is cooperative, and the task may still end with another status.
* A sampling/createMessage request from the server, which is deprecated in [MCP-2026-07-28], is not supported by this document.

# Example vCons

TODO

# JSON Schema for the tool-invocation Extension

This appendix provides a JSON schema for the parameters defined by the tool-invocation extension.
The text in the preceding sections of this document defines the normative definition of these parameters.
If the JSON schema provided below differs from that in the text of the preceding sections, the text is the definitive definition and the JSON schema below is to be considered incorrect.

This schema supplements, and does not replace, the core vCon JSON schema.
To fully validate a vCon that uses this extension, apply both the core vCon schema and this schema to the vCon.

The JSON schema definition below has been formatted with line breaks to print nicely on the page.
The unformatted version of the JSON schema can be downloaded from:
https://raw.githubusercontent.com/dgpetrie/draft-petrie-vcon-tool-invocation/refs/heads/main/tool_invocation_json_schema.json

~~~
{::include tool_invocation_json_schema.pp}
~~~

# CDDL for the tool-invocation Extension

This appendix provides a [CDDL] definition for the parameters defined by the tool-invocation extension.
The text in the preceding sections of this document defines the normative definition of these parameters.
If the CDDL provided below differs from that in the text of the preceding sections, the text is the definitive definition and the CDDL below is to be considered incorrect.

This CDDL is used together with the core vCon CDDL in [VCON-CORE].
It adds Event Object types through the event-type-ext socket of the core CDDL, and needs no other core socket.
Each Event Object type it defines ends with a socket of its own, so that another vCon extension can add parameters to it.

The unformatted version of the CDDL can be downloaded from:
https://raw.githubusercontent.com/dgpetrie/draft-petrie-vcon-tool-invocation/refs/heads/main/tool_invocation.cddl

~~~ cddl
{::include tool_invocation.cddl}
~~~

# Acknowledgments
{:numbered="false"}

