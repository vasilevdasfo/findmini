# findmini.org

Lost cat — Mini. Pacifica / Daly City, California. $500 reward.

Live: https://findmini.org

## Edit the site

Single file: `index.html` (no build step). Change text → commit → GitHub Pages auto-deploys in ~30 sec.

## Photos / videos

`media/` — replace files keeping the same names, or add new ones and reference them in `index.html`.

## Hosting

- GitHub Pages, custom domain `findmini.org` (CNAME file enforces it)
- DNS at registrar:
  - `A` → `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153` (apex)
  - `CNAME www` → `vasilevdasfo.github.io`
- HTTPS auto-issued by GitHub (Let's Encrypt)
