# jkhowland.com

Personal website for Joshua Howland. Plain static HTML/CSS — no build tools, no frameworks.

## Running Locally

- `python3 -m http.server 8000` then open http://localhost:8000
- Or use `/dev` to start the server and open the browser

## Site Structure

- `index.html` — Homepage / blog post listing
- `about.html` — About page
- `projects.html` — Projects page
- `css/style.css` — All styles
- `images/` — Image assets
- `posts/` — Individual blog post HTML files (when added)

## Design

Preserved from the original jkhowland.me blog:
- Blue top bar and accent color: `#3c9def`
- Gold header border and link accents: `#ffc328`
- System font stack, large readable type
- Responsive — hamburger menu on mobile (pure CSS, no JS)
- Max width 920px, sidebar on desktop

## Adding Content

- Use `/new-post "Title"` to scaffold a new blog post
- Blog posts are standalone HTML files in `posts/`
- Add post links to `index.html` manually
- Each page shares the same header/footer markup — keep them in sync

## Conventions

- No JavaScript unless absolutely necessary
- No build step — everything ships as-is
- Keep pages self-contained (shared markup is copy-pasted, not templated)
- Mobile-first responsive design
