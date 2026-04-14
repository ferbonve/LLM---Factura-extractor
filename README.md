# 📄 FER Agent Contable --- Extracción automática de datos fiscales desde PDFs

Este proyecto implementa un **agente inteligente basado en LangGraph +
LLMs (Groq)** que permite **extraer automáticamente información fiscal
desde documentos PDF** como:

-   Facturas
-   Recibos
-   Tickets
-   Liquidaciones
-   Comprobantes de pago

El sistema procesa múltiples archivos en lote, interpreta su contenido y
devuelve datos estructurados en formato JSON listos para análisis
contable o integración en pipelines de datos.

------------------------------------------------------------------------

# 🚀 Features principales

✅ Extracción automática de texto desde PDFs\
✅ Interpretación fiscal con LLM\
✅ Validación matemática de totales\
✅ Procesamiento batch de documentos\
✅ Output estructurado en JSON\
✅ Arquitectura basada en agentes (LangGraph)\
✅ Modular y extensible

------------------------------------------------------------------------

# 🧠 Arquitectura del agente

Flujo del sistema:

PDF → Extractor de texto → LLM parser fiscal → Verificador matemático →
JSON final

Componentes principales:

  Nodo            Función
  --------------- ----------------------------------
  PDF extractor   Extrae texto usando pdfplumber
  Agent           Interpreta contenido fiscal
  Verificador     Controla consistencia matemática
  Output          Devuelve JSON estructurado

------------------------------------------------------------------------

# 📦 Tecnologías utilizadas

-   Python
-   LangGraph
-   LangChain
-   Groq API
-   pdfplumber
-   JSON parsing

------------------------------------------------------------------------

# 📂 Estructura esperada del proyecto

    project/
    │
    ├── facts/                      # Carpeta con PDFs a procesar
    ├── FER_AGENT_Contable.ipynb
    ├── facturas_parser_prompt.txt
    └── README.md

------------------------------------------------------------------------

# ⚙️ Instalación

Instalar dependencias:

``` bash
pip install langchain-groq
pip install pdfplumber
pip install langgraph
```

------------------------------------------------------------------------

# 🔑 Configuración

Definir tu API Key de Groq:

``` python
groq_api_key = "YOUR_API_KEY"
```

Recomendado usar variables de entorno:

``` bash
export GROQ_API_KEY="YOUR_API_KEY"
```

------------------------------------------------------------------------

# ▶️ Uso

Colocar los PDFs dentro de la carpeta:

    facts/

Ejecutar el notebook o script:

``` python
resultado = app.invoke({
    "messages": [
        HumanMessage(content="Extraeme el total de esta factura")
    ],
    "pdf_path": path
})
```

Salida esperada:

``` json
{
  "moneda": "ARS",
  "neto_gravado_21": 25206.61,
  "iva_21": 5293.39,
  "total": 30500
}
```

------------------------------------------------------------------------

# 🧪 Procesamiento batch

El sistema permite recorrer automáticamente todos los PDFs:

``` python
for filename in os.listdir(folder):
```

Ideal para:

-   auditorías masivas
-   pipelines contables
-   automatización administrativa
-   ETL fiscal

------------------------------------------------------------------------

# 🧮 Verificador matemático

El agente incluye un módulo que valida:

TOTAL = NETO + IVA + PERCEPCIONES + IMPUESTOS

Detecta inconsistencias y mejora confiabilidad del output.

------------------------------------------------------------------------

# 📊 Casos de uso

Este proyecto puede utilizarse para:

-   Automatización contable
-   Data extraction pipelines
-   Auditoría documental
-   Preprocesamiento para BI
-   Entradas para sistemas ERP
-   Dataset labeling fiscal

------------------------------------------------------------------------

# 🔮 Mejoras futuras posibles

Ideas para evolucionarlo:

-   soporte OCR (facturas escaneadas)
-   API REST
-   exportación a Excel
-   integración con bases SQL
-   UI web
-   clasificación automática de comprobantes
-   confidence score por campo

------------------------------------------------------------------------

# 👨‍💻 Autor

Proyecto desarrollado como agente experimental de **extracción fiscal
inteligente basada en LLMs + grafos de ejecución**.
