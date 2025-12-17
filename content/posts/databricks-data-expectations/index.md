---
title: "Databricks Data Quality Expectations Guide"
date: 2025-09-21
summary: "Practical walkthrough for implementing data quality expectations in Databricks Delta Live Tables (Lakeflow Declarative Pipelines). Covers a four-step process to profile data, formalize and translate rules, quarantine noncompliant records, and balance strict versus permissive checks, with lessons learned and links to the talk, slides, and demo code." 
draft: false
tags: ["databricks", "data quality"]
slug: "data-quality-expectations"
author: "Matthew Norberg"
---

**Disclaimer:** This presentation was originally created when the technology was called Delta Live Tables (DLT). Databricks has since rebranded it as Lakeflow Declarative Pipelines. While the name has changed, many of the strategies and techniques for applying data quality rules remain the same. For the latest documentation, see the [Databricks Lakeflow Declarative Pipelines](https://docs.databricks.com/aws/en/dlt/).

---

Ensuring high data quality is critical for analytics, decision-making, and building reliable AI/ML models. Without it, organizations risk costly errors, unreliable insights, and ineffective models.  

In this talk, I share a practical guide to implementing **data quality expectations** within Databricks’ Delta Live Tables (DLT). Think of expectations as “unit tests for data” — rules that define what your data should look like. By applying them, you can:  

- Measure the proportion of data that meets quality standards  
- Quarantine or block bad data before it flows downstream  
- Detect bugs in code that cause quality issues  
- Lay the foundation for data quality monitoring and reporting  

The session walks through a **four-step process** for writing expectations: profiling your data, collaborating with domain experts, documenting rules, and translating them into SQL. I also share lessons learned, tips for managing quarantines, and strategies to balance strict vs. loose expectations.  

📺 **Watch the full presentation on YouTube** https://www.youtube.com/watch?v=Uk3kN97NgPk&t=2s 

📑 **Download the slides and demo code on GitHub** https://github.com/mnorberg-dev/data-expectations 

Whether you’re a developer working hands-on with DLT pipelines or simply looking to understand how to raise the bar on your organization’s data quality, this talk will give you actionable tools to get started.  