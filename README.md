# Figma-to-Code

A Claude skill that converts Figma design files into production-ready Next.js websites with headless CMS integration.

In my own experience, this skill reduced a 100-hour development effort to less than 5 hours — the skill itself ran for about 45 minutes, followed by a couple hours of prompting to reach near pixel-perfect results.

## What It Does

Figma-to-Code is a 14-phase automation that handles the full journey from design file to deployed website:

1. **Pre-flight** — Validates credentials, explores the Figma file, and auto-discovers all pages
2. **Project Init** — Scaffolds a Next.js 14+ project with TypeScript, Tailwind, and App Router
3. **Design Tokens** — Extracts colors, typography, spacing, and other tokens from Figma variables
4. **Assets** — Exports images, configures fonts, and uploads assets to CDN
5. **Content Types** — Creates CMS content types modeled around reusable Figma components
6. **Global Content** — Extracts navigation, footer, site settings from Figma and populates the CMS
7. **Components** — Generates React components with zero hardcoded strings and inline-editor data attributes
8. **Page Content** — Extracts page-level content from Figma and creates CMS entries
9. **Pages** — Assembles pages using dynamic routing driven by CMS data
10. **Content Audit** — Scans for any remaining hardcoded content
11. **Editor Audit** — Validates inline-editor data attributes across all components
12. **QA** — Automated screenshot comparison across desktop, tablet, and mobile viewports
13. **Production Audit** — Migrates temporary image URLs to permanent CDN, runs build verification
14. **Deployment** — Deploys to Vercel or AWS Amplify

## What You Need

- **Claude** — Pro or Team plan, running via Claude Code
- **Figma** — Your design file, with the Figma MCP tool connected to Claude
- **A headless CMS with a GraphQL API** — The skill was built around a specific CMS, but the plain-English instructions can be adapted to any GraphQL-based CMS (Hygraph, Contentful, Strapi, etc.)

## How to Use It

1. Clone or download this repo
2. Set up Claude Code with the Figma MCP tool
3. Point Claude at the `SKILL.md` file in this repo as a skill
4. Provide your Figma file URL, CMS credentials, and project configuration
5. Let it run

The skill manages its own context across the 14 phases using a breakpoint system — it writes continuity files between phase groups so it can pick up where it left off after clearing context.

## Customizing It

At the end of the day, this is just a set of plain English instructions to an LLM. Every phase is a markdown file you can read, understand, and modify. If you use a different CMS, a different framework, or want to change how components are generated — just edit the relevant phase file. No code to learn, just English to rewrite.

## A Note on Results

This runs on an LLM, so results will vary. The same site could come out quite clean on the first pass, and on a second run could require more prompting and iteration. The complexity of your Figma file, how well-structured your design is, and your CMS setup all affect the outcome. My own experience was a roughly 20x reduction in effort, but your mileage may genuinely vary.

## License

MIT — use it however you like.
