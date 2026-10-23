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

## Verla en local

Hay que servirla por HTTP, no abrir el archivo:

```bash
python -m http.server 8000
```

Abrir `index.html` directamente deja la sección de registro **sin cohete**. El
modelo se pide con XHR a `rocket.glb` y el navegador bloquea esa petición en
`file://`. El resto de la página sí se ve bien abriendo el archivo; si falta el
cohete, la consola lo avisa.

## Estructura

```
index.html    la página entera
rocket.glb    el cohete del cierre (18,5 MB, carga diferida)
logo.png      el wordmark
vercel.json   cabeceras y cleanUrls
CLAUDE.md     contexto para Claude Code
```

## Antes de publicarlo de verdad

El contenido del evento es provisional. Falta: link del RSVP, fecha
definitiva, los seis releases reales, el line-up y la dirección exacta.
La lista completa está en `CLAUDE.md`.

⚠️ `.env.local` lleva un token de Vercel. Está ignorado por git. No lo subas.
