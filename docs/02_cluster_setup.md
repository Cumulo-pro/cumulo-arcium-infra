# 02 — Cluster Setup  
**Cluster creation & membership flow**

Arcium ARX Nodes can operate alone, but their true purpose is to work within a **Cluster**, which is the set of nodes that performs confidential computations.  
In this section, you will learn:

- How to create your own Cumulo cluster #1  
- How to register a node in a cluster
- How to validate that you are already joined
- Internal on-chain flow of invitations and membership

---

## 🧩 1. What is a Cluster in Arcium

A **Cluster** is a group of ARX nodes registered in Solana that:

- Process MXE computations  
- Coordinate through the Arcium runtime
- Have a node limit (`max-nodes`)
- Are identified by their **Cluster Offset**
- Register members in an on-chain list

An ARX Node **can be in multiple clusters**, but it needs to be invited or have permissions to enter.

---
