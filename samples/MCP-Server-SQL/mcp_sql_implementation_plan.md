# Implementation Plan: Model Context Protocol (MCP) SQL Connector

## 🎯 Target Objective
Deploy an absolute, zero-trust Model Context Protocol (MCP) server that safely exposes targeted team databases (such as analytical views inside BigQuery or AlloyDB) as raw executable tools directly to external LLM cores.

## 🧬 Architectural Mapping
* **Corresponds to:** `docs/chapter-2-interfaces-obs/chapter-2-interfaces-obs.md` (MCP vs A2A Tools)
* **Technology Stack:** Node.js/TypeScript or Python with Anthropic `@modelcontextprotocol/sdk`.

## 🧱 Modular Structural Design
1. `mcp_server.py`: Instantiates the MCP base engine and registers available tool definitions.
2. `db_client.py`: Controls strict, read-only SQL connection limits and handles query serialization.
3. `sandbox.py`: Enforces string-cleaning logic to prevent malicious SQL-injection payloads from unvetted user prompts.

## ⚙️ Operational Scope & Error Boundaries
* **Enclosure Security:** Implements parameterized execution models to prevent query modifications.
* **Error States Managed:** SQL parsing anomalies, timeout blocks on large table scans, and database disconnection failures.

## 🧪 Testing & Validation Plan
* **Security Validation (`test_sandbox.py`):** Asserts that passing a string like `; DROP TABLE Users;` halts execution immediately at the application border.
* **Protocol Assertions (`test_mcp_spec.py`):** Validates that the server schema answers correctly to standard MCP client handshakes.
