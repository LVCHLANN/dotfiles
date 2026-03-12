# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Build Commands

```bash
# Build (with tests)
mvn -B -ntp clean package

# Build (skip tests)
mvn -B -DskipTests package
```

The build produces a shaded JAR in `target/`. SnakeYAML is bundled and relocated to `com.discordlogger.shaded.snakeyaml`. There are no unit tests — validation is done via live server testing.

## Architecture

DiscordLogger is a Paper/Spigot plugin (Java 21, Paper 1.21+) that logs Minecraft server events to Discord via webhooks, supporting both plain text and rich embed messages.

### Plugin Lifecycle

`DiscordLogger.java` (main class) — `onEnable()` runs config migration, initializes `Log`, registers listeners via `EventRegistry`, sets up commands, runs the update check, and fires the server start event. `onDisable()` fires the server stop event.

### Core Subsystems

**`Log.java`** — Central static hub for all logging. Uses volatile fields for async-safe reads. Three main methods:
- `plain()` — plain text message
- `event()` — categorized event (embeds or plain text based on config)
- `eventWithThumb()` / `eventFieldsWithThumb()` — embed with player avatar / structured fields

**`DiscordWebhook.java`** — Pure static HTTP client using Java 21's `HttpClient`. No external JSON library — escaping is handled manually. Dispatches async via Bukkit scheduler; falls back to sync on plugin disable.

**`EventRegistry.java`** — Registers all 16 listeners in one place (`registerAll()`). Also exposes `fireServerStart()` / `fireServerStop()` for manual triggers.

**`ConfigMigrator.java`** — Detects config version via comment trailer (`# CONFIG VERSION V<N>`), flattens both old and new YAML to dotted-path maps, preserves user values for surviving keys, rotates old file to `config.old.yml`.

**`Commands.java`** — Routes `/discordlogger` (aliases: `dlogger`, `dlog`) to `Subcommand` implementations. Currently only `reload`.

### Listeners

16 listeners across three packages — `listener/player/`, `listener/server/`, `listener/moderation/`. All use `@EventHandler(priority = EventPriority.MONITOR, ignoreCancelled = true)`.

Complex ones worth reading before contributing: `Ban.java` (parses command args, delays to next tick), `PlayerDeath.java` (resolves killer type).

### Adding a New Event

1. Create listener in `src/main/java/com/discordlogger/listener/<category>/`
2. Register it in `EventRegistry.registerAll()`
3. Add a boolean toggle to `config.yml` under the appropriate `log.*` section
4. Call the appropriate `Log.*` method

### Modifying Config

Increment the version in the comment trailer (`# CONFIG VERSION V<N+1>, SHIPPED WITH V<version>`). `ConfigMigrator` auto-migrates user configs on next startup.

### Key Conventions

- Volatile fields in `Log.java` for async-safe access without locks
- Color map rebuilt locally then assigned atomically
- `Names.java` caches UUID→nickname; `display()` returns `"Nick (Real)"` format when nicknames differ
- `Icons.java` holds shared icon URL constants
- Plugin degrades gracefully (console-only logging) if webhook URL is missing/invalid — no restart needed after fixing config and running `/dlog reload`

## Planned Work

### Config Generator Rewrite (`docs/assets/js/generator.js`)

The current file is a 758-line monolith with 660+ lines of inline CSS. Plan is to split into a modular step-based system.

**New structure:**
```
docs/assets/
  css/generator.css
  js/generator/
    main.js               ← entry point, step registry array
    state.js              ← shared state + localStorage persistence (30-day expiry)
    router.js             ← step navigation, progress bar, isSkipped() logic
    loader.js
    yaml-builder.js       ← token-replaces the version's config.template.yml
    yaml-importer.js      ← regex-based import of existing config.yml, no new deps
    discord-preview.js    ← live CSS embed preview
    datetime-validator.js ← Java DateTimeFormatter validation
    steps/
      step-version.js     ← sets active config version for all subsequent steps
      step-webhook.js
      step-import.js
      step-style.js
      step-events.js      ← data-driven from options.json
      step-colors.js      ← data-driven from options.json, auto-skipped if plain text
      step-output.js
```

**Core step interface** — each step is a plain JS object:
```js
{ id, title, render(), onEnter(), onExit(), isSkipped() }
```

**Version separation (critical):** The generator supports multiple config versions and this must be preserved. Steps are data-driven from the selected version's `options.json` and `config.template.yml` — nothing hardcoded in step files. `step-version.js` sets which version's data loads for all subsequent steps. Adding v10 = new `docs/assets/configs/v10/` folder + entry in `generator.config.js`, no step changes needed.

**New features to add:** drag-and-drop config import, live Discord embed preview, Java DateTimeFormatter live validation, localStorage save/resume, better webhook error messages, clickable progress bar, auto-skip colors step when plain text selected.

**Technical constraints:** native ES modules (no bundler), no new dependencies, no build step, `generator.config.js` stays unchanged as the global config contract.

**Suggested start order:** `state.js` → `router.js` → one sample step to validate architecture → remaining steps.

### Nested Config Sub-Options Pattern

Currently all event toggles are flat booleans (`log.player.teleport: true`). Need a pattern for sub-options — additional settings that only apply when the parent event is enabled (e.g. a coords toggle under teleport).

**Everything that needs to change when adding a sub-option:**

- `config.yml` / `config.template.yml` — add sub-key under parent (e.g. `teleport-coords: true`), add `{{TOKEN}}` placeholder in template
- `ConfigMigrator` — bump config version, new key migrates with a sensible default (usually `true` so existing installs don't change behaviour)
- Listener (Java) — read sub-key only after parent toggle guard passes
- `options.json` — needs schema extension: `suboptions: [{ id, label, configKey, default }]` on parent items that have them
- Generator UI (`step-events.js`) — sub-options render indented beneath parent, disabled/hidden when parent is off

Sub-options only appear for config versions whose `options.json` includes the `suboptions` field — older versions degrade gracefully.

## Branch Strategy

- `main` — production, updated via PR from `dev`
- `dev` — development/staging
- `feat/*, ref/*, bug/*` — feature branches, created from `dev`, PRed back to `dev`

## CI/CD

- **CI** (`.github/workflows/ci.yml`): Triggers on push/PR to `main`/`dev`, builds with Temurin JDK 21
- **Release** (`release-on-merge.yml`): Triggers on merge to `main`, uses `.github/release-spec.md` for notes, publishes GitHub Release with JAR