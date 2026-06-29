# 🎰 Rifa App

Sistema completo de control de rifas de 150 números con dos interfaces: una para la **organizadora** (registro y sorteo) y otra para los **participantes** (elección de números).

**🔗 Links:**
- Panel organizadora: [rifa-app.pages.dev](https://rifa-app.pages.dev)
- Interfaz cliente: [rifa-app.pages.dev/cliente.html](https://rifa-app.pages.dev/cliente.html)

---

## ✨ Funcionalidades

### Panel organizadora (`index.html`)
- Tablero visual de 150 números con colores por estado
- Registro de participantes: nombre, teléfono, notas y fecha
- Edición de cualquier número en cualquier momento
- Exportación a Excel `.xlsx` con todos los participantes
- Sorteo con ruleta animada estilo casino entre números pagados
- Pantalla de revelación del ganador con animación y destellos
- Anulación del ganador desde 3 lugares distintos
- Sincronización en tiempo real con Google Sheets
- Indicador de estado de sincronización (cargando / guardando / error)

### Interfaz cliente (`cliente.html`)
- Grilla de 150 números con estado en tiempo real
- Colores claros: gris = disponible, rojo = ocupado, verde = seleccionado
- Selección/deselección con un toque
- Barra inferior con conteo, total a pagar y lista de números elegidos
- Envío directo al WhatsApp de la organizadora con mensaje pre-redactado

---

## 🗂️ Estructura del proyecto

```
rifa-app/
├── index.html      ← panel de control para la organizadora
├── cliente.html    ← interfaz pública para participantes
├── .gitignore
└── README.md
```

Ambos archivos son HTML autocontenidos — sin frameworks locales, sin build, sin node_modules. Todo corre directo en el navegador.

---

## 🏗️ Arquitectura

```
Navegador (cliente.html / index.html)
        │
        │  fetch GET
        ▼
Google Apps Script (Web App)
        │
        │  read / write
        ▼
Google Sheets (base de datos)
```

- **Frontend:** HTML + React 18 (CDN) + Babel Standalone (CDN) + SheetJS (CDN)
- **Backend:** Google Apps Script publicado como Web App
- **Base de datos:** Google Sheets
- **Hosting:** Cloudflare Pages (auto-deploy desde GitHub)

---

## 📊 Modelo de datos

Cada fila en Google Sheets representa un número de la rifa:

| Campo | Tipo | Descripción |
|---|---|---|
| `numero` | número | 1 al 150 |
| `estado` | string | `disponible` / `reservado` / `pagado` |
| `nombre` | string | Nombre del participante |
| `telefono` | string | Teléfono de contacto |
| `notas` | string | Notas adicionales |
| `fecha` | string | Fecha de registro (formato `dd/mm/yyyy`) |
| `esGanador` | boolean | `true` si fue seleccionado en el sorteo |

Solo se escriben en Sheets los números que tienen participante asignado. Los números disponibles no generan filas.

---

## 🔄 Flujo de estados

```
disponible → reservado → pagado → [SORTEO] → ganador
                ↓
           disponible  (si se cancela)
```

La organizadora cambia los estados manualmente desde el panel. Los clientes solo ven qué números están ocupados (reservados + pagados aparecen como "ocupado" en `cliente.html`).

---

## 🔗 API — Google Apps Script

El Apps Script expone un único endpoint GET con distintas acciones:

### Leer todos los datos
```
GET /exec
→ { ok: true, data: [...] }
```

### Guardar un número
```
GET /exec?action=saveOne&data=<JSON codificado>
→ { ok: true }
```
El JSON contiene todos los campos del número. Si la fila ya existe en Sheets, se actualiza. Si no, se agrega.

### Marcar / anular ganador
```
GET /exec?action=setGanador&numero=<N>&anular=<true|false>
→ { ok: true }
```
Marca `esGanador = true` en el número indicado y `false` en todos los demás. Si `anular=true`, pone `false` en todos.

---

## ⚙️ Configuración desde cero

### 1. Google Sheets

Crea una hoja de cálculo con estos encabezados exactos en la fila 1:

```
numero | estado | nombre | telefono | notas | fecha | esGanador
```

### 2. Apps Script

En la hoja ve a **Extensiones → Apps Script**, crea un proyecto nuevo y pega este código:

```javascript
const SHEET_NAME = "Hoja 1";

function doGet(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);

  if (e.parameter.action === "saveOne" && e.parameter.data) {
    const n = JSON.parse(decodeURIComponent(e.parameter.data));
    const rows = sheet.getDataRange().getValues();
    let filaExistente = -1;
    for (let i = 1; i < rows.length; i++) {
      if (Number(rows[i][0]) === Number(n.numero)) { filaExistente = i + 1; break; }
    }
    const rowData = [n.numero, n.estado, n.nombre, n.telefono, n.notas, n.fecha || "", n.esGanador];
    if (filaExistente > 0) {
      sheet.getRange(filaExistente, 1, 1, 7).setValues([rowData]);
    } else {
      if (rows.length === 0) sheet.appendRow(["numero","estado","nombre","telefono","notas","fecha","esGanador"]);
      sheet.appendRow(rowData);
    }
    return ContentService.createTextOutput(JSON.stringify({ ok: true })).setMimeType(ContentService.MimeType.JSON);
  }

  if (e.parameter.action === "setGanador") {
    const numero = Number(e.parameter.numero);
    const anular = e.parameter.anular === "true";
    const rows = sheet.getDataRange().getValues();
    for (let i = 1; i < rows.length; i++) {
      const esEste = Number(rows[i][0]) === numero;
      sheet.getRange(i + 1, 7).setValue(anular ? false : esEste);
    }
    return ContentService.createTextOutput(JSON.stringify({ ok: true })).setMimeType(ContentService.MimeType.JSON);
  }

  const rows = sheet.getDataRange().getValues();
  if (rows.length <= 1) return ContentService.createTextOutput(JSON.stringify({ ok: true, data: [] })).setMimeType(ContentService.MimeType.JSON);
  const headers = rows[0];
  const data = rows.slice(1).map(row => {
    const obj = {};
    headers.forEach((h, i) => obj[h] = row[i]);
    obj.esGanador = obj.esGanador === true || obj.esGanador === "TRUE";
    return obj;
  });
  return ContentService.createTextOutput(JSON.stringify({ ok: true, data })).setMimeType(ContentService.MimeType.JSON);
}
```

Publica como **Aplicación web**:
- Ejecutar como: **Yo**
- Quién tiene acceso: **Cualquier persona**

### 3. Actualizar URLs en el código

En ambos archivos HTML, actualiza la constante con tu URL de Apps Script:

```js
const APPS_SCRIPT_URL = "https://script.google.com/macros/s/TU_URL/exec";
```

Y en `cliente.html`, el número de WhatsApp de la organizadora:

```js
const WHATSAPP_NUM = "51XXXXXXXXX"; // código de país + número
```

---

## 🚀 Despliegue en Cloudflare Pages

1. Sube el repositorio a GitHub
2. Ve a [pages.cloudflare.com](https://pages.cloudflare.com) → **Create a project → Connect to Git**
3. Selecciona el repositorio `rifa-app`
4. Configuración de build:
   - Framework preset: **None**
   - Build command: *(vacío)*
   - Output directory: *(vacío)*
5. **Save and Deploy**

Cloudflare detecta cambios en GitHub y redespliega automáticamente en cada `git push`.

---

## 📱 Uso recomendado

| Quién | URL | Dispositivo |
|---|---|---|
| Organizadora | `rifa-app.pages.dev` | PC o celular (navegador) |
| Participantes | `rifa-app.pages.dev/cliente.html` | Celular (cualquier navegador) |

---

## 🛠️ Tecnologías

| Tecnología | Versión | Uso |
|---|---|---|
| React | 18.2.0 (CDN) | UI del panel organizadora |
| Babel Standalone | 7.23.2 (CDN) | Transpilación JSX en navegador |
| SheetJS / xlsx | 0.18.5 (CDN) | Exportación a Excel |
| Canvas API | nativa | Animación de la ruleta |
| Google Apps Script | — | Backend / API REST |
| Google Sheets | — | Base de datos |
| Cloudflare Pages | — | Hosting con CI/CD |
| GitHub | — | Control de versiones |

---

## 👤 Autor

**Jhoeliel Palma** — Arquitecto de Soluciones Digitales  
Chimbote, Perú
