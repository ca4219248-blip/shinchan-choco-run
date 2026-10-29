# Shinchan Choco Run

A single-file HTML5 canvas platformer that runs entirely in the browser. No build step, no dependencies — open the page and play.

**Play:** https://ca4219248-blip.github.io/shinchan-choco-run/

## Controls

- Desktop: Arrow keys / A-D to move, Space or Up to jump, P or Esc to pause.
- Mobile: on-screen left / right / JUMP buttons (shown automatically on touch devices).

## What's in the game

- **20 levels**, generated from a fixed seed per level — every player sees the same level 7, but the layout is procedural rather than hand-placed. Difficulty scales with the level number: gaps get wider, moms throw pans faster, and their walk speed rises.
- **5 visual themes** that cycle by level — daylight, sunset, night with stars, pink, and mint — each with its own sky gradient and colour overlay.
- **Level select grid** showing all 20 levels, locked ones greyed out. Progress unlocks two levels at a time.
- **Question blocks**: hit one from below for 50 points, and sometimes for the Action Kamen power-up — a short invincible dash form that also ignores moms.
- **Secret**: staying safe for 600 ticks also triggers the power-up once per save, and marks the save with the hidden character.
- Mom throws pans, trash bins and gaps hurt, 3 hearts per life, 25 chips = 1 extra life.

## Cloud save (Supabase)

Progress is written in two places:

1. `localStorage` — instant, works offline.
2. Supabase Postgres — a cloud backup keyed by a **sync code** the game generates for the player.

On boot the game merges the local save with the cloud copy (best of both: highest score, furthest level, most chips), so progress survives a site update, a cache clear, or a switch to another device.

### How players use it

Open **Cloud Sync** from the main menu:

- The sync code (format `TFG-XXXXX-XXXXX-XXXXX-XXXXX-XXXXX`) is displayed. Copy it somewhere safe — it is the only key to the cloud save.
- **Restore From Code** pulls a cloud save onto a new device.
- **New Code** starts a fresh cloud identity.
- **Sync Now** forces a push.

### Database design

Table `public.player_saves`:

| column | type | notes |
| --- | --- | --- |
| `sync_code` | `text` | primary key, 20–64 chars |
| `data` | `jsonb` | `{hi, un, chips, mus, sfx, ak, v}` |
| `created_at` / `updated_at` | `timestamptz` | audit timestamps |

Row level security is **on** and there are no RLS policies, so the table cannot be read or written directly with the public API key. Access happens only through two `security definer` RPCs:

- `load_game(p_code text) returns jsonb`
- `save_game(p_code text, p_data jsonb) returns jsonb`

Both reject codes shorter than 20 characters, and `save_game` rejects non-object payloads and anything over 16 KB. The sync code is a 100-bit random secret generated with `crypto.getRandomValues`, so a save cannot be reached without it.

The browser only ever holds the project's publishable (public) API key — never a service key.

## Files

- `index.html` — the whole game, including the Supabase client calls.

## Notes

- Sound is generated with the Web Audio API; nothing is downloaded at runtime except the Supabase requests.
- If the cloud is unreachable the game keeps saving locally and shows `Cloud: offline` — no progress is lost.
- Save schema is versioned (`v: 2`). Older saves without a field still load, because every field falls back to a default.
