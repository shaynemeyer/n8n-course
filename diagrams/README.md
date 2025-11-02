# Diagrams and Visual Aids

This directory contains Mermaid diagrams and visual aids used throughout the course.

All diagrams are created using Mermaid syntax and can be viewed directly in GitHub or any Mermaid-compatible markdown viewer.

## Diagram Index

### Module 1: Introduction
- [n8n Interface Overview](#n8n-interface-overview)
- [Basic Workflow Flow](#basic-workflow-flow)
- [Installation Options](#installation-options)

### Module 2: Core Concepts
- [Node Types](#node-types)
- [Data Flow](#data-flow)
- [Execution Modes](#execution-modes)

---

## n8n Interface Overview

```mermaid
graph TB
    subgraph "n8n Workflow Editor"
        A[Node Panel<br/>Left Sidebar] --> B[Canvas<br/>Main Area]
        B --> C[Settings Panel<br/>Right Sidebar]
        D[Top Menu Bar] --> B
        E[Executions Panel<br/>Bottom] --> B
    end

    style A fill:#f9f9f9,stroke:#333
    style B fill:#fafafa,stroke:#333
    style C fill:#f9f9f9,stroke:#333
    style D fill:#EA4B71,stroke:#333,color:#fff
    style E fill:#f0f0f0,stroke:#333
```

---

## Basic Workflow Flow

```mermaid
flowchart LR
    A[Manual Trigger] -->|Empty Data| B[Set Node]
    B -->|Data Added| C[HTTP Request]
    C -->|API Response| D[Process Data]
    D -->|Formatted Output| E[Send Email]

    style A fill:#EA4B71,stroke:#333,color:#fff
    style B fill:#00A8A0,stroke:#333,color:#fff
    style C fill:#7D54D9,stroke:#333,color:#fff
    style D fill:#FF9D00,stroke:#333,color:#fff
    style E fill:#5CA5E8,stroke:#333,color:#fff
```

---

## Installation Options

```mermaid
graph TD
    A[Choose Installation Method] --> B{Your Needs?}
    B -->|Quick Start| C[n8n Cloud]
    B -->|Learning| D[npm/npx]
    B -->|Production| E[Docker]
    B -->|Desktop Use| F[Desktop App]

    C --> G[Sign up & Start<br/>Immediately]
    D --> H[Install Node.js<br/>then npm install n8n -g]
    E --> I[docker run n8nio/n8n]
    F --> J[Download & Install<br/>Desktop App]

    style A fill:#f0f0f0,stroke:#333
    style B fill:#EA4B71,stroke:#333,color:#fff
    style C fill:#00A8A0,stroke:#333,color:#fff
    style D fill:#7D54D9,stroke:#333,color:#fff
    style E fill:#FF9D00,stroke:#333,color:#fff
    style F fill:#5CA5E8,stroke:#333,color:#fff
```
