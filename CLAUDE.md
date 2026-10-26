# Fina Releases — landing del evento

Página única para el evento de lanzamiento de producto de Fina. Objetivo: registro (RSVP).
Español, público de contadores, founders y equipos de finanzas.

- **Producción:** https://fina-releases-fina-b5773c0d.vercel.app
- **Casi todo vive en `index.html`.** Sin build, sin framework, sin dependencias instaladas.
  CSS y JS inline. Fuera del HTML solo hay dos archivos: `logo.png` y `rocket.glb`.
  De red: Google Fonts, Three.js r128 desde cdnjs y su `GLTFLoader` desde jsDelivr.

⚠️ **Para verla en local hay que servirla por HTTP**, no abrir el archivo. El cohete se
pide con XHR a `rocket.glb` y el navegador bloquea esa petición en `file://`: la sección
de registro se queda vacía. Cualquier servidor estático vale (`python -m http.server 8000`).
Si falta el cohete, la consola lo dice.

## Desplegar

```bash
vercel deploy --prod --yes --scope fina-b5773c0d
```

El proyecto es `fina-releases` en el equipo `fina-b5773c0d` (ids en `.vercel/project.json`).
El repo está en https://github.com/finaProject/fina-releases-landing, pero **no hay CI**:
commitear y subir no despliega nada. El deploy se corre a mano desde esta carpeta, así que
después de cada push hay que acordarse de ejecutarlo o producción se queda atrás.

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
texto                #150041   (--fg, la tinta del wordmark)
```

El sistema completo, con los componentes renderizados y el bloque de tokens listo para
copiar, está documentado aparte como artefacto reutilizable.

**La página es clara.** Fue oscura hasta septiembre de 2026 y se invirtió a petición de
Diego. El histórico importa porque quedan decisiones heredadas de aquello.

⚠️ **El gradiente de marca no sirve como texto sobre blanco.** El extremo naranjo #FFB470
no llega a 2:1 de contraste. Los titulares usan `--grad-text` (violeta → rosa,
#620AFF → #FD37BE); el rosa → naranjo se queda donde es relleno: el botón primario y el
tinte bajo el vidrio del cintillo. Lo mismo con los chips: como texto van en
`--ink-warm` #B8651A y `--ink-cyan` #00898D.

⚠️ **Nada de mezcla aditiva ni `mix-blend-mode: screen`.** Sobre blanco, screen con
cualquier cosa da blanco, y el aditivo es invisible. Toda la página está en mezcla normal.
Si reaparece un `AdditiveBlending` o un `screen`, ese elemento simplemente no se ve.

Ojo: existe un segundo sistema, `fina-rrss` (el de Claude Design), con los mismos colores
desplazados un dígito — `#6209FF`, `#FD38BE`, `#FFB570`. La página está unificada hacia
`tokens.css`. Si alguien decide que manda el otro, hay que cambiarlos todos, no mezclar.

**Tipografía.** Figtree para todo el lenguaje humano. DotGothic16 (pixel) solo para la capa
máquina: eyebrows `// 01`, versiones `v2.6.0`, chips de estado, duraciones de la agenda,
labels del contador y footer. La regla es *lo que escribe el sistema va en pixel, lo que
escribe una persona va en Figtree*. Es lo que sostiene el aire techie; si se rompe, la
página se vuelve una landing genérica.

El hero va en Figtree **600**, no 900. A 162px el peso alto empasta las contraformas.
Compensación óptica: a más tamaño, menos peso. Los títulos de sección sí van en 800.

Única excepción conocida a la regla: las descripciones de la rueda de la agenda (`.aw-sub`)
van en pixel aunque son prosa humana. Viene así del handoff de diseño, marcado como final.

## Las piezas animadas

Son lo más frágil del archivo. Cada una tiene invariantes que cuestan caro de redescubrir.

La regla común a todas: **el scroll no pinta, mueve un objetivo.** El handler solo
actualiza un número y un bucle de `requestAnimationFrame` lo persigue con un lerp
normalizado por tiempo, `1-(1-k)^(dt/16.7)`. El exponente es lo que hace que corra igual
a 60 que a 120 Hz; sin él va al doble de velocidad en pantallas rápidas.

