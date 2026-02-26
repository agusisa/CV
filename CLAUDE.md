# Agustin Diez Martinez — Personal CV

> **REGLA PARA CLAUDE CODE**: Este archivo es tu fuente de verdad sobre el proyecto. Cada vez que hagas cambios significativos (nuevos archivos, cambios de arquitectura, cambios de deploy, nuevas dependencias, nuevas configuraciones, bugs resueltos), **actualizá este archivo** al final de la sesión antes de terminar. Esto incluye:
> - Agregar archivos nuevos al "Project Structure"
> - Actualizar conteos de líneas si cambiaron mucho
> - Documentar nuevos patrones o convenciones que se introdujeron
> - Registrar cambios de infraestructura (DNS, Caddy, paquetes)
> - Actualizar la sección "Change Log" al final con fecha y resumen
>
> Si no estás seguro de si hubo un cambio relevante, actualizalo igual. Es mejor tener contexto de más que de menos.

---

## Project Structure

```
CV-web.html        Main web CV — self-contained HTML with inline CSS/JS
CV-export.html     PDF export version (uses html2pdf.js for direct download)
CV.md              Markdown source of the CV content
avatar.jpg         Profile photo
assets/            Company logos (dualboot, trenda, overactive, xoor, technisys)
LINKEDIN_UPDATES.md  LinkedIn recommendation quotes used in testimonials
CLAUDE.md          Project context file for Claude Code (this file)
.gitignore         Git ignore rules
.claude/settings.json         Claude Code project settings
.claude/commands/update-context.md  Custom /update-context slash command
```

## Architecture

Both `CV-web.html` and `CV-export.html` are fully self-contained — no build step, no framework, no external CSS. Everything is inline in a single HTML file each.

### CV-web.html (~2150 lines)

The main web version includes:

- **Theming**: Two themes (dark/warm) via CSS variables and `data-theme` attribute. Dark is default. Persisted in `localStorage`.
- **i18n**: Full internationalization in English, Spanish, and Portuguese. A `translations` JS object with 92 keys per language. Language toggle button cycles EN → ES → PT. Persisted in `localStorage`. All translatable elements have `data-i18n` attributes and are updated via `innerHTML`.
- **Layout**: Sticky sidebar (340px) + scrollable main content + fixed vertical timeline. On mobile (≤1024px), sidebar scrolls naturally above content, timeline is hidden.
- **Scroll sync**: On desktop, the sidebar scrolls proportionally with main content via `requestAnimationFrame`.
- **Timeline navigation**: Fixed vertical rail with clickable year nodes. Active state tracks scroll position via `IntersectionObserver`.
- **Canvas animations**: Two layered effects on the timeline (Knight Murmurations + Data Tunnel) and a nebula effect behind the CTA — all using Canvas 2D API, all very subtle.
- **Testimonials**: LinkedIn recommendations that fade in on scroll.
- **Clipboard**: Email and phone are copyable with a "Copied!" toast animation.
- **Responsive**: Full mobile adaptation at 1024px breakpoint.

### CV-export.html (~1150 lines)

Simplified version optimized for PDF generation:
- Uses `html2pdf.js` (loaded from CDN) to generate and download a PDF on page load
- Stripped of animations, scroll sync, and interactive features
- Visually aligned with the web version's styling
- **i18n-aware**: Reads `?lang=es` or `?lang=pt` from URL params and applies translations before PDF generation. The download link in `CV-web.html` automatically appends the current language.
- Includes the same `translations` object and `data-i18n` attribute pattern as the web version

## Key Conventions

- No build tools, bundlers, or frameworks — everything is vanilla HTML/CSS/JS
- CSS variables for all colors, shadows, borders, and radii
- Glassmorphism aesthetic: `rgba()` backgrounds + `backdrop-filter: blur()`
- All SVG icons are inline (no icon library dependencies)
- Company logos loaded from `assets/` or `logos-api.apistemic.com` with `onerror` fallback

## Editing Guidelines

- When modifying text content, update all three languages in the `translations` object (en/es/pt) and ensure the `data-i18n` key exists on the HTML element.
- Theme colors are defined in four places: `:root` (default warm), `@media (prefers-color-scheme: dark)`, `[data-theme="light"]`, and `[data-theme="dark"]`.
- Canvas animation parameters (opacity, particle count, alpha, spread) are intentionally minimal — keep them subtle.
- The `CV-export.html` content should stay in sync with `CV-web.html` for the experience descriptions and AI statements.

---

## VPS / Deployment

### Infrastructure

| Component | Detail |
|-----------|--------|
| **VPS** | Vultr, Ubuntu 22.04 (Jammy), IP: `216.238.115.79` |
| **Web server** | Caddy 2.10.2 (auto HTTPS via Let's Encrypt) |
| **Domain** | `agustindiezcv.com.ar` (registered at NIC.ar) |
| **DNS** | Cloudflare (free plan), nameservers: `dee.ns.cloudflare.com`, `kyle.ns.cloudflare.com` |
| **Repo** | Private GitHub repo `agusisa/CV`, cloned to `/var/www/cv` via fine-grained PAT (read-only) |
| **Node.js** | v20.20.0 (for Claude Code CLI) |
| **Claude Code** | Installed globally via npm |

### Caddy Configuration

File: `/etc/caddy/Caddyfile`

```
agustindiezcv.com.ar {
    root * /var/www/cv
    file_server
    encode gzip
    rewrite / /CV-web.html
}

www.agustindiezcv.com.ar {
    redir https://agustindiezcv.com.ar{uri} permanent
}
```

### Deploy Workflow

```bash
# From VPS:
deploy-cv    # alias for: cd /var/www/cv && git pull

# Or manually:
cd /var/www/cv && git pull && sudo systemctl restart caddy
```

### DNS Setup (Cloudflare)

- A record: `@` → `216.238.115.79` (DNS only, grey cloud)
- A record: `www` → `216.238.115.79` (DNS only, grey cloud)
- NIC.ar delegated to Cloudflare nameservers
- Status: Waiting for NIC.ar propagation (configured 2026-02-20)

### Troubleshooting

```bash
# Check DNS propagation
dig agustindiezcv.com.ar +short

# Check Caddy status
sudo systemctl status caddy
sudo journalctl -u caddy --no-pager -n 50

# Validate Caddyfile
caddy validate --config /etc/caddy/Caddyfile --adapter caddyfile

# Test local serving
curl -s -o /dev/null -w "%{http_code}" http://localhost

# Check ports
sudo ufw status
ss -tlnp | grep -E ':80|:443'

# Server public IP
curl -s ifconfig.me
```

### Adding Another Site

To serve another domain from this VPS, add a new block to the Caddyfile:

```
otherdomain.com {
    root * /var/www/other-site
    file_server
}
```

Caddy automatically obtains SSL certificates for each domain.

---

## Change Log

| Date | Summary |
|------|---------|
| 2026-02-20 | Initial project setup: CV-web.html with full i18n (EN/ES/PT), theming, canvas animations, testimonials. CV-export.html with PDF generation. VPS configured on Vultr with Caddy. Domain `agustindiezcv.com.ar` registered, DNS delegated to Cloudflare. Waiting for NIC.ar propagation. |
| 2026-02-20 | Diagnosed DNS non-propagation (NXDOMAIN — NIC.ar still pending). Rotated GitHub PAT (old token was exposed in git remote URL). Pulled latest commit (99ad7eb: About section + i18n updates). Updated CLAUDE.md with new project files (.claude/, .gitignore, CLAUDE.md). |
