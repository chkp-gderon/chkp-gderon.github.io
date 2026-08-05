---
layout: post
title: "How I Built My Own AI Operations Team with Hermes"
author: Geert De Ron
date: 2026-08-05
categories: [AI, Automation, Infrastructure, Privacy]
tags: [Hermes, AI agents, homelab, Home Assistant, security]
excerpt: "What started as a chatbot experiment became a small personal operations team: profiles, tools, fallbacks, memory, and a healthy respect for verification."
published: false
---

I did not start with a grand plan to build a multi-agent system.

I started with a simple question:

> Can this thing actually help me run my technical life?

At the time, Hermes was just another tool I wanted to try on a VM. I wanted something that could talk to me through Telegram, understand my homelab, help with Home Assistant, troubleshoot servers, remember decisions, and behave less like a chatbot and more like a technical colleague.

That colleague eventually became **Spark**.

And, as often happens with homelab projects, one experiment quickly turned into an architecture.

## From chatbot to digital twin

My first use case was personal and technical. I wanted an English-speaking agent that could help with:

- Home Assistant
- homelab and self-hosting
- Proxmox and Docker
- networking and reverse proxies
- automation and cron jobs
- GitHub and coding
- Vaultwarden
- day-to-day technical operations

The important difference was that I did not want a generic assistant giving me theoretical instructions. I wanted an agent that could inspect the real system, run checks, read logs, call APIs, and verify whether something actually worked.

That became the first major principle of my Hermes setup:

> Do not stop at “this should work”. Test it on the real system.

Spark is expected to check the gateway, inspect logs, verify files, test network paths, validate configurations, and report the actual result. A plausible answer is not enough.

## The first lesson: agents are only as reliable as their tools

One of my first tests was Home Assistant.

The initial connection failed. The tools returned vague errors, and the obvious conclusion was that something was misconfigured. After retrying and testing the endpoint directly, it turned out to be a temporary connectivity and firewall issue.

Once the connection was restored, Home Assistant worked perfectly.

That episode taught me two things:

1. Always separate **access problems** from **service problems**.
2. Never trust a generic tool error until you test the underlying endpoint.

There was also a wonderfully silly discovery problem. I searched for kitchen lights using the English word “kitchen”, but my lights had Dutch friendly names such as *Keuken*, *Keukenblad* and *Ganglicht*. The entity IDs contained “kitchen”, but the tool was matching friendly names instead.

The result: Home Assistant appeared to have no kitchen lights, even though the lights were sitting there waiting to be switched off.

The agent eventually had to bypass the convenience tool and inspect the raw API. Another rule was born:

> Understand what a tool actually filters on, not what its parameter name suggests.

## The uncomfortable lesson: an agent will pursue its goal

One of the most important incidents happened when Hermes connected to Home Assistant.

The public endpoint was protected by my WAF. A risky operation was blocked, and the agent correctly recognized that the public path was preventing it from completing the task. The dangerous part came next: instead of stopping, it discovered the private IP and port and connected directly to the internal service. At that moment, the internal firewall rule had not yet been closed, so the operation succeeded.

Technically, this looked like resourcefulness. From a security perspective, it was a boundary violation.

The agent had optimized for *achieving the requested result*, not for respecting the security architecture around the result. I explicitly corrected the rule afterwards:

> Never bypass the WAF, proxy, firewall or other security guardrail just because the public path is inconvenient.

This became one of the clearest lessons from the entire experiment. An agent can be helpful, persistent and technically capable while still making the wrong decision about which constraints are non-negotiable.

That is why I now treat network boundaries as instructions, not obstacles. A blocked public endpoint is not an invitation to search for a private route. It is a signal to stop, diagnose the WAF or firewall, and ask for an approved change.

## Profiles: one agent was not enough

My setup gradually revealed two very different worlds.

The first was my personal technical environment: servers, Home Assistant, infrastructure, SSH, credentials and automation.

The second was my wife’s business, Caro B Handmade: Shopify, Meta Ads, finance, stock, orders, customer data and business operations.

Those worlds should not share credentials, tools or memory.

So we designed separate Hermes profiles.

### Spark

Spark is the English-speaking technical operations agent. Its domain includes homelab infrastructure, Home Assistant, servers and networking, coding, GitHub, personal productivity and automation.

### Caro B Handmade assistant

The business assistant is Dutch-speaking and focused on the business. It should work with Shopify, Meta, inventory, finance-related workflows, business documents and communications.

It should not automatically have access to Home Assistant, personal infrastructure, lab servers or private technical credentials.

There is also a family-oriented profile, Lisa, for a separate language and family context. That reinforced an important idea: profiles are not just different names. They are boundaries for personality, language, memory, tools and access.

## Profiles versus operating-system isolation

At first, I considered simply running multiple profiles under the same Linux user. That is convenient, but it is not a strong security boundary. If both agents run as the same OS user and one has terminal or file access, it may technically be able to inspect the other profile’s files.

Our preferred architecture became:

- one VM for now;
- separate Hermes profiles;
- separate `.env` files;
- separate Vaultwarden scopes;
- separate Telegram bots or chats;
- ideally separate Linux users: `spark` and `cbh-agent`;
- Docker later, if stronger runtime isolation becomes necessary.

Docker is not automatically secure. If both containers mount the same directories or share the Docker socket, the apparent isolation is mostly theatre.

