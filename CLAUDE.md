# Fina Releases — landing del evento

Página única para el evento de lanzamiento de producto de Fina. Objetivo: registro (RSVP).
Español, público de contadores, founders y equipos de finanzas.

- **Producción:** https://fina-releases-fina-b5773c0d.vercel.app
- **Todo vive en `index.html`.** Sin build, sin framework, sin dependencias instaladas.
  CSS y JS inline. Lo único externo son Google Fonts y Three.js desde cdnjs.

## Desplegar

```bash
vercel deploy --prod --yes --scope fina-b5773c0d
```

El proyecto es `fina-releases` en el equipo `fina-b5773c0d` (ids en `.vercel/project.json`).
No hay CI ni remoto de git conectado: se despliega a mano desde esta carpeta.

Si no tienes acceso a ese equipo, Diego Aponte (diego.aponte@finapartner.com) tiene que
invitarte, o despliegas a tu propio scope y le pasas la URL.

⚠️ **`.env.local` contiene un token OIDC de Vercel.** Está en `.gitignore`. No lo commitees,
no lo pegues en un chat y no lo subas a ningún sitio.

## Reglas que no hay que romper

**`index.html` tiene que seguir siendo un documento HTML completo** — `<!DOCTYPE>`, `<html lang="es">`,
`<head>` con `<meta charset>` y `<meta name="viewport">`, `<body>`. Vercel lo sirve crudo. Sin el
viewport el móvil renderiza a 980px y se descuadra entero.

**Protección de despliegue apagada a propósito.** El proyecto nació con `ssoProtection` y devolvía
302 al login de Vercel. Es una landing pública: tiene que quedar apagada.
`vercel project protection fina-releases --scope fina-b5773c0d` muestra el estado.

## Design system

La paleta y la tipografía salen del sistema real de Fina, no son inventadas.
Fuente: `~/fina-landing/tokens.css` en la máquina de Diego.

```
--fina-purple        #620AFF
--fina-purple-light  #AB5AFF
--fina-pink          #FD37BE
--fina-orange        #FFB470
--color-info         #00CFD5   (estado "en producción")
gradiente de marca   linear-gradient(92deg,#FD37BE 0%,#FFB470 100%)
fondo                blanco, con un velo violeta arriba (#FAF7FF → #FFFFFF)
texto                #14101F
```

**La página es clara.** Fue oscura hasta septiembre de 2026 y se invirtió a petición de
Diego. El histórico importa porque quedan decisiones heredadas de aquello.

