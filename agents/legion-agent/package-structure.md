---
id: "agent-package-structure-001"
title: "Package Structure"
type: "reference"
category: "backend/agent"
tags: ["agent", "package", "structure"]
version: "1.0.0"
created: "2026-06-24"
updated: "2026-06-24"
author: "jxncyjq"
status: "published"
parent: null
children: []
related_docs:
  - id: "agent-index-001"
    relation: "related_to"
    path: "./index.md"
---

# Package Structure

The `cmd` package is only the process entrypoint. Agent behavior lives under
`internal`, grouped by component responsibility rather than generic helpers.

