# Isotipo de apertura

Portada a pantalla completa: el isotipo de Fina, resuelto como vidrio, ocupa la ventana y tapa lo que viene debajo. Al scrollear, el fondo se disuelve y **las dos barras se abren hacia fuera** en diagonal, dejando pasar a la página.

En vivo: <https://fina-releases-fina-b5773c0d.vercel.app>

---

## Cómo se comporta

El avance no es una duración, es **scrub**: va ligado al scroll y retrocede si subes. Tres cosas se mueven a ritmos distintos, y ese desfase es lo que hace que funcione.

| Recorrido | Qué pasa | Por qué |
|---|---|---|
| 0 → 28% | El aviso "scrollea para descubrir" se apaga | Ya cumplió su función |
| 0 → 80% | Las barras se separan, escalan y se desvanecen | El gesto principal |
| 22% → 72% | El fondo opaco se disuelve | **Espera a propósito** |

Ese retraso del fondo es la decisión central. Si se disolviera desde el primer píxel verías la página antes de que el isotipo se abriera, y se perdería la sensación de que algo se aparta para dejarte pasar. Primero se abren las barras sobre blanco sólido, y solo entonces aparece lo que hay detrás.

---

## Las seis capas de vidrio

El isotipo no es glow: es **vidrio**. La diferencia importa porque el glow es luz *emitida* (halos hacia fuera, sobre fondo oscuro) y el vidrio es luz *transmitida* — casi todo pasa a través y lo que se ve son los cantos. Sobre blanco solo funciona el segundo.

Cada mitad apila, en orden:

1. **Sombra proyectada** — 26 px abajo, muy difusa, al 16%. Es lo que la despega del papel.
2. **Cuerpo translúcido** — relleno teñido al 13–22%. Aquí iba el núcleo oscuro de la versión con glow; ahora el fondo se ve a través.
3. **Dispersión cromática** — dos copias del contorno desplazadas en direcciones opuestas y desenfocadas, una rosa y otra violeta. Es lo que lo hace leer como cristal y no como plástico de color: el vidrio real separa las longitudes de onda en los bordes.
4. **Canto** — 12 px con el degradado de marca, que se desplaza lentamente por SMIL.
5. **Brillo interior** — el mismo trazo erosionado 16 px y desenfocado, para que el blanco quede *dentro* del borde y no encima.
6. **Especular** — filo blanco desplazado arriba-izquierda, como si la luz entrara desde ahí. Respira en ciclos de 6 s.

Encima, un destello que recorre la pieza cada 9 s, desfasado medio ciclo entre las dos barras.

---

## Invariantes

Cosas que parecen mejorables y no lo son. Cada una costó descubrirla.

**Cada mitad es un `<svg>` propio, no un `<g>` dentro de uno compartido.**
Transformar un grupo SVG **no se compone en GPU**: el navegador rerasteriza el subárbol entero, con sus desenfoques, en cada frame. Separarlas permite `translate3d` + `will-change` y que cada una viaje en su propia capa. Por eso los `defs` están duplicados con ids sufijados (`-a` / `-b`).

**El scroll no escribe variables CSS heredadas.**
Una custom property en el contenedor invalida el estilo de **todo el subárbol**. El JS escribe `transform` y `opacity` directo sobre tres nodos concretos.

**`color-interpolation-filters="sRGB"` en los diez filtros.**
Por defecto SVG hace los desenfoques en espacio lineal, lo que obliga a convertir ida y vuelta en cada pasada. Cuesta aproximadamente el doble.

**El scroll mueve un objetivo; un bucle aparte lo persigue.**
Con un lerp normalizado por tiempo — `1 - (1-0.16)^(dt/16.7)` — el recorrido es idéntico a 60 y a 120 Hz. Pintar directo desde el evento de scroll da saltos discretos en cada tic de rueda.

**Durante el arrastre se congelan el latido y el destello.**
Son los que obligan a recalcular los desenfoques frame a frame. Parados, el presupuesto se va entero al movimiento. Sobre un ciclo de 6 s no se percibe.

