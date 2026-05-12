# DilliDekho — heritage walks site

Static HTML/CSS/JS site for DilliDekho. Built lean so we can deploy directly (Vercel / Netlify / GitHub Pages) **or** use it as a visual blueprint to rebuild in Framer.

## Structure

```
dillidekho/
├── index.html              # Home: hero, walks, why, how, proof, contact
├── blog.html               # Blog listing (cards grid)
├── blog-post.html          # Sample post: "Djinn letters at Firozshah Kotla"
├── styles/
│   └── main.css            # Single stylesheet — design tokens at top
├── scripts/
│   └── main.js             # Mobile nav toggle only
└── README.md
```

No build step. No dependencies. Open `index.html` in a browser and it works.

## Design system

- **Palette**: Mughal terracotta (`#A0421A`), warm cream (`#FAF4EC`), warm ink (`#1F1B16`), old gold (`#C9A24B`)
- **Type**: Fraunces (display, Google Fonts) + Inter (body) + Mukta (Devanagari accents)
- **Motifs**: 8-point Mughal star (logo + dividers), jali lattice (subtle bg), pointed arches (inside card art), monument silhouettes (Qutub, Humayun, Firozshah)
- **No Persian/Urdu script** — English + occasional Devanagari only

All colors and type sizes are CSS custom properties at the top of `styles/main.css` — change once, propagates everywhere.

## Preview locally

```powershell
# Any static server works. Quickest:
cd dillidekho
python -m http.server 8080
# Open http://localhost:8080
```

Or just double-click `index.html` (works without a server, except some font features may load slightly differently).

## Deploy

**Vercel** (recommended, ~3 minutes):
1. Push this repo to GitHub
2. Import the repo in Vercel
3. Set "Root Directory" to `dillidekho/`
4. Deploy. Point `dillidekho.in` DNS at the Vercel CNAME. (`dillidekho.com` is owned and redirects to `.in` — `.in` is the canonical domain.)

**Netlify** (drag-drop):
1. Zip the `dillidekho/` folder
2. Drag onto netlify.com/drop
3. Custom domain in dashboard

**GitHub Pages**:
1. Move site files to `/docs` or repo root of a new repo
2. Settings → Pages → main branch
3. Free SSL, custom domain supported

## Scaling the blog

The blog is rendered as static HTML right now. Three scale paths when post count gets meaningful:

1. **Keep static, copy `blog-post.html`** for each new post. Manual but fastest. Good through ~20 posts.
2. **Move to a static site generator** (Astro / 11ty / Hugo) — same HTML/CSS, but posts written as Markdown. Recommended at 20+ posts.
3. **Rebuild in Framer** with their CMS — visual editor, non-dev-friendly for adding posts. This site's design is intentionally one Framer artisan can re-create in a day.

## Editing the home page copy

Hero copy: `index.html` → search `<section class="hero">`
Walk cards: search `<article class="walk-card">` (three of them)
SKU pricing/duration: inside `.walk-meta` divs
Why us features: search `<section id="why">`
Contact email/WhatsApp: search `mailto:hello@dillidekho.in` and `wa.me/`

Replace `919999999999` in the WhatsApp link with the real number when assigned.

## Next milestones

- [x] Buy `dillidekho.in` (canonical) and `dillidekho.com` (redirect); point DNS
- [ ] Deploy to Vercel
- [ ] Replace SVG monument silhouettes with real photography (Unsplash interim, then your own from pilot walk)
- [ ] Write blog posts 2 and 3 (Iron Pillar, Heritage Walks vs Bowling)
- [ ] Add a real Calendly link to the contact CTA
- [ ] Reserve the WhatsApp Business number
- [ ] Run pilot walk; replace the empty social-proof block with quote + photos
