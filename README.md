# 🧾 facturai — Extraction and Validation of receipts using AI Agents

Agente construido con **LangGraph** y **Groq** que extrae y valida automáticamente los valores fiscales de facturas argentinas en PDF. Procesa lotes de comprobantes (facturas, notas de débito/crédito, recibos) y devuelve un JSON estructurado por factura, con validación contable para asegurarse mayor precisión en los datos extraidos de la factura.

---

## 🎯 ¿Qué problema resuelve?
Hay dos empleados que 

Cargar facturas manualmente a un sistema contable es lento y propenso a errores. Este agente automatiza la extracción de todos los campos fiscales relevantes para Argentina (netos por alícuota, IVAs, percepciones) y verifica que los números sean matemáticamente consistentes antes de darlo por válido.

---

## 🏗️ Arquitectura del grafo

```
START
  │
  ▼
[.pdf text extractor]   ← Extrae el texto del PDF con pdfplumber
  │
  ▼
[agent]                 ← LLM de Groq + system prompt especializado → JSON fiscal
  │
  ▼
[verificador]           ← Valida las relaciones matemáticas entre campos
  │
  ▼
END
```

Cada nodo recibe y retorna el `AgentState` completo:

```python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    pdf_path: str     # Ruta al PDF
    pdf_text: str     # Texto extraído
```

---

## 🧠 El system prompt: el corazón del agente

El archivo `facturas_parser (system_prompt).txt` define exactamente qué debe extraer el LLM y cómo debe razonar. Sus pilares son:

### Campos fiscales a extraer

El prompt instruye al modelo a identificar cada campo por su **significado y alícuota**, no por su nombre exacto, cubriendo todas las variantes que pueden aparecer en comprobantes argentinos:

| Campo JSON | Variantes reconocidas en el documento |
|---|---|
| `neto_total` | Subtotal, Neto, Importe neto, Neto total... |
| `neto_gravado_21` | Neto gravado 21%, Base imponible 21%... |
| `neto_gravado_1050` | Neto gravado 10.5%, Gravado 10.5%... |
| `neto_gravado_27` | Neto gravado 27%, Base imponible 27%... |
| `no_gravado` | No gravado, Exento, Operaciones exentas... |
| `iva_21` | IVA 21%, I.V.A. 21%, IVA alícuota 21%... |
| `iva_1050` | IVA 10.5%, IVA 10,50%... |
| `iva_27` | IVA 27%, I.V.A. 27%... |
| `percepciones_iva` | Percepción IVA 3%, Percepción IVA 6%, Ret. IVA... |
| `percepciones_ganancias` | Percepción Ganancias, Ret. Ganancias... |
| `percepciones_iibb` | Percepción IIBB, Percepción Ing. Brutos... |
| `total` | Total, Total a pagar, Importe total... |

### Reglas matemáticas que el LLM debe respetar

El prompt le enseña al modelo las tres relaciones fiscales que siempre deben cumplirse:

```
1. neto_total = neto_gravado_21 + neto_gravado_1050 + neto_gravado_27 + no_gravado

2. iva_21   = neto_gravado_21   × 0.21
   iva_1050 = neto_gravado_1050 × 0.105
   iva_27   = neto_gravado_27   × 0.27

3. total = neto_total + iva_21 + iva_1050 + iva_27
           + percepciones_iva + percepciones_ganancias + percepciones_iibb
```

Si los valores del documento no cumplen estas reglas, el modelo debe reportarlos tal como aparecen y bajar el `confidence` a `"baja"` — **nunca inventar valores para que las fórmulas cierren**.

### Output del LLM

El modelo devuelve exclusivamente JSON válido (sin markdown, sin texto adicional):

```json
{
  "moneda": "ars",
  "confidence": "alta",
  "neto_total": 64236.81,
  "neto_gravado_21": 64236.81,
  "neto_gravado_1050": null,
  "neto_gravado_27": null,
  "no_gravado": null,
  "iva_21": 13489.73,
  "iva_1050": null,
  "iva_27": null,
  "percepciones_iva": null,
  "percepciones_ganancias": null,
  "percepciones_iibb": null,
  "total": 77726.54
}
```

