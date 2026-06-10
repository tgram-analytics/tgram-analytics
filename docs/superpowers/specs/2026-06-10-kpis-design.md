# KPIs in tgram-analytics — Design

**Date:** 2026-06-10
**Status:** Approved
**Code location:** `server/` (FastAPI backend + Telegram bot)

## Goal

Give every project a set of KPIs surfaced at the top of its section in the `/digest` message:

- **Default web/SaaS KPIs** computed for all projects: unique visitors, sessions, pageviews.
- **Pinned custom KPIs**: events the user pins per project, with one optionally designated as the **North Star metric** and displayed first and most prominently.

KPIs are managed through a "🎯 KPIs" button in the project menu. No new top-level bot command.

## Data model

New table `kpis` (one Alembic migration):

| column | type | notes |
|---|---|---|
| `id` | UUID | primary key |
| `project_id` | UUID | FK → `projects.id`, `ON DELETE CASCADE` |
| `event_name` | text | the pinned event |
| `is_north_star` | boolean, default false | at most one per project |
| `position` | int | display order among non-North-Star KPIs |
| `created_at` | timestamptz | server default `now()` |

Constraints:

- Unique on `(project_id, event_name)` — an event can be pinned once per project.
- Partial unique index on `(project_id) WHERE is_north_star` — at most one North Star per project, enforced at the DB level.

## Service layer

New `app/services/kpis.py`:

- `list_kpis(session, project_id)` — ordered: North Star first, then by `position`.
- `add_kpi(session, project_id, event_name)` — appends at the end; idempotent on duplicates.
- `remove_kpi(session, project_id, event_name)`.
- `set_north_star(session, project_id, event_name)` — clears the previous North Star flag, sets the new one (pins the event first if not already pinned).

## Management UI

New handler module `app/bot/handlers/kpis.py`, following the inline-keyboard conventions of the alerts/funnels handlers.

- Add a "🎯 KPIs" button to the project menu in `app/bot/handlers/projects.py` (`_show_project_menu`), callback data `menu:kpis:<project_id>`.
- KPI screen lists pinned KPIs; the North Star is marked ⭐.
- "➕ Add KPI" lists known event names (via `event_meta_cache` / `list_event_names`) as buttons; tapping one pins it.
- Tapping an existing KPI shows actions: "⭐ Set as North Star", "🗑 Remove", "« Back".
- Standard "« Back" navigation to the project menu.

## Digest KPI block

`_project_digest_lines` in `app/bot/handlers/digest.py` renders each project section as:

```
📦 MyApp
  ⭐ signup: 142  ▲ +18%          ← North Star, always first when set
  👥 Visitors: 3,204  ▲ +5%       ← distinct visitor_hash in period
  👤 Sessions: 4,011  ▲ +3%       ← existing distinct session_id line
  📄 Pageviews: 12,500  ▼ -2%     ← count of pageview events
  🎯 checkout: 89  ▲ +4%          ← other pinned KPIs, position order
  [alerted events as today, excluding any already shown as KPIs]
```

Rules:

- All deltas use the existing `_format_delta` with the current 7d vs prior-7d windows.
- Visitors and Sessions always appear; Pageviews is hidden when zero in both periods (SDK-only projects).
- Pinned KPI counts are fetched in one grouped query (same pattern as the existing alerted-events query) and deduplicated against the alerted-events list.
- When a project has no North Star, append a hint line: `💤 Pin your North Star with the 🎯 KPIs button in /projects`.
- A pinned event with zero recent occurrences still renders, showing 0 and its delta.
- Event names are user-supplied: HTML-escape everywhere, matching existing code.

The block applies wherever `_project_digest_lines` is used; digests are currently on-demand only (`/digest`), so no scheduler changes are needed.

## Out of scope

- KPI targets/goals, computed ratios or formulas, retention/DAU-WAU-MAU metrics.
- Surfacing KPIs in `/overview`, `/report`, or scheduled messages.
- Reordering UI for pinned KPIs (position is set by insertion order).

## Testing

Following existing `server/tests` patterns:

- Service tests: pin/unpin, duplicate pin is idempotent, North Star uniqueness (switching clears the old flag), cascade delete with project.
- Digest tests: section ordering (North Star → defaults → pinned → alerted), dedup of events that are both pinned and alerted, Pageviews hidden when zero in both windows, hint line when no North Star, delta formatting with seeded events.
- Handler tests: KPI menu navigation, add/remove/set-North-Star callbacks.
