<div align="center">

<img src="docs/assets/hero.png" alt="um-dsh-plugin-skills" width="100%">

# um-dsh-plugin-skills

**A distilled, bilingual deep-read of the official DeepSeek Harness developer guide**

[![License](https://img.shields.io/badge/license-MIT-5B7FFF)](LICENSE)
[![Stars](https://img.shields.io/github/stars/UnforgetMemory/um-dsh-plugin-skills?style=flat&color=5B7FFF)](https://github.com/UnforgetMemory/um-dsh-plugin-skills/stargazers)
[![Guide](https://img.shields.io/badge/guide-18%20pages-5B7FFF)](umdshdev/guide/basic/index.md)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-unforgetmemory-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/unforgetmemory)

[简体中文](README.md) · **English**

</div>

---

## ✨ What Is This

A distilled reading of the official DeepSeek Harness documentation, [develop section](https://deepseek-harness.github.io/deepseek-harness/en/develop/basic/), covering all **18 pages** under `/en/develop/`:

> **basic (4) · cordis-tutorial (8) · framework (3) · practice (3)**

- One mirrored file per page: **concise Chinese explanation + English terminology + key code skeletons**
- Ships as the content layer of the self-contained dev skill [`umdshdev/`](umdshdev/README.md) (SKILL.md routing + professions + references + guide layer, installed together with the skill)
- Raw official Markdown is kept under `.um.agents/memory/raw-source/` for cross-checking; the official source always wins on details

## 🗺️ Reading Guide

| Group | Contents | Prerequisites |
|-------|----------|---------------|
| [basic](umdshdev/guide/basic/index.md) | Getting started: first plugin → tools → config → packaging & publishing | None |
| [cordis-tutorial](umdshdev/guide/cordis-tutorial/index.md) | Chapter-by-chapter tutorial of the underlying Cordis framework (lifecycle / services / events / config / composition / HMR / into the Harness) | None (starts from a scratch dir, no API key needed) |
| [framework](umdshdev/guide/framework/index.md) | Reference: plugin lifecycle · event system · services & dependencies | basic |
| [practice](umdshdev/guide/practice/index.md) | Advanced practice: three-role capability design · dynamic Cordis · LLM adapters | basic + framework/service |

**Recommended path**: `basic → framework → practice`; `cordis-tutorial` can be read in parallel, on demand.

## 📂 Directory Tree

```
umdshdev/guide/
├── basic/
│   ├── index.md                  # Your first plugin (apply / ctx / effect / inject / three forms)
│   ├── config.md                 # Plugin config (Config schema, validation, HMR)
│   ├── publish.md                # Packaging & installation (bundle / profile / layer order / git install pitfalls)
│   └── tool.md                   # Building tools (defineTool skeleton)
├── cordis-tutorial/
│   ├── index.md                  # Tutorial overview
│   ├── 01-first-plugin.md        # First plugin
│   ├── 02-lifecycle-and-effects.md   # Lifecycle and effects
│   ├── 03-services.md            # Services
│   ├── 04-events.md              # Events
│   ├── 05-config.md              # Config
│   ├── 06-composition-and-hmr.md # Composition and HMR
│   └── 07-into-the-harness.md    # Into the Harness
├── framework/
│   ├── index.md                  # Plugins & lifecycle (Fiber state machine / auto cleanup / nesting / HMR)
│   ├── events.md                 # Event system (emit / bail / serial / waterfall / typed events)
│   └── service.md                # Services & dependencies (provide / consume / isolate / built-ins)
└── practice/
    ├── index.md                  # Three-role capability design (Service Definition / Provider / Consumer)
    ├── dynamic-cordis.md         # Extending a running agent with Cordis tools
    └── llm-adapter.md            # LLM adapters (StreamChunk protocol)
```

## 🧠 Core Mental Models

| Model | In one sentence |
|-------|-----------------|
| **Everything is a plugin** | A module exporting `apply(ctx)`; `ctx` registers capabilities with automatic cleanup; `inject` declares dependencies and waits for readiness |
| **Every registration is an effect** | `ctx.on / tools.register / llm.registerAdapter / effect` are disposed automatically when the Fiber unloads — HMR leaves no residue |
| **Config = Schema** | Schemastery validates, fills defaults, and reports errors; every tunable belongs in config, never hard-coded |
| **Distribution = bundle + profile** | Bundles contribute layers, profiles decide composition order; later layers override line by line, patches replace config wholesale |
| **Capability = three roles** | Service Definition (contract) ← Provider (implementation) and Consumer (presentation) depend only on the contract, never on each other |
| **Composition = cordis.yml** | Patches stack into layers; `isolate` isolates service instances; `group` organizes plugin groups |

## 💡 How to Use

- Every file is a distilled guide: Chinese-first explanation with English terminology and code skeletons preserved, linking back to the official docs.
- For full details (nested schemas, PTC patterns, event signatures, built-in service lists), follow the "related pages" links in each page to the official source.
- Machine-fetched raw copies under `.um.agents/memory/raw-source/` are for cross-checking only — not deliverables.

## ☕ Support

If this guide helps you, buy me a coffee:

[![ko-fi](https://ko-fi.com/img/githubbutton_sm.svg)](https://ko-fi.com/unforgetmemory)

## 📌 Meta

- Distillation date range: see this repository's git history.
- Official site version: deepseek-harness `master` at fetch time.
- All rewrites are local files; nothing has been pushed to any remote.
