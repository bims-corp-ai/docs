# BIMS-Corp · Public Documentation

This repository hosts the public-facing documentation for the BIMS-Corp internal-tooling environment. It is served as a static site via GitHub Pages.

**Live site:** https://bims-corp-ai.github.io/docs/

## What's here

| Path | Audience | What it is |
|---|---|---|
| [`/`](https://bims-corp-ai.github.io/docs/) | All | Landing hub — links to the two guides below |
| [`/qa-onboarding/`](https://bims-corp-ai.github.io/docs/qa-onboarding/) | QA / PM (Windows 11) | Step-by-step install + first-bug walkthrough |
| [`/IT-OPS-HANDBOOK.html`](https://bims-corp-ai.github.io/docs/IT-OPS-HANDBOOK.html) | IT Ops | Operational reference for BIMS-Corp infrastructure |
| [`/qa-onboarding/source.md`](qa-onboarding/source.md) | Engineering | Markdown source-of-truth for the QA guide |

## Updating

Edit the files in this repo, commit + push to `main`. GitHub Pages redeploys within 30–60 seconds.

For larger changes that originate in the source workbench at [`bims-corp-ai/bims-corp-ops`](https://github.com/bims-corp-ai/bims-corp-ops), copy the updated HTML/MD files over and commit here.

## Why a separate repo?

`bims-corp-ai/bims-corp-ops` (the source workbench) is private, and GitHub Pages on the Free plan requires a public repository. This repo holds only the artifacts that are appropriate for public hosting (QA-team onboarding, IT ops reference) — none of the engineering source, plan documents, audit data, or hooks live here.
