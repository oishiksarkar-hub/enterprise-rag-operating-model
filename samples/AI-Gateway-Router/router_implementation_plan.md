# Implementation Plan: Central AI Gateway Router

## 🎯 Target Objective
Build a lightweight, high-throughput gateway service that acts as the single entry point for all enterprise AI requests. This component evaluates the incoming intent, checks global security constraints, and forwards the query to the correct team-specific RAG endpoint.

## 🧬 Architectural Mapping
* **Corresponds to:** `docs/chapter-2-interfaces-obs/chapter-2-interfaces-obs.md` (Routing, Security Propagation)
* **Technology Stack:** Python 3.11+, FastAPI, Asyncio, Google Cloud IAM/OIDC Clients.

## 🧱 Modular Structural Design
The code will be structured into three isolated modules:
1. `auth.py`: Validates corporate JWT/OIDC tokens, extracts identity scopes, and wraps outgoing headers.
2. `router.py`: Implements semantic intent classification using an ultra-cheap model (e.g., Gemini Flash) to determine which team owns the query context.
3. `main.py`: The core asynchronous execution loop managing error handling and proxying the request downstream.

## ⚙️ Operational Scope & Error Boundaries
* **DLP Check:** Integrates with Google Cloud DLP API to sanitize inputs before routing.
* **Resiliency Plan:** Implements circuit breakers and retries with exponential backoff for downstream team calls.
* **Error States Managed:** `401 Unauthorized` (Malformed token), `403 Forbidden` (Insufficient scopes), `504 Gateway Timeout` (Downstream team RAG lag).

## 🧪 Testing & Validation Plan
* **Unit Tests (`test_auth.py`):** Mocks JWT verification loops using expired, forged, and valid signatures to guarantee security bounds.
* **Integration Tests (`test_routing_flow.py`):** Asserts that an "HR-focused string" resolves correctly to the HR agent endpoint URL.
