# Guardrails Nanobrowser

**AI Security for Agentic Browsers | HUJI AI Security Hackathon 2026**

Guardrails Nanobrowser is a team project developed during the HUJI AI Security Hackathon. It extends the open-source Nanobrowser project with a runtime security layer designed to protect AI-powered browser agents from malicious or untrusted web content.

## Overview

AI browser agents interact with webpages whose content may contain malicious instructions or sensitive information. Guardrails treats webpage content as untrusted input and applies security controls before the content reaches the LLM.

The goal is to allow the browser agent to complete its intended task while reducing the risk of prompt-injection attacks and sensitive-data leakage.

## Key Security Features

- **Prompt-Injection Defense** – detects and blocks malicious instructions embedded in webpage content.
- **Data Loss Prevention (DLP)** – identifies and prevents sensitive information from being exposed to the LLM.
- **Safe LLM Context** – sanitizes untrusted webpage content before it is passed to the model.
- **Security Events** – provides visibility into detected and blocked security events.
- **Task Preservation** – aims to preserve the agent's legitimate task while filtering malicious content.

## Demo

The project includes a working security demo showing the browser agent operating with and without the Guardrails protection layer.

**Demo video:** https://youtu.be/rF_ouMImNMI

With protection disabled, malicious webpage instructions can influence the agent. With Guardrails enabled, prompt-injection attempts and sensitive data are blocked while the legitimate task can still be completed.

## Team Project

Developed collaboratively by a four-person team during the **HUJI AI Security Hackathon, May 2026**.

The project was designed and implemented collaboratively, including the security architecture, prompt-injection protection, DLP mechanisms, and demonstration environment.

## Built On

This project is based on the open-source [Nanobrowser](https://github.com/nanobrowser/nanobrowser) project.

The original Nanobrowser code and functionality belong to their respective authors and contributors. Guardrails Nanobrowser represents our team's security-focused extension developed for the hackathon.

## License

This repository retains the original project's Apache License 2.0. See [LICENSE](LICENSE) for details.
