# Generative Profile / Resume — Initial Plan

## Purpose

Build a Linux-distributed Go application that turns a Markdown-based professional profile into an interactive, generative website. Visitors can ask natural-language questions about the profile; the application returns useful answers through an engaging UI assembled from reusable and query-specific UI assets.

## Initial repository layout

```text
.
├── assets/
│   ├── shared_assets/  # Reusable, reviewed UI components and common resources
│   ├── cached_assets/  # Generated, query-keyed UI assets safe to reuse
│   └── main_assets/    # Core application-owned frontend assets and layouts
├── resume/             # Resume, bio, projects, and supporting profile data in Markdown
└── INITIAL_PLAN.md
```

### Asset ownership

| Directory | Intended contents | Cache behavior |
| --- | --- | --- |
| `assets/shared_assets/` | Cards, timelines, skills visualizations, icons, common partials, shared scripts/styles | Versioned with the application; never generated per visitor request |
| `assets/cached_assets/` | Generated HTML fragments, CSS, JSON metadata, and optional media for a classified request | Read/write at runtime, organized under a deterministic cache key |
| `assets/main_assets/` | Base layouts, Tailwind source/output, Alpine components, global styles, animation definitions | Versioned with the application |
| `resume/` | Canonical source facts: resume, experience, projects, skills, contact information, preferences | Read by the backend as generation context; treated as the source of truth |

## Application architecture

### Go backend

The compiled executable should own the complete server lifecycle:

1. Load and validate Markdown files from `resume/` at startup (and optionally reload them in development).
2. Serve the initial HTML document plus static assets.
3. Handle normal page requests and HTMX partial requests.
4. Classify each visitor question into a strict, canonical query tag.
5. Resolve a cache key from that tag and generation-input version.
6. Return a validated cached result when available; otherwise invoke the model, validate and persist the generated assets, then return them.
7. Enforce rate limits, request size limits, logging, and safe error fallbacks.

Suggested Go packages when implementation begins:

```text
cmd/gen-ui/              # executable entry point
internal/server/          # routing, templates, HTTP/HTMX handling
internal/profile/         # Markdown parsing and profile context
internal/classify/        # tag schema and model-backed classification
internal/cache/           # key derivation, filesystem storage, expiry, locking
internal/generate/        # model calls and structured generated-output validation
internal/assets/          # shared/main asset discovery and serving
web/                      # server-rendered templates and Tailwind/Alpine sources
```

## Request and cache flow

```text
Visitor question
      ↓
Input validation and normalization
      ↓
Strict question classifier → canonical query tag
      ↓
Cache-key builder (tag + profile version + generator/UI version)
      ↓
Cached entry found? ── yes → validate entry → return HTMX fragment
      │
      no
      ↓
Generate structured answer + permitted interactive asset specification
      ↓
Validate, sanitize, persist atomically under assets/cached_assets/<cache-key>/
      ↓
Render and return HTMX fragment
```

### Query tagging contract

The classifier must produce a bounded schema rather than free text. A starting shape:

```json
{
  "intent": "experience | skills | projects | education | contact | availability | summary | other",
  "subject": "canonical-slug-or-general",
  "audience": "recruiter | collaborator | visitor",
  "format": "answer | timeline | cards | comparison | highlight",
  "scope": "specific | overview"
}
```

The backend validates every field against allow-lists, serializes the normalized object with stable field ordering, and hashes it. Cache identity should also include:

- a `profile_version` hash of relevant Markdown content;
- a `generation_schema_version` for prompt/output changes;
- a `ui_asset_version` for shared component changes;
- the model/provider identifier if outputs may differ materially.

This prevents old answers from surviving a resume or UI update while still allowing semantically equivalent questions to reuse work.

### On-disk cache entry

```text
assets/cached_assets/<cache-key>/
├── metadata.json        # tag, versions, timestamps, expiry, validation status
├── response.json        # structured model output
├── fragment.html        # sanitized HTMX-ready rendered fragment
└── assets/              # optional generated assets permitted by policy
```

Writes should use a temporary directory and atomic rename. A per-key lock or single-flight mechanism avoids duplicate model calls when identical requests arrive concurrently. Entries should support a configurable TTL and maximum total size; expired or invalid entries are regenerated.

## Frontend direction

- **HTMX:** send question forms to Go endpoints and swap returned fragments into the conversation/profile stage. Prefer server-rendered HTML for the content users see.
- **Alpine.js:** keep local interaction small and progressive—dismissible panels, expanded cards, animation state, theme controls, and keyboard behavior.
- **Tailwind CSS:** compile a local production stylesheet. Use it for layout and tokens; reserve custom CSS for branded effects and animations that do not fit utilities well.
- **Custom components:** build a small, accessible component vocabulary in `shared_assets/` (answer card, project card, timeline, skill cluster, empty state, loading state, error state).
- **Accessibility:** semantic controls, focus management after HTMX swaps, reduced-motion support, keyboard access, color contrast, and no essential information conveyed only through animation.

Generated output should select from approved components and data fields—not emit arbitrary executable JavaScript or unreviewed HTML. This keeps the interactive experience safe, consistent, and cacheable.

## Profile content plan

Keep the resume content split into focused Markdown files, for example:

```text
resume/
├── profile.md           # concise introduction, location/timezone, links
├── experience.md        # roles and achievements
├── projects.md          # project narratives, technologies, outcomes
├── skills.md            # skills and proficiency/context
├── education.md
├── contact.md           # only information intended for public display
└── instructions.md      # voice, factual constraints, and display preferences
```

Use explicit front matter or a small schema for stable identifiers, dates, links, and tags. The generator should cite only supplied profile content, acknowledge uncertainty, and never invent career facts.

## Security and operational baseline

- Keep model credentials in environment variables or an external secret store—never in the executable, cache, or Markdown.
- Set a clear generated-asset allow-list (MIME types, size limits, extension limits) and prevent path traversal.
- Sanitize generated HTML; preferably generate typed data then render trusted server templates.
- Avoid storing personally sensitive visitor questions unless the privacy policy explicitly permits it.
- Provide a graceful non-model fallback answer and a cache status-independent user experience.
- Make cache location configurable for packaged Linux installs, since the executable directory may be read-only.

## Distribution target

The product should build to a single Linux executable with embedded baseline web assets where practical. Runtime-writable data (especially `cached_assets/`) should be configurable via a data directory. Start with a local process listening on a configurable host/port; systemd packaging or container support can follow once the core experience is stable.

## Suggested implementation milestones

1. Scaffold the Go module, executable, configuration, and the directory layout above.
2. Add Markdown profile loading and a conventional server-rendered profile page.
3. Add Tailwind build pipeline, Alpine, reusable components, and an HTMX question form with a deterministic mock responder.
4. Define the query-tag schema and implement normalized cache-key generation with filesystem persistence.
5. Integrate a model for classification and structured generation; add output validation and safe rendering.
6. Add cache expiry, invalidation, concurrency protection, observability, tests, and production packaging.

## Decisions to make next

- Which model provider/API should generate tags and responses?
- Will the site be public, authenticated, or both—and where will it be deployed?
- Which resume/contact details may be publicly exposed?
- Should cached generations be committed for demo use, or always remain runtime data?
- What visual personality should the profile have: restrained editorial, playful/experimental, technical, or another direction?
- What kinds of interaction are in scope beyond answers (timelines, project explorers, skill maps, downloadable resume)?