**1. Portada — isotipo a pantalla completa** (`#gate`)

Documentada en detalle en `ISOTIPO.md`, junto con `isotipo-componente.html`, que es la
versión reenviable del componente. Lo esencial:

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

- Su lerp usa k = 0.16 (el de la rueda de la agenda es 0.12).
- No usar variables CSS heredadas para el scrub: invalidan el estilo de todo el subárbol.
  Se escribe `transform` y `opacity` directo sobre tres nodos.
- `color-interpolation-filters="sRGB"` en los doce filtros. Sin eso los desenfoques trabajan
  en linearRGB y cuesta el doble.
- El fondo opaco espera al 22% del recorrido antes de disolverse. Si se disuelve desde el
  primer píxel se ve la página antes de que el isotipo se abra y se pierde el efecto.
- La portada lleva `pointer-events:none`. Nunca secuestrar el scroll.

**2. Hero — título en partículas** (`#title`)
El `<h1>` va en `.sr-only` y la frase se dibuja como polvo: se rasteriza el texto en un
canvas fuera de pantalla y se muestrea su canal alfa en una rejilla para sacar el destino
de cada partícula.

⚠️ **El paso de la rejilla tiene que ser entero.** Se calcula en `build()` y el `* DPR` va
**dentro** del `Math.round`, no fuera:

```js
var step = Math.max(3, Math.round(Math.sqrt(bw*bh)/300 * DPR));   // bien
var step = Math.max(3, Math.round(Math.sqrt(bw*bh)/300) * DPR);   // roto
```

Con la segunda forma y un `devicePixelRatio` fraccionario —Windows al 125%, que es el
escritorio típico— sale un paso decimal. El bucle indexa `data[(y*bw+x)*4+3]` con esa
coordenada y lee un byte desalineado: la R o la G de un píxel vecino en vez de la alfa.
Solo el 39% de las partículas cae sobre una letra, el resto se reparte por todo el bloque
y la frase no llega a formarse. En móvil el DPR es 2 o 3, el paso sale entero y no se ve.

**3. Hero — edificio en alambre** (`#bld`)
Geometría 3D procedural sobre canvas 2D, sin librerías. Cinco cajas: dos alas de 9 pisos, un
cuerpo posterior que deja el patio al frente, zócalo y atrio.

- El temblor de línea es **determinista** (hash del índice). Con `Math.random()` por frame
  titila.
- Las líneas sobresalen 2–7px de cada esquina. Eso es lo que da el aire de plano trazado a
  mano en vez de render limpio; no es un bug.
- Color por posición en pantalla, interpolando el gradiente de marca.
- El zócalo llevó rotulación ("CARACAS CAMPUS", letras como polilíneas 3D que giraban con
  el edificio). Se quitó en octubre de 2026 junto con el alfabeto de trazo y la rama de
  color azul marino que existía solo para ellas.

**4. Agenda — rueda con scroll** (`#agenda`)
Sección de 500vh (100vh por momento) con el escenario en `position:sticky`. El scroll gira
una rueda de cinco números; todo deriva de un único `pos` continuo.

- Dentro de cada tramo, el 20% inicial y final quedan **quietos** (`HOLD`) y el centro se
  interpola con smoothstep. Sin ese reposo la rueda nunca se detiene en un número.
- `STEP_ANGLE` 22°, `LERP` 0.12. Clic en un número hace `scrollTo` a su punto de reposo.
- Ningún ancestro puede llevar `overflow:hidden` o el sticky deja de funcionar. El `body`
  tiene `overflow-x:hidden`, que en Chrome convive bien con esto, pero es frágil: si el
  escenario deja de quedarse fijo, mirar ahí primero.

**5. Cierre — cohete 3D** (`#stage`)
Three.js r128 desde cdnjs más `GLTFLoader` desde jsDelivr. El cohete es `rocket.glb`
(18,5 MB, 470k triángulos), **en carga diferida**: no se pide hasta que la sección está a
dos pantallas. Si el CDN o el modelo fallan, la sección sigue: solo no hay cohete.

- El modelo se normaliza en tiempo de ejecución —se mide su bounding box y se escala a 6,6
  unidades— para que la cámara y las luces no tengan que tocarse.
