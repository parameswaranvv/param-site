---
title: "Open Policy Agent (OPA) - a Sidecar container for REST Authorization"
date: 2020-04-13T15:59:28-05:00
draft: true
description: "Using Open Policy Agent as a Sidecar container for REST Endpoint Authorization"
tags: 
    - Open Policy Agent
categories:
    - Security policy as code
    - Policy Engine
author: Param V V
ReadingTime: 10 min
---

With the widespread adoption of the cloud, the applications and data residing in the cloud become vulnerable to security threats.
There is little to no doubt that organizations need to invest an equal amount of effort and focus on securing their applications and data. 
As teams adopt a cloud native distributed architecture fashion to deliver applications, there is a need to automate the enforcement of stringent security
 measures and controls.
 
The `Open Policy Agent (OPA)` (pronounced `oh-pa`) is one such step in the direction of enforcing security policies as code.

## What?

OPA is an open source, general-purpose policy engine that unifies policy enforcement across the stack. 
OPA provides a high-level declarative language that let’s you specify policy as code and simple APIs to offload policy decision-making from your software. 
You can use OPA to enforce policies in microservices, Kubernetes, CI/CD pipelines, API gateways, and more.

At the outset, there are two aspects to security policy enforcement. 

* Policy Enforcement
* Policy Decision Making

OPA simplies this by decoupling the maintaince of policies, providing access to decision making endpoints using structured data, and enforcement. 
When your software needs to make policy decisions it queries OPA and supplies structured data (e.g., JSON) as input. 
OPA accepts arbitrary structured data as input.

<div style="alignment: center"><img src="/images/OPA/OPA-Overview.png" /></div></br>

## Why?

## When?

## How?


