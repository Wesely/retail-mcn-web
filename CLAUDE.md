# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A **website design project** (not an application) for a spin-off MCN management company being carved out of Vannise (a live-commerce / 直播電商 leader). The deliverable is a **4-page static website** — now consolidated to a single canonical version at the repo root (the earlier `demo_v1`–`demo_v4` explorations have been removed; v4 won). The source brief and specs live in `docs/` and `DESIGN.md`.

There is no build system, package manager, dependency tree, or test suite. Everything is hand-authored static HTML/CSS. Fonts load from Google Fonts via CDN.

## Repository layout

- `index.html` + `index.css` — the entry page at the repo root, linking to the four pages in `pages/`.
- `pages/` — the four pages: `page1.html`+`page1.css` (Brands), `page2.html` (Creators, inline styles), `page3.html`+`page3.css` (System architecture), `page4.html` (TBD board, inline styles).
- `DESIGN.md` — the visual spec (theme, OKLCH color tokens, typography, motion, enforced bans). The styling bible; read it before touching any page CSS.
- `docs/master-brief.md` — **the source of truth.** The complete business context + 4-page design brief. Read this before doing any design work. Its appendices A/B/C define terminology, hard compliance rules, and per-page task lists.
- `docs/PRODUCT.md` — product/brand strategy (register, audiences, brand personality, design principles, accessibility). The "who/why" behind the design.

This repo tracks the GitHub remote `Wesely/retail-mcn-web` (which previously held only a placeholder landing page that the site at root replaces).

## The four pages

The `index` entry page links to four pages (all in `pages/`). Their roles:

1. **Page 1 — Brands / 廠商** (external landing): "一份合約，接通整個直播銷售生態". Pain→solution table, process, placeholder cases, CTA.
2. **Page 2 — Creators / 直播主·團購主** (external landing): "人到就好，剩下的我們搞定".
3. **Page 3 — System architecture** (internal only): data model (場地/品牌/直播主+Tag/案型/結算 libraries), pre/mid/post-stream flow, AI governance plan.
4. **Page 4 — TBD board** (internal only): the red "紅色 TBD 板" living-spec, tech-feasibility matrix, risk tracking with owner/due-date/status.

Styling: `index`, `page1`, and `page3` have separate `.css` files (`index.css` at root; `page1.css`, `page3.css` in `pages/`). `page2` and `page4` keep their CSS inline in a `<style>` block. Theme is light with OKLCH tokens; fonts are Bricolage Grotesque + Noto Sans TC + JetBrains Mono / Geist Mono, loaded per-page from the Google Fonts CDN.

## Viewing / previewing

Pure static files — open `index.html` directly in a browser, or serve the repo root:

```sh
python3 -m http.server 8000   # from the repo root, then open http://localhost:8000
```

Links are relative (`index → pages/pageN.html`, each page → `../index.html`), so serve (or open) from the repo root.

## CRITICAL: external-page compliance (Master Brief 附錄 B)

Pages 1 & 2 are public-facing. These must **never** appear in copy, CTAs, meta descriptions, or alt text:

- "校友會" (alumni-network relationships) — private trust basis only
- "Tony" — has not been brought on; not public
- "Vannise" / the former company name — say "直播電商龍頭背景" instead, never name it
- Real team-member names (except Andy, if he opts in) — use roles like "資深直播企劃"
- Concrete take-rate numbers (the 5-10%) — say "業界友善的合作模式"
- Equity / compensation / investment figures
- Unverified real cases (Sunny 營養師, 水產教父, "Lowara Star 一天 50 台", etc.) — use anonymized placeholders ("某資深營養師主播") until authorized
- 牧羊人集團 / 騰氏集團 / 樂波科技 names — unless explicitly approved as case studies
- The internal "直播主分級 S/A/B/C" model — externally say only "Tag 化配對"

Pages 3 & 4 are internal-only and have **none** of these restrictions — write them plainly.

## Other standing rules from the brief

- Company name is **Live MCN** (decided 2026-06-03; the Master Brief still lists it as TBD — Live MCN supersedes that). Use it in H1/titles/logos across the site. Equity structure (股權結構) remains TBD.
- **Platform-neutral**: never hard-code dependence on any single live-commerce / order-capture system (「就醬播」 is only one option among many).
- **No C-end**: this is B2B-only; the system never touches consumer payment flow or holds customer data.
- All case studies / metrics are **placeholder fake data** until real data is collected and authorized.
- The Master Brief is a living document — when business decisions resolve, the TBD board (page 4) is where they get reflected.