**Pasada la apertura se oculta con `visibility`.**
Son cinco filtros de desenfoque, uno con `stdDeviation:40`. Dejarlos pintando invisibles durante toda la página cuesta caro.

**`pointer-events:none` en la portada.**
El scroll nunca se secuestra: no hay `preventDefault` ni saltos forzados. La barra se mueve como siempre.

---

## Integración

Tres bloques: el CSS, el markup y el script. No hay dependencias — ni librerías, ni fuentes, ni peticiones de red.

Necesita dos cosas del entorno:

- Un elemento espaciador que dé el recorrido de scroll. El de abajo es `.gate-spacer` a `135vh`.
- Estos tokens, o sustituirlos por literales:

```
--fina-pink        #FD37BE
--fina-purple      #620AFF
--fg-dim           #4C4270
--pixel            tipografía del aviso (aquí DotGothic16)
```

Los hexadecimales del propio isotipo salen muestreados del archivo de marca: la barra superior es violeta `#7C1FEA → #A658F1` y la inferior naranjo → rosa `#FFAB7B → #FF42BB`. **El rosa pertenece solo a la barra de abajo**; llevar el violeta hasta el rosa es un error fácil de cometer.

### Accesibilidad

El SVG de la primera mitad lleva `role="img"` y etiqueta; el de la segunda va `aria-hidden`. Con `prefers-reduced-motion` la portada deja de ser fija, se convierte en un bloque normal de 300 px, desaparecen el aviso y el espaciador, y la página abre directo. También se para el SMIL a mano con `pauseAnimations()`, porque **no obedece al media query**.

### Si el fondo deja de ser blanco

El cuerpo translúcido está calibrado contra blanco. Sobre un fondo oscuro el vidrio se ve casi negro: hay que subir las opacidades del relleno (capa 2) y revisar el especular.

---

## CSS

```css
/* ============================================================
   PORTADA — el isotipo ocupa la pantalla completa y tapa lo que
   viene debajo. Al scrollear, el fondo se disuelve y las dos
   barras se abren hacia fuera.

   Rendimiento: cada mitad es un SVG propio con sus defs, para
   que el navegador la promocione a capa de composición y la
   mueva en la GPU. El scroll no toca variables CSS heredadas
   (invalidan todo el subárbol): se escriben transform y opacity
   directo sobre tres nodos. El valor se persigue con un lerp
   por tiempo, así que la rueda no se siente a saltos.
   ============================================================ */
@keyframes iso-breathe      { 0%,100%{ opacity:.55 } 50%{ opacity:1 } }
@keyframes iso-breathe-soft { 0%,100%{ opacity:.82 } 50%{ opacity:1 } }
@keyframes iso-travel       { from{ stroke-dashoffset:0 } to{ stroke-dashoffset:-100 } }
@keyframes gate-in          { from{ transform:scale(.94) } to{ transform:scale(1) } }
@keyframes gate-bob         { 0%,100%{ transform:translateY(0) } 50%{ transform:translateY(5px) } }

.gate{
  position:fixed; inset:0; z-index:60;
  display:grid; place-items:center;
  overflow:hidden;
  pointer-events:none;          /* el scroll pasa de largo hacia la página */
}
.gate-bg{
  position:absolute; inset:0;
  will-change:opacity;
  background:#FFFFFF;
}

.iso-stack{
  position:relative;
  width:min(84vh, 88vw); aspect-ratio:1;
  animation:gate-in 1.1s cubic-bezier(.22,.61,.36,1) both;
}
.iso-half{
  position:absolute; inset:0;
  width:100%; height:100%;
  display:block; overflow:visible;
  transform:translate3d(0,0,0);
  will-change:transform, opacity;
  backface-visibility:hidden;
}

.iso-breathe     { animation:iso-breathe 6s cubic-bezier(.22,.61,.36,1) infinite; }
.iso-breathe-soft{ animation:iso-breathe-soft 6s cubic-bezier(.22,.61,.36,1) infinite; }
.iso-travel-a    { animation:iso-travel 9s linear infinite; }
.iso-travel-b    { animation:iso-travel 9s linear infinite; animation-delay:-4.5s; }

/* mientras el dedo arrastra, el latido y el destello se congelan:
   son los que obligan a recalcular los desenfoques en cada frame */
.gate.is-scrubbing .iso-breathe,
.gate.is-scrubbing .iso-breathe-soft,
.gate.is-scrubbing .iso-travel-a,
.gate.is-scrubbing .iso-travel-b{ animation-play-state:paused; }

.gate-hint{
  position:absolute; left:50%; bottom:clamp(26px,6vh,54px);
  transform:translateX(-50%);
  margin:0; will-change:opacity;
  display:flex; flex-direction:column; align-items:center; gap:10px;
  font-family:var(--pixel); font-size:12px; font-weight:400;
  letter-spacing:.18em; text-transform:uppercase; color:var(--fg-dim);
  white-space:nowrap;
}
.gate-hint .chev{
  width:22px; height:13px; fill:none;
  stroke:var(--fina-pink); stroke-width:2.4;
  stroke-linecap:round; stroke-linejoin:round;
  animation:gate-bob 1.9s ease-in-out infinite;
}

.gate-spacer{ height:135vh; }
.hero{ padding-top:clamp(26px,5vw,54px); }

@media (prefers-reduced-motion:reduce){
  .gate{ position:relative; z-index:1; height:auto; padding:clamp(30px,6vw,60px) 0; }
  .gate-bg, .gate-hint{ display:none; }
  .iso-stack{ width:min(300px,56vw); animation:none; }
  .gate-spacer{ display:none; }
}
```

