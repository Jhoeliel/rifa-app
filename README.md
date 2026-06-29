# 🎰 Rifa App 

Control de rifas de 150 números con sorteo animado, sincronización en tiempo real vía Google Sheets y exportación a Excel.

**Demo:** [rifa-app.pages.dev](https://rifa-app.pages.dev) *(actualizar con URL real tras desplegar)*

---

## ✨ Funcionalidades

- **Tablero visual** — grilla de 150 números con color por estado (disponible / reservado / pagado / ganador)
- **Registro de participantes** — nombre, teléfono, notas y fecha por número
- **Edición** — cualquier número puede modificarse en cualquier momento
- **Sorteo con ruleta animada** — gira entre los números pagados y frena progresivamente en el ganador
- **Pantalla de ganador** — animación de revelación con destellos y trofeo
- **Anular ganador** — desde el header, desde el modal del número o desde la lista de participantes
- **Exportar a Excel** — descarga `.xlsx` con todos los participantes registrados
- **Sincronización con Google Sheets** — los datos se leen y escriben en una hoja de cálculo compartida, accesible desde cualquier dispositivo
- **Indicador de estado de sync** — muestra en tiempo real si está cargando, guardando o si hubo un error

---

## 🗂️ Estructura

```
rifa-app/
├── index.html    ← app completa (sin dependencias locales)
└── README.md
```

Un solo archivo HTML autocontenido. Sin frameworks instalados localmente, sin build steps, sin node_modules.

---

## ⚙️ Configuración

### 1. Google Sheets

Crea una hoja de cálculo con estos encabezados en la fila 1:

```
numero | estado | nombre | telefono | notas | fecha | esGanador
```

### 2. Apps Script

En la hoja ve a **Extensiones → Apps Script**, crea un nuevo proyecto con este código y publícalo como **Aplicación web** (ejecutar como: Yo / acceso: Cualquier persona):

```javascript
const SHEET_NAME = "Hoja 1";

function doGet(e) {
  const sheet = SpreadsheetApp.getActiveSpreadsheet().getSheetByName(SHEET_NAME);

  if (e.parameter.action === "saveAll" && e.parameter.data) {
    const numeros = JSON.parse(decodeURIComponent(e.parameter.data));
    sheet.clearContents();
    sheet.appendRow(["numero","estado","nombre","telefono","notas","fecha","esGanador"]);
    numeros.forEach(n => {
      sheet.appendRow([n.numero, n.estado, n.nombre, n.telefono, n.notas, n.fecha || "", n.esGanador]);
    });
    return ContentService
      .createTextOutput(JSON.stringify({ ok: true }))
      .setMimeType(ContentService.MimeType.JSON);
  }

  const rows = sheet.getDataRange().getValues();
  const headers = rows[0];
  const data = rows.slice(1).map(row => {
    const obj = {};
    headers.forEach((h, i) => obj[h] = row[i]);
    obj.esGanador = obj.esGanador === true || obj.esGanador === "TRUE";
    return obj;
  });
  return ContentService
    .createTextOutput(JSON.stringify({ ok: true, data }))
    .setMimeType(ContentService.MimeType.JSON);
}
```

### 3. Conectar la URL al proyecto

En `index.html`, reemplaza el valor de `APPS_SCRIPT_URL` con tu propia URL de Apps Script:

```js
const APPS_SCRIPT_URL = "https://script.google.com/macros/s/TU_URL_AQUI/exec";
```

---

## 🚀 Despliegue en Cloudflare Pages

1. Sube el repositorio a GitHub
2. Ve a [Cloudflare Pages](https://pages.cloudflare.com)
3. Conecta el repositorio `rifa-app`
4. Configuración de build:
   - **Framework preset:** None
   - **Build command:** *(vacío)*
   - **Output directory:** `/` *(o dejar en blanco)*
5. Haz clic en **Save and Deploy**

Cloudflare sirve el `index.html` directamente — no necesita compilación.

---

## 📊 Modelo de datos

Cada número de la rifa tiene esta estructura:

| Campo | Tipo | Descripción |
|---|---|---|
| `numero` | número | 1 al 150 |
| `estado` | string | `disponible` / `reservado` / `pagado` |
| `nombre` | string | Nombre del participante |
| `telefono` | string | Teléfono de contacto |
| `notas` | string | Notas adicionales |
| `fecha` | string | Fecha de registro (formato local PE) |
| `esGanador` | boolean | `true` si fue seleccionado en el sorteo |

---

## 🔄 Flujo de estados

```
disponible → reservado → pagado → [SORTEO] → ganador
                ↓
           disponible  (si se cancela)
```

---

## 🛠️ Tecnologías

| Tecnología | Uso |
|---|---|
| React 18 (CDN) | Interfaz de usuario |
| Babel Standalone (CDN) | Transpilación JSX en el navegador |
| SheetJS / xlsx (CDN) | Exportación a Excel |
| Google Apps Script | Backend / base de datos |
| Google Sheets | Almacenamiento de datos |
| Cloudflare Pages | Hosting |
| Canvas API | Animación de la ruleta |

---

## 👤 Autor

**Jhoeliel Palma** — Arquitecto de Soluciones Digitales  
Chimbote, Perú
