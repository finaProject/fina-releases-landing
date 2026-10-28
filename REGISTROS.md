# Gestión de registros — Fina 360°

Cómo se filtran las respuestas del formulario de Luma, cómo se decide a quién se
aprueba y cómo llega el correo de confirmación.

| | |
|---|---|
| Evento | <https://luma.com/jxv3y34r> · calendario `cal-6hJ4kE1XV3TLrUD` |
| Hoja | [Registro Fina 360°](https://docs.google.com/spreadsheets/d/1tlixwrGRKzyZtuqekFil8CglP8WI7CZeG89290Q_hPo/edit) |
| Panel | `registros-panel.html` en esta rama, publicado como artefacto |
| Cupos | 120 · el registro es **con aprobación previa** |

---

## Lo que no existe

Tres supuestos que conviene no repetir, porque cambian todo el diseño:

1. **Luma no tiene plan "estándar".** Son dos: **Free** y **Plus** (59 $/mes en plan
   anual). No hay nada intermedio.
2. **Luma no se sincroniza con Google Sheets** en ningún plan. No hay integración
   nativa, ni en Free ni en Plus.
3. **Un artefacto no puede leer la hoja.** Corre en un marco aislado sin red: no
   alcanza ni Sheets ni Luma. Sí puede guardar datos propios, preguntarle a Claude y
   ofrecer descargas, que es sobre lo que está construido el panel.

Consecuencia práctica: **mientras el evento esté en Free, la hoja no aporta nada.**
Sería un sitio donde aparcar un CSV a mano. El panel toma ese CSV directamente. La
hoja queda reservada para el camino B, donde sí tiene un papel: alojar el Apps Script.

---

## Camino A — plan Free (el que está construido)

La API de Luma requiere Plus. En Free, la aprobación se hace desde la web de Luma,
que es además **quien manda el correo de confirmación**. El panel se encarga de la
parte difícil —leer, filtrar y decidir— y entrega la lista lista para pegar.

### El flujo, de principio a fin

1. **Exportar.** En Luma: `Manage → Guests → Download CSV`.
2. **Cargar.** En el panel, paso 1: elegir el archivo, arrastrarlo, o pegar el CSV
   como texto.
3. **Mapear.** Paso 2. El panel muestra las columnas que encontró y pide tres cosas:
   cuál es el correo, cuál es el nombre, y cuáles son respuestas del formulario. Lo
   adivina (busca `email`/`correo`, descarta `Created at` y compañía) pero la última
   palabra es tuya. **El mapeo se guarda**: la siguiente tanda no vuelve a preguntarlo.

   Es editable a propósito. Las preguntas del formulario de Luma no se pueden leer
   desde fuera del registro, así que el panel no puede traer los encabezados
   cableados. Si mañana agregas una pregunta, aparece sola como columna nueva.
4. **Criterios.** Paso 3: un campo de texto donde escribes a quién quieres dentro y a
   quién no, en español corriente. También se guarda.
5. **Evaluar (opcional).** "Evaluar pendientes con IA" lee cada registro contra esos
   criterios y devuelve *Aprobar · Rechazar · Revisar* con una frase de motivo. Va en
   tandas de 15 y se puede detener a mitad; lo ya evaluado queda guardado.

   **La IA sugiere, no decide.** Ningún veredicto cambia el estado de nadie: sigue
   haciendo falta que pulses aprobar o rechazar.
6. **Filtrar.** Paso 4. Por cada columna marcada como respuesta el panel arma el
   filtro que corresponda: casillas con el conteo de cada opción cuando la respuesta
   es cerrada, caja de búsqueda cuando es texto libre. Se combinan entre sí, con el
   filtro de estado y con la búsqueda general.
7. **Decidir.** Paso 5. Aprobar o rechazar fila por fila, o marcar varias y usar los
   botones del pie. El contador de cupos libres baja con cada aprobación.
8. **Exportar.** "Copiar correos aprobados" deja en el portapapeles un correo por
   línea. "Descargar CSV" baja lo que estés viendo —filtros incluidos— con las
   respuestas, el veredicto de la IA, su motivo y tu decisión.
9. **Aplicar en Luma.** `Manage → Guests → Bulk Update Status`, pegar los correos,
   elegir *Approved*. **Luma manda el correo de confirmación.**

### El paso que falta comprobar

La documentación de Luma no dice si **Bulk Update Status está disponible en Free**.
Es una comprobación de un minuto en `Manage → Guests`, y conviene hacerla antes de
que haya registros de verdad:

> Pega **un solo correo tuyo**, apruébalo, y confirma que te llega el correo de
> confirmación de Luma.

Si resultara que Bulk Update Status es de pago, en Free queda aprobar de a uno desde
la lista de invitados — el panel sigue sirviendo igual para decidir, solo cambia el
último paso.

### Dónde viven los datos

En la base de datos del artefacto, no en el navegador:

| Ruta | Qué guarda |
|---|---|
| `config/mapping` | El mapeo de columnas y el texto de los criterios |
| `registros/<correo>` | Una ficha por invitado: fila original, veredicto de la IA, motivo, decisión y cuándo se tomó |

Esto significa que **las decisiones se comparten** entre todo el que abra el panel, y
sobreviven a republicaciones. Reglas de acceso, las que trae por defecto:

- quien puede **ver** el artefacto, lee las decisiones pero no las cambia (el panel
  le muestra los botones desactivados y el letrero "solo lectura");
- quien puede **usarlo** (Contributor) o editarlo, decide.

Volver a importar un CSV **no borra decisiones**: las filas que ya existen conservan
su estado y solo se refrescan sus respuestas. El correo es la llave, en minúsculas.

Abierto fuera de un artefacto —el archivo servido por HTTP desde esta rama— el panel
funciona igual pero sin base de datos ni IA: lo dice en la cabecera y lo que decidas
se pierde al recargar. Sirve para probar, no para trabajar.

---

## Camino B — con Luma Plus

Con Plus aparece la API, y con ella lo que pediste literalmente: **aprobar desde la
hoja y que Luma mande el correo solo**.

### Qué hace falta

1. Contratar **Luma Plus** en el calendario `cal-6hJ4kE1XV3TLrUD`.
2. Generar una API key en `Settings → Developer`.
3. Pegar el Apps Script de abajo en la hoja (`Extensiones → Apps Script`).

### Los dos endpoints

**Traer los registros con sus respuestas**

```http
GET https://public-api.luma.com/v1/events/guests/list?event_id=evt-XXXX
x-luma-api-key: <API_KEY>
```

Devuelve cada invitado con `registration_answers`: las respuestas del formulario, que
es justo lo que el CSV trae como columnas. Esto es lo que permite volcar los registros
en la hoja cada pocos minutos — la sincronización automática que la idea original daba
por hecha, y que solo existe aquí.

**Aprobar o rechazar**

```http
POST https://public-api.luma.com/v1/events/guests/update-status
x-luma-api-key: <API_KEY>
Content-Type: application/json

{
  "event_id": "evt-XXXX",
  "guest_id": "persona@correo.com",
  "status": "approved",
  "send_email": true,
  "message": "¡Nos vemos el 24 de noviembre!"
}
```

- `guest_id` acepta el correo del invitado, que es lo que tenemos a mano.
- `send_email` viene en `true` por defecto: **aprobar por API dispara el correo de
  Luma**. Ese es el punto entero del camino B.
- `status`: `approved` para aceptar, `declined` para rechazar.
- El `event_id` (`evt-…`, distinto del `jxv3y34r` de la URL pública) sale de
  `GET /v1/calendar/list-events`.

> Estos datos están tomados de la documentación de Luma, no de una llamada real:
> el evento está en Free y la API devuelve 403. Confirma los nombres de los campos
> contra la documentación el día que actives Plus, antes de correr el script sobre
> registros de verdad.

### ⚠️ Antes de pegar la API key en ningún sitio

**La API key da acceso completo al calendario**: leer la lista de invitados con sus
correos y respuestas, y cambiar el estado de cualquiera. No es una llave de solo
lectura.

- **No la escribas dentro del código del script.** Va en
  `Configuración del proyecto → Propiedades del script`, con el nombre `LUMA_API_KEY`.
  El script de abajo la lee de ahí.
- Aun así: **cualquiera con permiso de editor sobre la hoja puede abrir el Apps Script
  y leer esa propiedad.** Revisa quién tiene acceso a la hoja antes de guardar la key,
  y trátala como una contraseña del calendario.
- Si se filtra, revócala en `Settings → Developer` y genera otra.
- No la pongas en este repositorio, ni en `index.html`, ni en el panel: el panel corre
  en el navegador de quien lo abra, donde cualquier clave es pública.

### El script

Pegar en `Extensiones → Apps Script` de la hoja. Crea un menú propio al abrirla.

```javascript
const API = 'https://public-api.luma.com/v1';
const EVENT_ID = 'evt-XXXX';          // el de list-events, no el de la URL
const COL_CORREO = 1;                 // columna A
const COL_DECISION = 2;               // columna B: "aprobar" / "rechazar"
const COL_RESULTADO = 3;              // columna C: la escribe el script

function onOpen() {
  SpreadsheetApp.getUi()
    .createMenu('Fina 360°')
    .addItem('Traer registros de Luma', 'sincronizar')
    .addItem('Aplicar decisiones', 'aplicarDecisiones')
    .addToUi();
}

function key_() {
  const k = PropertiesService.getScriptProperties().getProperty('LUMA_API_KEY');
  if (!k) throw new Error('Falta LUMA_API_KEY en las propiedades del script.');
  return k;
}

function llamar_(ruta, opciones) {
  const res = UrlFetchApp.fetch(API + ruta, Object.assign({
    headers: { 'x-luma-api-key': key_() },
    muteHttpExceptions: true
  }, opciones || {}));
  const code = res.getResponseCode();
  if (code === 403) throw new Error('403: la API de Luma requiere plan Plus.');
  if (code >= 300) throw new Error(code + ': ' + res.getContentText());
  return JSON.parse(res.getContentText());
}

/** Vuelca los invitados y sus respuestas del formulario en la pestaña activa. */
function sincronizar() {
  let cursor = null, invitados = [];
  do {
    const q = '?event_id=' + encodeURIComponent(EVENT_ID) +
              (cursor ? '&pagination_cursor=' + encodeURIComponent(cursor) : '');
    const page = llamar_('/events/guests/list' + q);
    invitados = invitados.concat(page.entries || []);
    cursor = page.has_more ? page.next_cursor : null;
  } while (cursor);

  // Las preguntas del formulario no son fijas: se arma el encabezado con lo que venga.
  const preguntas = [];
  invitados.forEach(e => (((e.guest || e).registration_answers) || []).forEach(r => {
    const q = r.label || r.question_id;
    if (q && preguntas.indexOf(q) < 0) preguntas.push(q);
  }));

  const filas = [['Correo', 'Decisión', 'Resultado', 'Nombre', 'Estado en Luma'].concat(preguntas)];
  invitados.forEach(e => {
    const g = e.guest || e;
    const resp = {};
    (g.registration_answers || []).forEach(r => { resp[r.label || r.question_id] = r.answer; });
    filas.push([g.user_email || g.email, '', '', g.user_name || g.name, g.approval_status]
      .concat(preguntas.map(q => resp[q] || '')));
  });

  const hoja = SpreadsheetApp.getActiveSheet();
  hoja.clear();
  hoja.getRange(1, 1, filas.length, filas[0].length).setValues(filas);
  hoja.setFrozenRows(1);
  SpreadsheetApp.getUi().alert(invitados.length + ' registros traídos de Luma.');
}

/** Lee la columna de decisión y la aplica en Luma. Luma manda el correo. */
function aplicarDecisiones() {
  const hoja = SpreadsheetApp.getActiveSheet();
  const datos = hoja.getDataRange().getValues();
  let hechas = 0, fallos = 0;

  for (let i = 1; i < datos.length; i++) {
    const correo = String(datos[i][COL_CORREO - 1] || '').trim();
    const orden = String(datos[i][COL_DECISION - 1] || '').trim().toLowerCase();
    const yaHecho = String(datos[i][COL_RESULTADO - 1] || '').trim();
    if (!correo || !orden || yaHecho) continue;

    const status = orden === 'aprobar' ? 'approved'
                 : orden === 'rechazar' ? 'declined' : null;
    if (!status) continue;

    try {
      llamar_('/events/guests/update-status', {
        method: 'post',
        contentType: 'application/json',
        payload: JSON.stringify({
          event_id: EVENT_ID,
          guest_id: correo,
          status: status,
          send_email: true   // por defecto ya es true; explícito para que se vea
        })
      });
      hoja.getRange(i + 1, COL_RESULTADO).setValue(status + ' ' + new Date().toISOString());
      hechas++;
    } catch (err) {
      hoja.getRange(i + 1, COL_RESULTADO).setValue('ERROR: ' + err.message);
      fallos++;
    }
    Utilities.sleep(400);   // sin esto se topa con el límite de la API
  }
  SpreadsheetApp.getUi().alert(hechas + ' aplicadas, ' + fallos + ' con error.');
}
```

La columna *Resultado* es a propósito: una fila ya aplicada se salta en la siguiente
corrida, así que volver a pulsar el menú no reenvía correos. Para rehacer una, borra
su celda de resultado.

Para que la hoja se refresque sola: `Activadores → Añadir activador → sincronizar →
Basado en tiempo`. **No pongas `aplicarDecisiones` en un activador de tiempo** —
mandar correos debe ser siempre un clic deliberado.

---

## Qué elegir

| | Free (camino A) | Plus (camino B) |
|---|---|---|
| Leer las respuestas | CSV descargado a mano | Automático, cada X minutos |
| Filtrar y decidir | El panel | El panel o la hoja |
| Aplicar en Luma | Pegar correos en Bulk Update Status | Un clic en el menú de la hoja |
| Correo de confirmación | Lo manda Luma | Lo manda Luma |
| Costo | 0 | 59 $/mes (plan anual) |
| Riesgo | Ninguno | Una API key con acceso total al calendario |

Para un evento de 120 cupos y una tanda de aprobaciones, el camino A cuesta treinta
segundos de pegar correos. El camino B tiene sentido si esto se vuelve recurrente.
