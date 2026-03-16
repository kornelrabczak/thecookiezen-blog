---
layout: post
title: "Model Context Protocol: Beyond AI integration"
featuredImage: "/images/mcp-beyond.jpg"
author: "Korneliusz Rabczak"
date: 2026-03-13 16:00:00 +0100
description: "Exploring the Model Context Protocol for business process automation and the challenges of optional output schemas."
keywords: [mcp, ai, automation, business process, integration, protocol]
categories: [ai, integration, automation]
comments: true
---

Could MCP be the universal integration layer
-------------------------------------

These are notes from a year ago, when I began learning more about the [Model Context Protocol](https://modelcontextprotocol.io), a protocol designed to connect Large Language Models (LLMs) to external tools. I wondered if this protocol could also serve as a universal integration layer for business process automation systems.

Business process automation tools struggle with integration challenges. Each new service requires writing a new adapter, handling different authentication schemes, and determining what capabilities are available. MCP has already solved the discovery problem with its standardized tool listing, and the protocol provides a clean way to invoke those tools. If workflow engines adopted MCP, they could open up to an entire ecosystem of integrations without the constant need for custom implementations.

Service providers like Google and Microsoft already expose MCP servers for their services. Others have begun offering MCP as a service. A workflow orchestration engine that speaks MCP would immediately gain access to all of these tools without writing any custom integration code, except for implementing the MCP client in the system.

<!-- more -->

Where MCP and workflow engines don't align
-------------------------------------

However, when I began mapping MCP to various workflow orchestration engine solutions, I encountered a fundamental architectural discrepancy.

MCP is designed for real-time, persistent connections between chat applications, LLMs, and tools. The client stays active, maintains a session, and receives responses directly. Most workflow engines, on the other hand, operate with a short-lived, event-based approach. A client is instantiated for a specific action, performs its task, and then shuts down immediately.

This works fine for synchronous operations. The client calls a tool, receives the response, passes it to the next workflow step, and then terminates. However, for asynchronous operations, which are common in real-world business processes, the model breaks down. Once a client initiates a long-running task and shuts down, there is no standard MCP way to receive the result.

The only solution for now is polling. You periodically spin up a new client instance to check if the job is finished. This works, but it's inefficient. What would really help are callbacks and triggers, which the MCP community has been discussing for the future. This would give clients the ability to register an endpoint that the MCP server could notify when a result is ready.


Why optional schemas are a problem
-------------------------------------

Another challenge is that output schemas are optional in MCP. This makes sense for a chat application. An LLM can parse unstructured text and extract meaning. If a tool returns free-form text, the LLM can handle it. However, for a workflow engine, this is a showstopper without additional result transformation.

When chaining automation steps, reliable, structured output is necessary to feed the next step. If a tool returns unstructured text, you need to build text-parsing logic to extract the necessary data.

The MCP specification enables tools to define an output schema, which solves this problem by allowing for validation and reliable parsing. However, it's optional. A third-party MCP server might not provide it, so the workflow engine must handle that case gracefully.

One solution would be to receive only a string and build parsing logic for each tool, ideally using an expression language to transform strings into structured objects. However, this approach feels like building fragile infrastructure on top of a foundation that could be solid if the schema were mandatory. Type-safe validation eliminates parsing ambiguity and creates the reliability that business processes require.

Making it work today
-------------------------------------

Although self-hosted MCP servers from third-party providers are currently the most common approach, major players like GitHub and Atlassian are emerging with "MCP-as-a-Service." Their alignment with initiatives like internal tool registries suggests that this trend will continue to grow.

Using MCP for non-AI use cases is feasible, but requires well thought implementation. Start with synchronous operations where the short-lived client model is effective. Plan the architecture to transition to callbacks when the protocol matures. Build the infrastructure to handle both structured and unstructured outputs, bearing in mind that the long-term goal is mandatory schemas.

The ecosystem is young and evolving fast. The question isn't whether MCP can work for business process automation but rather how quickly the protocol will mature to support the patterns that automation platforms depend on.