- Despega solo al entrar en pantalla y se repite con clic. El umbral de 6px distingue clic de
  arrastre.
- La estela de chispas vive en el espacio de **la escena**, no en el del cohete. Eso es lo
  que hace que se quede atrás al subir en vez de viajar pegada.
- El loop solo corre cuando la sección está a la vista. WebGL es caro y esto está al final.

⚠️ El modelo es pesado para el público objetivo (pymes venezolanas, mucho móvil). No se ha
comprimido porque hace falta tooling que no está instalado: con Node, `gltf-transform` con
Draco y texturas a 1024px lo dejaría en 2–3 MB. El blob ya está en el historial de git.

Todas respetan `prefers-reduced-motion`: sin portada, sin rotación, sin estela, un solo
frame estático.

## Contenido: qué es real y qué no

**Casi todo el contenido del evento es inventado y hay que reemplazarlo.**

| Qué | Dónde | Estado |
|---|---|---|
| Link del RSVP | `id="rsvp-link"` y el botón del nav | **apunta a `#`** |
| Fecha | 5 sitios en el HTML + `TARGET` en el script del contador | martes 24.11.2026 |
| Duración del momento 02 | `DATA` en el script de la rueda | **vacía, llegó así del handoff** |
| Dirección | sección de sede | sin confirmar, solo "Caracas" |
| Cupos / correo | hero, cierre, footer | 120 · hola@finapartner.com |
| `build 2026.10` | eyebrow del hero | sigue diciendo octubre |

El evento **ya no se cuenta como seis releases** sino como un ecosistema: las cifras del
hero son `01 Ecosistema`, `60 Minutos` y `120 Cupos`. Queda un resto del marco viejo en el
párrafo de `// 00 Qué es esto`, que aún habla de "cada cosa nueva que entró a producción".

La página **no muestra horas**. Se quitaron a propósito en octubre de 2026. El `TARGET` del
contador conserva las 18:00 porque necesita un instante al que llegar, pero esa hora no
aparece en ninguna parte.

Si cambias la fecha, cambia también `TARGET` — está en ISO con offset `-04:00` (Venezuela,
no Chile). Y verifica el día de la semana: ya hubo un error de "jueves" en una fecha que
caía viernes.

El vocabulario contable es **venezolano**, no chileno: *estado de cuenta*, *factura
electrónica*. Hubo que corregirlo una vez porque la primera versión asumía Chile.

## Referencia visual

La estructura viene de https://hack.platan.us/26-co — corchetes `[01]`, eyebrows numerados,
cintillo infinito, cifras gigantes, hairlines en vez de cards.
La paleta y la tipografía son de Fina, no de ahí. Esa mezcla es deliberada.

El changelog con `+` y la baraja del line-up que había en esa referencia ya no existen:
se eliminaron con la sección de preguntas y con el cambio de la agenda a rueda.

La página son hoy cinco bloques: portada (`#gate`), hero (`#top`), qué es esto + cifras,
dónde (ficha de sede), agenda en rueda (`#agenda`) y registro (`#rsvp`).

## Trampa: los textos con degradado

Cinco elementos pintan texto con degradado (`.grad`, `.stat .v`, `.eyebrow .on`,
`.viz-tag b` y los números de la rueda `.aw-num span`) mediante `background-clip: text` +
`color: transparent`. Los de la rueda los escribe el script, que ya usa `backgroundImage`.

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

**Lo llevan**: la barra del nav, el contador, el visor del edificio, la ficha de la sede,
el botón fantasma y —desde octubre de 2026— el cintillo. Chips y filas de lista no. La
idea sigue siendo que si todo es vidrio nada lo es, y el CTA principal tiene que seguir
siendo sólido.

El cintillo fue una excepción pedida expresamente: antes era relleno rosa → naranjo. Sobre
blanco el `backdrop-filter` no tiene nada que refractar, así que bajo el relleno
translúcido va un tinte de marca al 34%; eso es lo que el desenfoque satura. Sin ese tinte
la banda se vuelve gris y desaparece.

Hay dos salidas de emergencia: `@supports not (backdrop-filter)` sube la opacidad del
relleno, y `prefers-reduced-transparency` devuelve superficie blanca sólida.
