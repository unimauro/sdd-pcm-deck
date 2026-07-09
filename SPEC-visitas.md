# SPEC — Dashboard de Transparencia: Visitas al Estado

> Especificación ejecutable (Spec-Driven Development). Este archivo es la **fuente de la
> verdad**: un agente de código (Claude Code) debe poder construir el producto completo a
> partir de él. Toda decisión no escrita aquí se resuelve a favor de la **simplicidad**, el
> **bajo costo** y la **protección de datos personales**.

---

## 0. Metadatos

| Campo | Valor |
|---|---|
| Producto | Dashboard público "¿Quién visita al Estado?" |
| Dueño | Carlos Mauro Cárdenas · carlos@cardenas.pe |
| Contexto | Demo en vivo — Jornada Técnica SGTD-PCM (ENIA/ENGD 2026-2030) |
| Licencia datos | Información pública, Ley 27806 (Transparencia y Acceso a la Información) |
| Estado | Datos reales, anonimizados |

---

## 1. Objetivo

Construir un **dashboard estático de una sola página** que visualice el *Registro de Visitas
en Línea* del Estado peruano de forma clara, honesta y reutilizable. Debe:

- Cargar rápido, funcionar **sin backend** y costar **US$ 0** de hosting.
- Comunicar los patrones reales (volumen, motivos, horarios, funcionarios más visitados).
- **Etiquetar siempre** la fuente, la cobertura y las limitaciones del dato.
- Ser **replicable** por cualquier entidad cambiando solo la carpeta de datos.

No-objetivos (fuera de alcance): login, base de datos, edición de datos, exponer datos
personales, scraping automatizado de la plataforma origen.

---

## 2. Fuente de datos (real)

- **Origen:** `datosabiertos.gob.pe` → dataset *Reporte de Registro de Visitas en Línea*.
  También exportable manualmente (botón "Excel") desde `visitas.servicios.gob.pe/consultas`
  por entidad. **Nunca** automatizar contra el reCAPTCHA de esa plataforma: la descarga es
  **manual** por una persona.
- **Formato:** un archivo `.xlsx` por mes. Fila 1 = título; **fila 2 = encabezados**; datos
  desde la fila 3.
- **Cobertura actual disponible:** enero 2025 – mayo 2026 (17 archivos). En datos abiertos
  hoy solo publica el **Gobierno Regional de Loreto**; otras entidades (p. ej. PCM) se
  obtienen por export manual. **Esto es parte del mensaje, no un defecto.**

### Esquema de columnas (exacto, en español)

```
Fecha de Registro | Fecha de Visita | Entidad visitada | Visitante |
Documento del visitante | Entidad del visitante | Funcionario visitado |
Hora Ingreso | Hora Salida | Motivo | Lugar específico | Observación
```

- Fechas en formato `DD/MM/AAAA`. Horas en `HH:MM`.
- `Documento del visitante` viene como `"DNI 12345678"` → **PII, se elimina** (ver §3).
- `Motivo` viene como `"<categoría> - <detalle>"` → usar solo la **categoría** (antes de `-`).
- `Entidad del visitante` viene como `"<tipo> - <detalle>"` → usar solo el **tipo**
  (Persona natural | Entidad pública | Entidad privada).

---

## 3. Reglas de privacidad — NO NEGOCIABLE

Estas reglas se validan con un **agente adversarial** antes del deploy (§8):

1. **Eliminar** por completo, y no publicar bajo ninguna forma: `Documento del visitante`
   (DNI) y `Visitante` (nombre de la persona que visita).
2. El **funcionario visitado** SÍ es información pública (ejerce función pública) → puede
   mostrarse agregado.
3. El dashboard consume **solo JSON agregados** (conteos), nunca la microdata fila por fila.
   Ningún registro individual llega al navegador.
4. Prohibido cualquier campo, tooltip o export que permita reidentificar a un visitante.

---

## 4. ETL (`etl/build.py`)

Script Python reproducible que:

1. Lee todos los `data/raw/*.xlsx` (encabezados en fila 2).
2. Normaliza: fechas → `AAAA-MM`; `Motivo` → categoría; `Entidad del visitante` → tipo;
   calcula duración = `Hora Salida − Hora Ingreso` (descartar negativos y > 12 h).
3. **Anonimiza**: descarta DNI y nombre del visitante en el primer paso.
4. Emite **`site/data/visitas_agg.json`** con esta forma (ejemplo con datos reales):

```json
{
  "_meta": {
    "fuente": "Registro de Visitas en Línea — datosabiertos.gob.pe (SGTD-PCM)",
    "periodo": "enero 2025 – mayo 2026",
    "cobertura": "Entidades que publican en la plataforma (hoy: GORE Loreto)",
    "total_visitas": 57738,
    "anonimizado": "Sin DNI ni nombres de visitantes",
    "fecha_corte": "2026-05-31"
  },
  "por_mes":            { "2025-01": 4368, "...": 0 },
  "top_entidades":      { "GOBIERNO REGIONAL DE LORETO": 57738 },
  "top_motivos":        { "Otros Motivos": 44064, "Reunión de trabajo": 13346 },
  "por_hora":           { "8": 7743, "9": 11540, "10": 12798 },
  "por_dia_semana":     { "Lun": 12112, "Mar": 12384, "...": 0 },
  "tipo_visitante":     { "Persona natural": 38106, "Entidad pública": 10762 },
  "top_funcionarios":   { "Apellidos Nombre (Entidad)": 16113 },
  "duracion_mediana_min": 146
}
```