## Markup

```html
<div class="gate" id="gate">
  <div class="gate-bg" id="gate-bg"></div>

  <div class="iso-stack">
    <svg class="iso-half" id="iso-a" viewBox="480 132 1400 1400" role="img" aria-label="Isotipo de Fina">
      <defs>
        <linearGradient id="gBody-a" gradientUnits="userSpaceOnUse" x1="1330" y1="330" x2="860" y2="960">
          <stop offset="0" stop-color="#7C1FEA" stop-opacity=".17"/>
          <stop offset="0.55" stop-color="#9440EE" stop-opacity=".14"/>
          <stop offset="1" stop-color="#A658F1" stop-opacity=".20"/>
        </linearGradient>
        <linearGradient id="gEdge-a" gradientUnits="userSpaceOnUse" x1="1330" y1="330" x2="860" y2="960">
          <stop offset="0" stop-color="#7C1FEA"/><stop offset="0.5" stop-color="#9440EE"/><stop offset="1" stop-color="#A658F1"/>
          <animateTransform attributeName="gradientTransform" type="translate" values="-95,113; 95,-113; -95,113" dur="9s" calcMode="spline" keySplines=".45 0 .55 1;.45 0 .55 1" repeatCount="indefinite"/>
        </linearGradient>
        <linearGradient id="gGloss-a" gradientUnits="userSpaceOnUse" x1="1330" y1="330" x2="860" y2="960">
          <stop offset="0" stop-color="#FFFFFF" stop-opacity=".95"/>
          <stop offset="0.45" stop-color="#FFFFFF" stop-opacity=".18"/>
          <stop offset="1" stop-color="#FFFFFF" stop-opacity="0"/>
        </linearGradient>
        <filter id="fDrop-a" x="-30%" y="-30%" width="160%" height="160%" color-interpolation-filters="sRGB">
          <feOffset dy="26"/><feGaussianBlur stdDeviation="26"/>
        </filter>
        <filter id="fFringe-a" x="-20%" y="-20%" width="140%" height="140%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="9"/></filter>
        <filter id="fInset-a" x="-15%" y="-15%" width="130%" height="130%" color-interpolation-filters="sRGB">
          <feMorphology operator="erode" radius="16"/><feGaussianBlur stdDeviation="20"/>
        </filter>
        <filter id="fSoft-a" x="-18%" y="-18%" width="136%" height="136%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="5"/></filter>
        <filter id="fGleam-a" x="-25%" y="-25%" width="150%" height="150%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="11"/></filter>
      </defs>

      <g filter="url(#fDrop-a)" opacity=".16"><path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="#3B1C6B"/></g>

      <path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="url(#gBody-a)"/>

      <g filter="url(#fFringe-a)" opacity=".5">
        <path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="none" stroke="#A658F1" stroke-width="9" stroke-linejoin="round" transform="translate(9,11)"/>
        <path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="none" stroke="#7C1FEA" stroke-width="9" stroke-linejoin="round" transform="translate(-9,-11)"/>
      </g>

      <g filter="url(#fInset-a)" opacity=".8"><path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="url(#gGloss-a)"/></g>

      <path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="none" stroke="url(#gEdge-a)" stroke-width="12" stroke-linejoin="round" opacity=".92"/>

      <g class="iso-breathe-soft" filter="url(#fSoft-a)" opacity=".85">
        <path d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" fill="none" stroke="#FFFFFF" stroke-width="9" stroke-linejoin="round" transform="translate(-7,-9)"/>
      </g>

      <g filter="url(#fGleam-a)" opacity=".85">
        <path class="iso-travel-a" d="M1123.2,355 L801.1,738.6 A150,150 0 0 0 1030.9,931.4 L1514.6,355 Z" pathLength="100" fill="none" stroke="#FFFFFF" stroke-width="26" stroke-linecap="round" stroke-dasharray="11 89"/>
      </g>
    </svg>

    <svg class="iso-half" id="iso-b" viewBox="480 132 1400 1400" aria-hidden="true">
      <defs>
        <linearGradient id="gBody-b" gradientUnits="userSpaceOnUse" x1="880" y1="1320" x2="1520" y2="740">
          <stop offset="0" stop-color="#FF42BB" stop-opacity=".21"/>
          <stop offset="0.55" stop-color="#FF779A" stop-opacity=".17"/>
          <stop offset="1" stop-color="#FFAB7B" stop-opacity=".22"/>
        </linearGradient>
        <linearGradient id="gEdge-b" gradientUnits="userSpaceOnUse" x1="880" y1="1320" x2="1520" y2="740">
          <stop offset="0" stop-color="#FF42BB"/><stop offset="0.5" stop-color="#FF779A"/><stop offset="1" stop-color="#FFAB7B"/>
          <animateTransform attributeName="gradientTransform" type="translate" values="95,-113; -95,113; 95,-113" dur="9s" calcMode="spline" keySplines=".45 0 .55 1;.45 0 .55 1" repeatCount="indefinite"/>
        </linearGradient>
        <linearGradient id="gGloss-b" gradientUnits="userSpaceOnUse" x1="880" y1="1320" x2="1520" y2="740">
          <stop offset="0" stop-color="#FFFFFF" stop-opacity=".95"/>
          <stop offset="0.45" stop-color="#FFFFFF" stop-opacity=".18"/>
          <stop offset="1" stop-color="#FFFFFF" stop-opacity="0"/>
        </linearGradient>
        <filter id="fDrop-b" x="-30%" y="-30%" width="160%" height="160%" color-interpolation-filters="sRGB">
          <feOffset dy="26"/><feGaussianBlur stdDeviation="26"/>
        </filter>
        <filter id="fFringe-b" x="-20%" y="-20%" width="140%" height="140%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="9"/></filter>
        <filter id="fInset-b" x="-15%" y="-15%" width="130%" height="130%" color-interpolation-filters="sRGB">
          <feMorphology operator="erode" radius="16"/><feGaussianBlur stdDeviation="20"/>
        </filter>
        <filter id="fSoft-b" x="-18%" y="-18%" width="136%" height="136%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="5"/></filter>
        <filter id="fGleam-b" x="-25%" y="-25%" width="150%" height="150%" color-interpolation-filters="sRGB"><feGaussianBlur stdDeviation="11"/></filter>
      </defs>

      <g filter="url(#fDrop-b)" opacity=".16"><path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="#3B1C6B"/></g>

      <path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="url(#gBody-b)"/>

      <g filter="url(#fFringe-b)" opacity=".5">
        <path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="none" stroke="#FFAB7B" stroke-width="9" stroke-linejoin="round" transform="translate(9,11)"/>
        <path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="none" stroke="#FF42BB" stroke-width="9" stroke-linejoin="round" transform="translate(-9,-11)"/>
      </g>

      <g filter="url(#fInset-b)" opacity=".8"><path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="url(#gGloss-b)"/></g>

      <path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="none" stroke="url(#gEdge-b)" stroke-width="12" stroke-linejoin="round" opacity=".92"/>

      <g class="iso-breathe-soft" filter="url(#fSoft-b)" opacity=".85">
        <path d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" fill="none" stroke="#FFFFFF" stroke-width="9" stroke-linejoin="round" transform="translate(-7,-9)"/>
      </g>

      <g filter="url(#fGleam-b)" opacity=".85">
        <path class="iso-travel-b" d="M1236.8,1309 L1558.9,925.4 A150,150 0 0 0 1329.1,732.6 L845.4,1309 Z" pathLength="100" fill="none" stroke="#FFFFFF" stroke-width="26" stroke-linecap="round" stroke-dasharray="11 89"/>
      </g>
    </svg>
  </div>

  <p class="gate-hint" id="gate-hint">
    Scrollea para descubrir
    <svg class="chev" viewBox="0 0 24 14" aria-hidden="true"><path d="M2 2 L12 12 L22 2"/></svg>
  </p>
</div>
<div class="gate-spacer" id="gate-spacer"></div>
```

