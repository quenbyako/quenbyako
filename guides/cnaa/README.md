# Cloud Native Application Architecture

<!--
[![CC BY-ND 4.0][cc-by-nc-nd-shield]][cc-by-nc-nd]

> [!NOTE]
> This work is licensed under a [Creative Commons Attribution-NonCommercial-NoDerivatives 4.0 International License][cc-by-nc-nd].

[cc-by-nc-nd]: https://creativecommons.org/licenses/by-nc-nd/4.0/
[cc-by-nc-nd-shield]: https://img.shields.io/badge/License-CC%20BY--NC--ND%204.0-lightgrey.svg
-->

## Introduction

Developing good, user-friendly applications in the cloud isn't the same thing as writing mods or banging out HTML landing pages.

In the cloud, an application lives in a world where everything is automated: it can be restarted, moved to another server, scaled, or shut down—all without human intervention. If an application isn't ready for that kind of life, it will break.

This set of articles is about the principles that help applications survive in a cloud environment. They're called Cloud Native principles. They weren't invented by theorists, but by engineers who learned the hard way on real projects.

## Where this came from

The principles are based on the 12-Factor App methodology—a set of rules formulated by the developers at Heroku in 2011. They looked at thousands of applications on their platform and identified common patterns in the ones that ran stably. These factors aren't specific to microservices or Kubernetes; they're about applications in principle—CLI apps, microservices, and even monoliths.

A lot has changed since 2011. We got Kubernetes, microservices, observability, and security-by-design. A lot of great practices and tools have emerged that can help make an application even more reliable and easier to operate. Therefore, this guide isn't just a repetition of the 12-Factor methodology but an adaptation to modern realities, supplemented with new areas that can (and should!) be considered during development.

The following topics are **not covered** in this guidelines:

- **API Design** — this is a very complex topic; it's not required to know it to work with applications
- **Application Code Architecture** — each programming language has its own approaches that are independent of the application's architecture
- **Event-Driven Design** — a few points were already covered in Kafka
- **DBA**, and how to design data storage

They are not covered, not because they are unimportant, but because they are separate and such complex topics that it is simply impossible to cover them within a single article. Furthermore, these topics are not critically important for every specialist. This guide is specifically a collection of practices that will help both service developers and their users (because one can understand what features services might have, or which ones the developer forgot to add).

## Who This is for

This article is for developers, architects, and end users who:

- Write applications for the cloud or Kubernetes
- Want to understand how to improve their service's UX
- Are tired of problems like "but it worked on my machine"
- Plan to build distributed systems

There’s no theory for the sake of theory here. Only practical advice you can apply tomorrow, if not today. This guide helps when you need to build distributed systems and you need them to be reliable. Or when you simply want to make your service more convenient—not just for end users, but for those who will operate it. No abstract lecturing—just actionable practices you can pick up and use.

## Frameworks, FaaS and SaaS

While reading this guide you might have a very reasonable doubt: “So… to build some CRUD service I have to follow ALL these practices? Wouldn’t it be easier to grab a framework, or use FaaS/SaaS?” And that’s a good doubt! The set of principles really can feel excessive for simple applications.

But you must understand **why you’re creating a new service at all**. Yes, there are a lot of practices, and the overwhelming majority in this guide are very, very desirable. That means that if there’s a need to build a new service — then ideally _every_ point should be satisfied. Obviously that’s incredibly difficult, and you almost never have the resources to execute everything.

The only application that fulfills _all_ these practices is: no code at all (just like the only tests that never fail are the ones that don’t exist). Before you take on the obligation to build a new service, you must be 100% sure that sooner or later the application will comply with the practices. You don’t have to implement them from day zero, but you do need the readiness to carry the effort through.

Most of the time, 80% of the logic you think you need does not require creating a new service. For CRUD operations you can use postgrest or something similar; for a gateway you can take gRPC federation, and so on. It’s better to spend a week researching existing solutions on the internet than several months implementing your own service that ends up doing the same thing a mature tool already provides.

So treat the guidelines as an unattainable ideal you should strive toward — but _actually_ strive, not shove it into a drawer with saying like “well, perfection is impossible anyway.”

## How to read this guide

You don’t need to read this article cover to cover like a novel. Think of it as a recipe collection or a reference. Each of the 19 factors is an independent principle you can study and adopt separately.

Some sections include suggested tooling that can help implement the practices. The guidelines are deliberately not tied to specific tools to preserve application universality (as long as the app communicates through standard interfaces—it remains portable). Still, you’ll probably want to play with tools, test a feature, write some tests, etc. So in a few places there are links to tools that may ease adoption. They are not mandatory (and you must be sure the application does not depend on any particular external service), but they can meaningfully simplify life.

Each section also includes a checklist to let you audit your application against the guide. It’s just a list of items that helps you see whether rules are followed and whether there are any obvious gaps. **If your application does not meet some item, that does not mean it’s bad**. These checklists are simply a ready-made task queue you can pull from when you have time to improve quality.

> [!CAUTION]
> Having a checklist **does not mean** you can skip reading the guide. The checklist is for a quick glance — a terse extract. But that extract is meaningless without a full understanding of the practices and their necessity. If you want to take something from the checklist to improve—first read the section and understand **why it matters**. Otherwise the whole point is lost.