5. El JSON debe traer siempre `_meta` con **fuente, periodo, cobertura y fecha_corte**.

---

## 5. Vistas del dashboard (una sola página)

Orden y contenido:

1. **Cabecera**: título, subtítulo y una franja `_meta` (fuente + cobertura + corte, visible).
2. **KPIs** (tarjetas): total de visitas · duración mediana (formato `Xh Ym`) · día pico ·
   hora pico. Números con `tabular-nums`.
3. **Serie mensual** (barras): visitas por mes; resaltar el máximo. Eje X = meses; separar años.
4. **Motivos** (barras horizontales, % del total): incluir la **advertencia de calidad** si
   una categoría domina (hoy "Otros Motivos" ≈ 76 %).
5. **Horario de atención** (barras): visitas por hora de ingreso (8–19 h).
6. **Día de la semana** (barras): Lun–Dom.
7. **Tipo de visitante** (dona o barras): persona natural / entidad pública / privada.
8. **Top funcionarios visitados** (tabla o barras): top 15, con conteo.
9. **Pie**: nota metodológica + enlace a la fuente + autor/contacto.

Cada gráfico lleva **título, unidad y una línea de fuente**. Si un dato es estimado o parcial,
marcarlo explícitamente (badge o nota).

---

## 6. Stack técnico

- **Frontend:** un solo `index.html` con HTML + CSS + JS vanilla + **Apache ECharts**
  (vía `<script>` local o CDN permitido). Sin framework, sin build de JS.
- **Datos:** `fetch('data/visitas_agg.json')`. Nada de llamadas externas en runtime.
- **Hosting:** GitHub Pages (rama `main`, raíz, con `.nojekyll`). Costo US$ 0.
- **ETL:** Python 3 + `openpyxl`. `data/raw/` y `data/processed/` van en `.gitignore`
  (la microdata NO se versiona); solo se versiona `site/data/visitas_agg.json`.

---

## 7. Diseño

- Tema oscuro institucional. Paleta: fondo `#0d1a2b`, acento ámbar `#f0a830`, secundario
  teal `#33c2af`, alerta coral `#ef6461`, neutro con leve sesgo azul.
- Tipografía de sistema; números con `font-variant-numeric: tabular-nums`.
- Responsive (móvil primero), sin scroll horizontal; gráficos dentro de contenedores con
  `overflow-x:auto`.
- **SEO + Open Graph** (compartible en WhatsApp): `og:title`, `og:description`, `og:image`
  1200×630, `theme-color`.
- Accesible: contraste AA, foco visible, respeta `prefers-reduced-motion`.

---

## 8. Calidad y agentes adversariales (gates previos al deploy)

Antes de publicar, ejecutar agentes de verificación que **fallen el build** si:

- **PII:** aparece cualquier DNI (`/\bDNI\b/`, patrones de 8 dígitos) o nombre de visitante
  en el HTML servido o en el JSON. → *bloquea deploy*.
- **Secretos/infra:** hay claves, tokens o IPs de servidores en el repo. → *bloquea*.
- **Fuente:** algún gráfico carece de fuente/fecha_corte. → *bloquea*.
- **Números:** los totales del JSON no cuadran con la suma de `por_mes`. → *bloquea*.
- **Tests unitarios** (`pytest`) del ETL: parseo de fecha, cálculo de duración (ignora
  negativos y outliers), extracción de categoría de motivo, y **aserción de anonimización**
  (el JSON de salida no contiene la columna DNI ni nombres).

Un segundo agente (revisor) intenta *romper* el resultado: inyectar una fila con DNI y
comprobar que el pipeline la anonimiza; verificar que abrir el sitio no filtra microdata.

---

## 9. Entregables y criterios de aceptación

- [ ] `etl/build.py` corre de cero y produce `site/data/visitas_agg.json`.
- [ ] `index.html` renderiza las 9 secciones con los datos reales.
- [ ] Cero PII en el sitio y en el JSON (verificado por el gate).
- [ ] Cada gráfico cita fuente y fecha de corte.
- [ ] Suite de tests en verde.
- [ ] Desplegado en GitHub Pages, carga < 2 s, funciona offline tras la primera carga.
- [ ] Franja `_meta` visible con la limitación de cobertura (hoy: una sola entidad).

---

## 10. Qué NO hacer

- ❌ Inventar o rellenar datos faltantes. Si no hay dato, se dice "sin dato".
- ❌ Publicar DNI, nombres de visitantes o microdata.
- ❌ Derivar afirmaciones no soportadas por los agregados.
- ❌ Automatizar descargas contra el reCAPTCHA de la plataforma origen.
- ❌ Exponer claves, IP del servidor o nombres de otros servicios.
- ❌ Presentar el dato de una sola entidad como si fuera nacional: **etiquetar la cobertura**.

---

_Fin del spec. Construir exactamente lo aquí descrito; ante ambigüedad, priorizar privacidad,
bajo costo y honestidad del dato._
