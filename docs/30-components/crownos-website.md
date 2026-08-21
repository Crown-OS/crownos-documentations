# crownos-website

**Status: Early** · TypeScript · default branch `main` ·
[repo](https://github.com/Crown-OS/crownos-website)

The CrownOS landing page. Next.js App Router, React 19, Tailwind CSS v4, Biome,
built and run with Bun.

The site builds and ships. Its **copy is placeholder** and contradicts the code
in several places — see [Content accuracy](#content-accuracy).

---

## Stack

| | |
|---|---|
| Framework | Next.js 16.2.4, App Router |
| React | 19.2.4, with the React Compiler enabled (`reactCompiler: true`) |
| Styling | Tailwind CSS v4 — `@import "tailwindcss"` plus `@theme` tokens in `globals.css` |
| Lint / format | [Biome](https://biomejs.dev/) 2.2.0. No ESLint, no Prettier. |
| Package manager | **Bun** — the lockfile is `bun.lock` |

---

## Build and run

```bash
cd crownos-website
bun install
bun run dev       # http://localhost:3000
bun run lint      # biome check
bun run format    # biome format --write
bun run build     # next build — also type-checks
```

Use Bun. npm or yarn will create a competing lockfile.

> There is **no `check-types` script** and **no test framework** in this repo.
> Type errors surface through `bun run build`.

### Biome configuration

From `biome.json`:

- Formatter: spaces, indent width 2
- Linter: recommended rules, plus `next` and `react` domains at "recommended"
- `suspicious.noUnknownAtRules` is **off** — required for Tailwind v4's `@theme`
  and `@apply`
- Import organisation on
- VCS integration on, honouring `.gitignore`
- Excludes `node_modules`, `.next`, `dist`, `build`

---

## Structure

```
src/
  app/
    page.tsx            /
    changelog/          /changelog
    community/          /community
    docs/               /docs
    download/           /download
    features/           /features
    globals.css         Tailwind v4 tokens, monochromatic palette
  components/
    landing/            Navbar, HeroSection, MarqueeSection, PrinciplesSection,
                        AIShowcaseSection, EcosystemSection, ArchitectureSection,
                        TestimonialsSection, FAQSection, DownloadSection, Footer,
                        SectionHeader, styles.ts, subpage-styles.ts, types.ts
    icons.tsx
  data/
    landing-page.ts     ecosystemFeatures, principles, heroMetrics, marqueeItems
```

Content is data-driven from `src/data/landing-page.ts`, so copy changes usually
mean editing that file rather than the components.

The palette is defined as tokens in `globals.css`: pitch `#050505` through a grey
scale, `--color-accent: #ffffff`, with a light-theme override under
`prefers-color-scheme`.

---

## Design conventions

This is the **only repository with an `AGENTS.md`**, and it is the project's
design brief. `CLAUDE.md` is a single line importing it. Key points, in the
author's words:

- *"Do Low level design before writing code. follow solid principles. write
  maintainable modular code without too much comments."*
- *"Always maintain a consistent design language and coding style across the
  project. create smooth transitions and parallax effects across the sections."*
- *"we have a unique, beautiful, minimalistic and professional monochromatic ui
  and gives a super stable user experience without compromising on the
  customizability."*
- Vision: *"we care about performance, open source and consumerism."*
- Ecosystem features: Clipboard Sync, Camera Share, Remote Phone Access, Quick
  File Sharing, Notification Sync, Second Screen, Instant Hotspot, Call Bridge
- UX reference: [lawsofux.com](https://lawsofux.com/). UI inspirations:
  giganticmedia.net (fonts and animation, without the red accents),
  tailwindcss.com/plus (type, spacing), stripe.com (layout), coreicon.dev (icons)

> Note the "without too much comments" guidance is scoped to this repository. It
> conflicts with the Rust house style, where `crownos-config` and
> `crowndictator` — the best-regarded crates in the project — carry substantial
> explanatory module docs. Match the repository you are in.

---

## Content accuracy

The site's copy is ahead of the software, in some places substantially. These are
known and should be corrected on the site rather than reflected in these docs.

| Site claims | Reality |
|---|---|
| "TOML profiles" | Configuration is **RON** |
| "Hyprcrown" compositor | The compositor is `crownpositor` |
| A `crownos` CLI (`sync`, `snapshot`, `rollback`, `pair`, `ask`) | No such binary exists in any repository |
| btrfs/snapper atomic rollback, ggml/ONNX AI runtime, BORE-EEVDF scheduler | None ship in the ISO profile |
| Three ISO editions and five download mirrors | One unbranded x86_64 Arch rescue image; no mirrors |
| "12.4k GitHub stars", "8.2k Discord members", "320+ curated packages", "320+ contributors" | Placeholder values |

### The `/docs` route

`src/app/docs/page.tsx` renders eight cards — Getting started, The shell, AI
runtime, Ecosystem features, Theming, Privacy & security, Contributing,
Reference — each with four sub-links.

**Every one of those 32 links points at `/docs` itself.** The "Open the SDK
reference" call to action does too.

That route is the gap this documentation repository fills. Wiring `/docs` to
these pages is worthwhile work.

---

## Other known issues

- **`README.md` is unmodified `create-next-app` boilerplate.** It describes
  npm/yarn/pnpm, the Geist font and Vercel deployment, and says nothing about
  CrownOS or the actual Bun + Biome workflow.
- No tests, no test framework.
- A stale branch, `cloudflare/workers-autoconfig`, exists alongside `main`.

---

## License

**No LICENSE file.** See
[Project status](../00-overview/project-status.md#licensing).