## Script

```js
/* ============================================================
   Scrub de la portada.

   El scroll no pinta: solo mueve un objetivo. Un bucle aparte
   persigue ese objetivo con un lerp normalizado por tiempo, de
   modo que el resultado es igual a 60 que a 120 Hz y los saltos
   gruesos de la rueda salen suavizados.
   ============================================================ */
(function(){
  var gate   = document.getElementById("gate");
  var spacer = document.getElementById("gate-spacer");
  var bg     = document.getElementById("gate-bg");
  var hint   = document.getElementById("gate-hint");
  var halves = [document.getElementById("iso-a"), document.getElementById("iso-b")];
  if(!gate || !spacer || !bg || !hint || !halves[0] || !halves[1]) return;

  if(window.matchMedia("(prefers-reduced-motion: reduce)").matches){
    halves.forEach(function(s){ if(s.pauseAnimations) s.pauseAnimations(); });
    return;
  }

  var target = 0, cur = 0, raf = 0, prev = 0, scrubbing = false;
  function clamp(v){ return v < 0 ? 0 : v > 1 ? 1 : v; }

  function readTarget(){
    var span = spacer.offsetHeight || 1;
    target = clamp((window.scrollY || window.pageYOffset) / span);
    if(!raf){ prev = 0; raf = requestAnimationFrame(loop); }
  }

  function paint(p){
    var open = p * 72, up = p * -61, sc = 1 + p * .42;
    var op   = Math.max(0, 1 - p * 1.25).toFixed(3);

    halves[0].style.transform = "translate3d(" + open.toFixed(2) + "%," + up.toFixed(2) + "%,0) scale(" + sc.toFixed(3) + ")";
    halves[1].style.transform = "translate3d(" + (-open).toFixed(2) + "%," + (-up).toFixed(2) + "%,0) scale(" + sc.toFixed(3) + ")";
    halves[0].style.opacity = halves[1].style.opacity = op;

    bg.style.opacity   = (1 - clamp((p - .22) / .5)).toFixed(3);
    hint.style.opacity = (1 - clamp(p / .28)).toFixed(3);
    gate.style.visibility = p >= .92 ? "hidden" : "visible";
  }

  function loop(now){
    var dt = prev ? Math.min(now - prev, 64) : 16.7;
    prev = now;

    /* lerp independiente del refresco: mismo recorrido a 60 y a 120 Hz */
    cur += (target - cur) * (1 - Math.pow(1 - .16, dt / 16.7));

    var moving = Math.abs(target - cur) > .0008;
    if(!moving) cur = target;

    if(moving !== scrubbing){
      scrubbing = moving;
      gate.classList.toggle("is-scrubbing", moving);
    }

    paint(cur);

    if(moving){ raf = requestAnimationFrame(loop); }
    else { raf = 0; prev = 0; }
  }

  addEventListener("scroll", readTarget, {passive:true});
  addEventListener("resize", readTarget);
  readTarget();
  paint(cur);
})();
```
