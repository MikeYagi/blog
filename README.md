# Blog EA1PYM

Hugo + tema Clean White (copiado en `themes/`). Lo publica Cloudflare Pages en https://ea1pym.pages.dev cada vez que se hace push a `main`.

Configuración en Cloudflare Pages:
- Framework preset: Hugo
- Build command: `hugo --gc --minify`
- Build output directory: `public`
- Variable de entorno: `HUGO_VERSION` = `0.170.1` (el tema necesita Hugo ≥ 0.141)

Uso:
- Nuevo artículo: `hugo new post/AAAA-MM-DD-titulo.md` (nace como borrador: quita `draft: true` para publicarlo).
- Vista previa local: `hugo server -D` y abrir http://localhost:1313
- Imágenes de un artículo: en `static/img/` o junto al .md.
- Artículos del homelab: revisar que no queden IPs, MACs, nombres, dominios .lan ni tokens antes de quitar `draft`.
