# Fina Releases — landing

Página única del evento de lanzamiento de Fina. Sin build ni dependencias:
todo está en `index.html` (CSS y JS inline).

**Producción:** https://fina-releases-fina-b5773c0d.vercel.app

## Empezar

```bash
vercel deploy --prod --yes --scope fina-b5773c0d
```

Para editar, abre la carpeta con Claude Code: `CLAUDE.md` tiene el contexto
completo — design system, cómo funcionan las tres piezas animadas y qué
contenido sigue siendo provisional.

Para ver la página en local basta con abrir `index.html` en el navegador.

## Estructura

```
index.html    la página entera
vercel.json   cabeceras y cleanUrls
CLAUDE.md     contexto para Claude Code
```

## Antes de publicarlo de verdad

El contenido del evento es provisional. Falta: link del RSVP, fecha
definitiva, los seis releases reales, el line-up y la dirección exacta.
La lista completa está en `CLAUDE.md`.

⚠️ `.env.local` lleva un token de Vercel. Está ignorado por git. No lo subas.
