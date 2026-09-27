# Implementation Plan: Declarative Tier 2 Infrastructure (IaC)

## 🎯 Target Objective
Provide Infrastructure-as-Code files allowing teams graduating from Tier 1 (Vertex AI Search) to deploy an automated, secure, network-isolated Tier 2 storage and computing stack instantly.

## 🧬 Architectural Mapping
* **Corresponds to:** `docs/chapter-3-golden-path/chapter-3-golden-path.md` (Tier 2 Off-Ramps)
* **Technology Stack:** Terraform (v1.5+), Google Cloud Provider Modules.

## 🧱 Modular Structural Design
1. `vpc.tf`: Generates private, isolated network subnets, restricting external internet data paths via Cloud NAT configs.
2. `alloydb.tf`: provisions an enterprise-grade Google Cloud AlloyDB cluster with the `pgvector` index layer initialized out of the box.
3. `iam.tf`: Sets up minimal service accounts, enforcing Least Privilege Access Rules to keep data secure.

## ⚙️ Operational Scope & Error Boundaries
* **Encryption Boundaries:** Forces mandatory Customer-Managed Encryption Keys (CMEK) via Cloud KMS for all sitting data blocks.
* **Network Isolation:** Restricts vector database port exposures entirely to internal enterprise service connections.

## 🧪 Testing & Validation Plan
* **Static Analysis (`tflint` / `terrascan`):** Automates configuration screening within CI pipelines to verify that zero public network entry gates are left open.
* **Spawn Testing:** Validates clean resource generation and complete asset teardowns across a test sandboxed GCP environment.