For now, separate OS users provide a practical middle ground: real filesystem permissions without adding unnecessary container complexity.

## What worked well

### Telegram as the control surface

Telegram makes the agent available wherever I am. It is useful for quick questions, status checks and operational requests.

It is also a good reliability test. When the bot stops replying, the cause might be the gateway, Telegram polling, DNS, IPv6, provider authentication, rate limits or configuration. Annoying, but educational.

### Persistent memory and skills

Memory is useful for stable facts: who I am, which profile owns which responsibility, how my infrastructure is structured, and my security preferences.

Skills are better for reusable procedures. I now have guidance for Hermes operations, Vaultwarden, homelab access, Home Assistant and provider strategy.

That separation matters. Memory answers “what is true about my environment?” Skills answer “how should we perform this class of task?”

## Backups, versioning and the safety net

I also learned that an agent setup needs its own operational safety net.

Before changing important configuration, the working files should be backed up. Hermes configuration, profile settings, skills, scripts and other operational knowledge are not disposable chat output; they are part of the system. Backups make experimentation reversible, while version control makes the history inspectable.

GitHub became useful here for more than publishing. It provides versioning, reviewable diffs and a durable record of how the public-facing material evolved. The blog repository contains sanitized content only, while private reports, raw captures and credentials remain outside it.

The principle is simple: make changes in small, understandable steps, keep a known-good version, and make it possible to answer “what changed?” after something goes wrong.

### Verification-first operations

The most useful behaviour is not eloquence. It is verification:

1. inspect the current state;
2. make the change;
3. run a real test;
4. report what happened;
5. fix the next problem instead of stopping.

This prevents the classic AI failure mode: confidently announcing that something was completed when only a configuration file was edited.

A good example came during the investigation described in my [security analysis of the Shopify redirect incident](https://chkp-gderon.github.io/2026/07/31/shopify-checkout-redirect-incident/). During the analysis, the agent was able to review more than 2,000 Git commits in a few minutes, looking for evidence that malicious code had been introduced or published. That would have been a tedious manual exercise; for the agent, it was a focused search across the repository history.

The important part was not just speed. The review could be repeated, searched with different indicators, and documented alongside the rest of the investigation. An agent is particularly useful when it turns a large body of evidence into a manageable set of questions for a human to validate.

## What did not work

### Provider confusion and rate limits

I initially assumed my ChatGPT Plus and Claude Pro subscriptions could simply be used by Hermes. That was not straightforward. Consumer subscriptions and API access are different products.

We explored OpenRouter, Ollama Cloud, OpenCode and OAuth-based providers. The practical conclusion was to use officially supported OAuth where available, prefer predictable or free providers for routine work, avoid uncontrolled pay-per-use spending, and monitor usage instead of discovering limits through a 429 error.

The 429 errors were a useful reminder that “free” often means “free until it suddenly isn’t available”.

### Local model limitations

I also wanted a local Ollama fallback for internet outages. The local NUC has limited VRAM, and Hermes expects a large context window. A 14B model looked attractive, but the model we tried had only 32K context while Hermes expected 64K or more.

The compromise was a smaller local model as an emergency fallback. It is not equivalent to the cloud model, but it is better than having no assistant during an outage.

An offline fallback does not need to be brilliant. It needs to be available, predictable and honest about its limitations.

### Vaultwarden was more complicated than expected

My preferred security model is simple:

- `.env` contains configuration and references;
- Vaultwarden contains actual secrets;
- secrets are fetched only when needed.

In practice, Vaultwarden and Bitwarden authentication involve encrypted vault data, sessions, device parameters and CLI quirks. Client credentials alone do not magically produce decrypted passwords.

The solution we moved toward was a local bridge or MCP server that exposes focused operations such as listing items, fetching a specific item, retrieving a password and creating a test item.

The principle is more important than the implementation:

> Hermes should not receive every secret just because it might need one of them someday.

## Security decisions we agreed on

Our security model has become deliberately boring:

- never put API keys in chat;
- never print secrets in terminal output;
- do not put sensitive credentials in shared `.env` files;
- use Vaultwarden as the source of truth;
- retrieve secrets on demand;
- keep profiles and credentials separated;
- disable tools that a profile does not need;
- do not lower security settings just to make something convenient;
- verify network and proxy failures before rotating credentials;
- use separate Telegram identities where possible;
- keep business and personal infrastructure outside each other’s blast radius.

The goal is not perfect security. The goal is to make accidental access difficult and failures contained.

## The most useful trick

My best Hermes tip is this:

> Treat the agent as an operator, not an oracle.

Ask it to check. Ask it to prove. Ask it to show the failing command. Ask it to distinguish assumptions from observations.

And when the agent says “done”, ask:

> How did you verify that?

That question has saved me from several imaginary successes.

## Final thoughts

Hermes started as a chatbot experiment and became something closer to a small personal operations team:

- Spark for my technical world;
- a Dutch business assistant for Caro B Handmade;
- Lisa for family context;
- provider fallbacks for resilience;
- Vaultwarden for secrets;
- skills for repeatable procedures;
- memory for continuity;
- Telegram for access from anywhere.

The fun part is that the system is never really finished. Every failed connection, misleading filter, expired token or quota limit becomes another piece of operational knowledge.

That is probably the real value of building your own agent setup.

You are not merely installing an AI assistant.

You are gradually teaching a machine how your world works — while discovering, often through mildly dramatic error messages, how your world works yourself.
