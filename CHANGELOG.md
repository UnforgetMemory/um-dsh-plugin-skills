# Changelog

## [Unreleased]

## [0.1.2] - 2026-09-09

### Added
- README.en.md: standalone English README mirroring the Simplified Chinese default, with cross-links, hero banner, badge row, and Ko-fi support section
- LICENSE: MIT license (Copyright (c) 2026 UnforgetMemory), surfaced via badge in both READMEs

### Changed
- README.md: redesign as centered bilingual landing page — hero banner (docs/assets/hero.png), badge row, table-based mental models

### Removed
- _test_img.png: unused image-analysis smoke fixture, no references remain

## [0.1.1] - 2026-09-09

### Added
- umdshdev SKILL.md: full 18-page guide map (group / path / topic) with verified official-site and CLI-reference fallback links
- umdshdev professions: explicit guide page lists replacing `<page>.md` placeholders
- publish-distribution reference: troubleshooting for the `declares no dsh.bundle` warning — reconcile mechanism, author fix, consumer activation via update plus profile restart
- guide/basic/publish: pitfall entry for bundle-less dependencies and the reconcile pass

### Fixed
- add required SKILL.md frontmatter (name, description) for the skills CLI ecosystem
- repair PowerShell-interpolated deep-dive pointers in all umdshdev references, now targeting real guide page paths
- replace dead `/en/develop/` official links (404; the section has no index page) with the section entry `/en/develop/basic/`

## [0.1.0] - 2026-09-08

### Added
- add umdshdev skill system: main router entry (SKILL.md), four professions (basic / tutorial / framework / practice), and eight reference files for on-demand, token-efficient loading
- add distilled content layer: 18-page bilingual distillation of the DSH develop guide under umdshdev/guide/ (self-contained, ships with the skill)
- add dynamic Cordis reference covering in-session plugin creation (cordis_define path)
- add project README index and .um.agents project memory area (memory/ stays local-only)

### Changed
- normalize raw-source storage into the shared area .um.agents/memory/raw-source/ per umpp artifact-routing spec
