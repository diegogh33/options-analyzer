# 📊 Options Analyzer · IBKR

Herramienta personal de análisis de opciones financieras sobre Interactive Brokers, con generación de informes institucionales, histórico sincronizado con Notion y comparación de operaciones asistida por Claude.

**URL:** https://diegogh33.github.io/options-analyzer

---

## Qué hace

- **Análisis rápido** de cualquier operación de opciones (Long/Short Call/Put) con score 0–100, sugerencias específicas y orden IBKR lista para ejecutar
- **Informe institucional** de 11 secciones generado por Claude con datos de earnings en tiempo real (Finnhub), precio introducido por el usuario y análisis completo del subyacente
- **Histórico** de los últimos 90 días con acceso a informes completos
- **Comparación** de hasta 3 operaciones con recomendación explícita de Claude
- **Sincronización con Notion** automática con un clic

---

## Arquitectura y servicios

| Servicio | Rol | Coste |
|---|---|---|
| **GitHub Pages** | Hosting del HTML estático | Gratis |
| **Cloudflare Worker** | Proxy entre el HTML y las APIs (evita CORS) | Gratis (100K req/día) |
| **Anthropic API** | Generación de informes institucionales y comparaciones | ~$0.03–0.05 por informe |
| **Finnhub API** | Fechas de earnings en tiempo real | Gratis (60 req/min) |
| **Notion API** | Almacenamiento persistente del histórico | Gratis |

### Por qué este diseño

El HTML no puede llamar directamente a `api.anthropic.com` desde el navegador por restricciones CORS. El Worker de Cloudflare actúa como intermediario: recibe la petición del HTML, añade las credenciales almacenadas de forma segura como variables de entorno, y reenvía a Anthropic o Notion según la ruta:

- Ruta `/` → Anthropic API (informes y comparaciones)
- Ruta `/notion/*` → Notion API (guardar y leer histórico)

---

## Credenciales y variables de entorno

Todas las credenciales sensibles están en el Worker de Cloudflare (**nunca en el código fuente**):

| Variable | Dónde obtenerla |
|---|---|
| `ANTHROPIC_API_KEY` | [console.anthropic.com](https://console.anthropic.com) → API Keys |
| `NOTION_TOKEN` | [notion.so/my-integrations](https://www.notion.so/my-integrations) → Options Analyzer → Token |

La **Finnhub API key** sí está en `index.html` (variable `FINNHUB_KEY`) porque es de solo lectura y no tiene permisos de escritura.

---

## Estructura en Notion

Página raíz: **📊 Options Trading** (en la raíz del workspace)

Contiene dos bases de datos:
- **📋 Análisis de Opciones** — una fila por cada análisis rápido generado
- **📄 Informes Institucionales** — una página por cada informe de 11 secciones

La integración de Notion llamada **Options Analyzer** debe tener acceso a ambas bases de datos (Settings → Connections en cada una).

IDs de las bases de datos (definidos en `index.html`):
```
NOTION_DS_ANALYSIS = '18388721bb10484fb9102ec6503cb057'
NOTION_DS_REPORTS  = '5590f9b865774b7bb78175af3eb03e78'
```

---

## Flujo de uso

### Análisis rápido
1. Selecciona el tipo de operación (Long/Short Call/Put)
2. Introduce ticker, precio actual (lo ves en IBKR en tiempo real), IV
3. Introduce strike, DTE, prima, delta (cópialo de la cadena de IBKR), contratos y capital total
4. Selecciona tendencia de mercado y catalizador próximo
5. Escribe la tesis en una línea — **importante** para comparaciones posteriores
6. Elige si quieres informe estándar o extendido
7. Pulsa **GENERAR ANÁLISIS**

El sistema evalúa 7 criterios (delta, DTE, IV, ARORC, moneyness, alineación macro, peso en cartera) y genera un score 0–100 con sugerencias, orden IBKR y gestión de la posición.

### Informe institucional (11 secciones)
Se genera automáticamente si seleccionas Estándar o Extendido. Secciones:
1. Descripción del negocio
2. Foso competitivo
3. Equipo directivo
4. Estrategia y catalizadores
5. Riesgos principales
6. Factores idiosincráticos
7. Situación financiera
8. Valoración (usando el precio introducido como referencia)
9. Earnings y catalizadores temporales (datos reales de Finnhub)
10. Stress tests (bajista / base / alcista)
11. Veredicto final: EJECUTAR / ESPERAR / DESCARTAR con ajustes sugeridos

**Coste:** ~$0.03–0.05 por informe (Anthropic API, modelo claude-sonnet)

### Guardar en Notion
Pulsa **GUARDAR EN NOTION** tras generar. El análisis y el informe se guardan en las dos bases de datos de Notion y quedan accesibles desde cualquier dispositivo.

### Comparación de operaciones
1. Ve a **HISTÓRICO** → activa **Modo comparación**
2. Selecciona 2 o 3 análisis
3. Ve a la pestaña **COMPARAR** y pulsa el botón

Claude recibe todos los datos (incluido el extracto del informe institucional si existe) y devuelve: su elección, justificación con datos concretos, análisis de cada operación, qué cambiaría y conclusión.

---

## Cómo actualizar el código

```bash
# Desde la carpeta del repositorio local
cp ~/Downloads/index.html index.html
git add index.html
git commit -m "descripción del cambio"
git push
```

GitHub Pages actualiza en ~1-2 minutos. Usar **Ctrl+Shift+R** para forzar recarga sin caché.

Para cambios en el Worker:
1. Editar `worker.js`
2. Ir a [dash.cloudflare.com](https://dash.cloudflare.com) → Workers & Pages → `options-proxy` → Edit code
3. Pegar el nuevo código → Deploy (instantáneo)

---

## Costes estimados

| Operación | Coste aprox. |
|---|---|
| Análisis rápido | $0.00 |
| Earnings via Finnhub | $0.00 |
| Informe estándar (11 secciones) | $0.03–0.05 |
| Comparación de 2–3 operaciones | $0.01–0.02 |
| Guardar en Notion | <$0.01 |

Con $10 de saldo en Anthropic: ~150–200 informes completos.

---

## Limitaciones

- El **histórico local** (localStorage) es por dispositivo — móvil y PC tienen copias separadas. Notion es la fuente común tras sincronizar.
- Los **informes** solo se pueden leer en el dispositivo donde se generaron (localStorage). La lectura desde Notion no está implementada.
- El **precio del subyacente** lo introduce el usuario desde IBKR — no se verifica automáticamente para evitar costes de API.