---

## ✅ Nodo verificador: segunda capa de validación

Aunque el LLM ya fue instruido para verificar las relaciones matemáticas, el nodo `verificador_matematico` las re-chequea en Python con una **tolerancia de $0.10** para cubrir diferencias de redondeo. Si detecta inconsistencias, las lista en `errores_validacion` y marca `valido: False`.

Esto agrega una capa de seguridad independiente del modelo: aunque el LLM cometa un error de redondeo o alucinación, el verificador lo captura.

---

## 🔁 Fallback automático entre modelos

Si un modelo de Groq falla por rate limit o error técnico, el agente prueba automáticamente el siguiente:

```python
modelos = [
    "llama-3.3-70b-versatile",
    "meta-llama/llama-4-scout-17b-16e-instruct",
    "qwen/qwen3-32b",
    "llama-3.1-8b-instant"
]
```

El agente también detecta rate limits que vienen embebidos en el texto de la respuesta (no solo como excepciones), buscando frases como `"rate limit"` o `"error code: 429"`.

---

## ⚙️ Instalación y uso

### 1. Instalar dependencias

```bash
pip install langchain-groq pdfplumber langgraph langchain-core pandas openpyxl
```

### 2. Obtener una API key de Groq

Gratis en [console.groq.com](https://console.groq.com). Configurarla en el notebook:

```python
groq_api_key = "tu_api_key_aqui"
```

### 3. Colocar las facturas

Crear una carpeta `facts/` y colocar los PDFs adentro.

### 4. Ejecutar el notebook

Correr todas las celdas. Al finalizar se genera `output.xlsx` con todos los resultados.

---

## 📊 Output por factura

```python
{
  'moneda': 'ARS',
  'confidence': 'alta',
  'neto_total': 64236.81,
  'neto_gravado_21': 64236.81,
  'neto_gravado_1050': None,
  'neto_gravado_27': None,
  'no_gravado': None,
  'iva_21': 13489.73,
  'iva_1050': None,
  'iva_27': None,
  'percepciones_iva': None,
  'percepciones_ganancias': None,
  'percepciones_iibb': None,
  'total': 77726.54,
  'errores_validacion': [],   # lista vacía = sin errores
  'valido': True,
  'FileName': '2026-02-01_..._BONVECCHIATO.pdf'
}
```

---

## ⚠️ Casos especiales manejados

| Situación | Comportamiento |
|---|---|
| PDF sin texto (escaneado) | Devuelve `{"error": "pdf_sin_texto"}` y continúa con el siguiente |
| Rate limit en Groq | Pasa automáticamente al siguiente modelo de la lista |
| Error técnico en un modelo | Idem |
| JSON con backticks en la respuesta | El verificador los limpia antes de parsear |
| Discrepancia real en el documento | El LLM reporta los valores tal como están y baja `confidence` a `"baja"` |

---

## 📁 Estructura del proyecto

```
/
├── FER_AGENT_COntable.ipynb                  # Notebook principal
├── facturas_parser (system_prompt).txt       # System prompt del LLM
├── facts/                                    # Facturas PDF a procesar
│   ├── factura1.pdf
│   └── factura2.pdf
└── output.xlsx                               # Resultados exportados (generado al correr)
```

---

## 🛠️ Stack

| Herramienta | Rol |
|---|---|
| [LangGraph](https://github.com/langchain-ai/langgraph) | Orquestación del agente como grafo de nodos |
| [Groq](https://groq.com) | Inferencia LLM de alta velocidad |
| [LangChain Core](https://github.com/langchain-ai/langchain) | Tipos de mensajes y abstracciones |
| [pdfplumber](https://github.com/jsvine/pdfplumber) | Extracción de texto de PDFs |
| [pandas](https://pandas.pydata.org) | Exportación a Excel |