⚠️ **El gradiente de marca no sirve como texto sobre blanco.** El extremo naranjo #FFB470
no llega a 2:1 de contraste. Los titulares usan `--grad-text` (violeta → rosa,
#620AFF → #FD37BE); el rosa → naranjo se queda donde es relleno: botones y marquesina.
Lo mismo con los chips: como texto van en `--ink-warm` #B8651A y `--ink-cyan` #00898D.

⚠️ **Nada de mezcla aditiva ni `mix-blend-mode: screen`.** Sobre blanco, screen con
cualquier cosa da blanco, y el aditivo es invisible. Toda la página está en mezcla normal.
Si reaparece un `AdditiveBlending` o un `screen`, ese elemento simplemente no se ve.

Ojo: existe un segundo sistema, `fina-rrss` (el de Claude Design), con los mismos colores
desplazados un dígito — `#6209FF`, `#FD38BE`, `#FFB570`. La página está unificada hacia
`tokens.css`. Si alguien decide que manda el otro, hay que cambiarlos todos, no mezclar.

**Tipografía.** Figtree para todo el lenguaje humano. DotGothic16 (pixel) solo para la capa
máquina: eyebrows `// 01`, versiones `v2.6.0`, chips de estado, horas de la agenda, labels
del contador y footer. La regla es *lo que escribe el sistema va en pixel, lo que escribe una
persona va en Figtree*. Es lo que sostiene el aire techie; si se rompe, la página se vuelve
una landing genérica.

El hero va en Figtree **600**, no 900. A 162px el peso alto empasta las contraformas.
Compensación óptica: a más tamaño, menos peso. Los títulos de sección sí van en 800.

## Las tres piezas animadas

Son lo más frágil del archivo. Cada una tiene invariantes que cuestan caro de redescubrir.

**1. Portada — isotipo a pantalla completa** (`#gate`)
Dos SVG independientes, `#iso-a` y `#iso-b`, uno por barra del isotipo. Están separados **a
propósito**: transformar un `<g>` dentro de un mismo SVG no se compone en GPU y el navegador
rerasteriza los desenfoques en cada frame. Por eso los `defs` están duplicados con ids
sufijados (`fDrop-a` / `fDrop-b`, etc.).

El isotipo está resuelto como **vidrio, no como glow**. Seis capas, en orden: sombra
proyectada (26px abajo, muy difusa) para que flote sobre el blanco; cuerpo translúcido
teñido al 13–22%, que deja pasar el fondo; dispersión cromática —dos copias del contorno
desplazadas en direcciones opuestas, una rosa y otra violeta, desenfocadas— que es lo que
lo hace leer como cristal y no como plástico de color; el canto con el degradado de marca;
un brillo interior erosionado 16px para que el blanco quede dentro del borde; y el especular
desplazado arriba-izquierda. Encima, el destello que recorre la pieza.

No metas relleno oscuro dentro de las barras: eso era la versión con glow y es justo lo que
se pidió quitar.

- El scroll **no pinta**: mueve un objetivo. Un bucle aparte lo persigue con un lerp
  normalizado por tiempo, `1-(1-0.16)^(dt/16.7)`, para que sea igual a 60 y a 120 Hz.
- No usar variables CSS heredadas para el scrub: invalidan el estilo de todo el subárbol.
  Se escribe `transform` y `opacity` directo sobre tres nodos.
- `color-interpolation-filters="sRGB"` en los doce filtros. Sin eso los desenfoques trabajan
  en linearRGB y cuesta el doble.
- El fondo opaco espera al 22% del recorrido antes de disolverse. Si se disuelve desde el
  primer píxel se ve la página antes de que el isotipo se abra y se pierde el efecto.
- La portada lleva `pointer-events:none`. Nunca secuestrar el scroll.

**2. Hero — edificio en alambre** (`#bld`)
Geometría 3D procedural sobre canvas 2D, sin librerías. Cinco cajas: dos alas de 9 pisos, un
cuerpo posterior que deja el patio al frente, zócalo y atrio.

- El temblor de línea es **determinista** (hash del índice). Con `Math.random()` por frame
  titila.
- Las líneas sobresalen 2–7px de cada esquina. Eso es lo que da el aire de plano trazado a
  mano en vez de render limpio; no es un bug.
- Color por posición en pantalla, interpolando el gradiente de marca.

**3. Cierre — cohete 3D** (`#stage`)
Three.js r128 desde cdnjs. La librea se pinta en un canvas 2D y se mapea al cilindro.
Si el CDN falla la sección sigue funcionando: solo desaparece el canvas.

- Despega solo al entrar en pantalla y se repite con clic. El umbral de 6px distingue clic de
  arrastre.
- El loop solo corre cuando la sección está a la vista. WebGL es caro y esto está al final.

Las tres respetan `prefers-reduced-motion`: sin portada, sin rotación, un solo frame estático.

## Contenido: qué es real y qué no

**Casi todo el contenido del evento es inventado y hay que reemplazarlo.**

| Qué | Dónde | Estado |
|---|---|---|
| Link del RSVP | `id="rsvp-link"` y el botón del nav | apunta a `#` |
| Fecha y hora | 4 sitios en el HTML + `TARGET` en el script del contador | viernes 23.10.2026, 18:00–21:00 |
| Dirección | sección de sede | sin confirmar |
| Cupos / correo | hero, cierre, footer | 120 · hola@finapartner.com |

Si cambias la fecha, cambia también `TARGET` en el script del contador — está en ISO con
offset `-04:00` (Venezuela). Y verifica el día de la semana: ya hubo un error de "jueves"
en una fecha que caía viernes.

El vocabulario contable es **venezolano**, no chileno: *estado de cuenta*, *factura
electrónica*. Hubo que corregirlo una vez porque la primera versión asumía Chile.

## Referencia visual

La estructura viene de https://hack.platan.us/26-co — corchetes `[01]`, eyebrows numerados,
marquesina infinita, cifras gigantes, hairlines en vez de cards, changelog con `+`.
La paleta y la tipografía son de Fina, no de ahí. Esa mezcla es deliberada.

## Trampa: los textos con degradado

Seis elementos pintan texto con degradado (`.grad`, `.stat .v`, `h1 .n`, `.eyebrow .on`,
`.log-plus`, `.viz-tag b`) mediante `background-clip: text` + `color: transparent`.

**Si vas a cambiarles el degradado, usa `background-image`, nunca el atajo `background`.**
El atajo reinicia `background-clip` a `border-box`: el degradado pasa a llenar la caja
entera y el texto se vuelve invisible. Se ve como rectángulos de color donde debería
haber palabras. Ya pasó una vez.

## El sistema de vidrio

Las superficies que flotan comparten una sola clase, `.glass`, definida con tokens
(`--glass-fill`, `--glass-blur`, `--glass-edge`, `--glass-lift`). Cuatro ingredientes la
hacen leer como cristal y no como una caja translúcida:

1. `backdrop-filter: blur(20px) saturate(180%)` — la saturación importa tanto como el
   desenfoque: el color de detrás se intensifica al pasar por el vidrio.
2. Un filo blanco de 1px arriba, que es donde pega la luz.
3. Canto con dispersión: violeta arriba-izquierda, rosa abajo-derecha. Es el mismo
   recurso que el isotipo, y es lo que ata las dos cosas.
4. Sombra baja y ancha, para despegarlo del papel.

**Lo llevan solo cinco elementos**: la barra, el contador, el visor del edificio, la ficha
de la sede y las tarjetas del line-up. Botones, chips, marquesina y filas del changelog no.
Si todo es vidrio, nada lo es — y el CTA principal tiene que seguir siendo sólido.

Hay dos salidas de emergencia: `@supports not (backdrop-filter)` sube la opacidad del
relleno, y `prefers-reduced-transparency` devuelve superficie blanca sólida.
