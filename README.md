# CPS-272 — Dev Presentation (Deployable)

Audience-friendly cut of the full design in `docs/`. For walking devs through the work
in ~30 minutes without drowning them in the annexes.

Standalone repo: push this folder to git and deploy to Vercel as a static site (no build
command, no framework — `index.html` is served at `/`).

## View it

- **Deck only:** open `index.html` in any modern browser. No build step, no server. First load needs internet once (Mermaid + markdown-renderer CDNs).
- **Deck + rendered docs:** the sidebar file list and the blue dotted annex links open the markdown **rendered inside the deck** (tables, checklists, even diagrams) — but browsers block local file reads, so serve the folder:
  - VS Code: right-click `index.html` → **Open with Live Server**, or
  - Terminal from the repo root: `npx -y serve . -l 8080`, then open `http://localhost:8080/`.
  - Without a server, the viewer says so and offers an **Open raw** fallback button.

## Style

Single static HTML in the team's sidebar-briefing style (same DNA as the PusoPay guide and the
face-id-matcher deck): sticky numbered agenda sidebar with scroll-spy and progress bar, cover
hero with stat cards, section kickers, endpoint blocks, Prev/Next pager plus arrow-key
navigation, and light/dark theme toggle. All CSS and JS are inline — one file to share.

## What's inside

| Section | Source of truth it distills |
|---|---|
| Problem, Solution | `docs/cps-272/TDD-CPS-272-Self-Service-Registration-KYC.md` §1–§4 |
| As-Is vs To-Be, Architecture, Key Flows | `docs/cps-272/02-architecture-diagrams.md` |
| Use Cases | `docs/cps-272/08-use-cases.md` (full UAT scripts) |
| API, Data Model | `docs/cps-272/03-api-design.md`, `docs/cps-272/04-data-model.md` |
| Glossary | `docs/cps-272/07-glossary.md` (abridged to 15 terms) |
| Tickets (collapsible, the centerpiece) | `docs/cps-272/05-user-stories-tickets.md` (full Jira fields) |
| Rollout, Risks, Next Steps | TDD §10–§14 |
| Rollout detail (phases, decisions, checklists) | `docs/09-rollout-plan.md` |

## Local planning board (Tickets section)

Each ticket has an assignment block: **assignee** (with a shared team list), **estimate**
(prefilled from the deck, editable), and **notes/comments** — all autosaved to the browser's
local storage with a saved timestamp and an "N of 21 assigned" counter. **+ Add ticket**
creates local tickets (same badges, same assignment block, deletable). **Export JSON**
downloads `{team, assignments, customTickets}` for Jira creation. **Import JSON** restores
such a file (replaces the local board after confirmation). Nothing leaves the browser.

## Keeping it in sync

This deck is a **distillation, not a fork**: ticket titles, points, priorities, and endpoint
names must match `05-user-stories-tickets.md`. If a ticket changes there, update the matching
`<details class="ticket">` block here. Diagrams are simplified copies — normative versions
stay in `02-architecture-diagrams.md`.
