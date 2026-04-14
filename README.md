# 🧾 facturai --- Invoice values extraction and validation with AI Agent.

Agent built with **LangGraph** and **Groq** that automatically extracts
and validates fiscal values from Argentine invoices in PDF format. It
processes batches of documents (invoices, debit/credit notes, receipts)
and returns structured JSON per invoice, including accounting
validation.

------------------------------------------------------------------------

## 🎯 What problem does it solve?

Manually entering invoices into an accounting system is slow and
error‑prone. This agent automates extraction of all relevant fiscal
fields for Argentina (taxable bases by VAT rate, VAT amounts,
perceptions) and verifies that numbers are mathematically consistent
before marking them as valid.

------------------------------------------------------------------------

## 🏗️ Graph architecture

    START
      │
      ▼
    [.pdf text extractor]   ← Extracts PDF text using pdfplumber
      │
      ▼
    [agent]                 ← Groq LLM + specialized system prompt → fiscal JSON
      │
      ▼
    [validator]             ← Validates mathematical relationships between fields
      │
      ▼
    END

Each node receives and returns the full `AgentState`:

``` python
class AgentState(TypedDict):
    messages: Annotated[Sequence[BaseMessage], add_messages]
    pdf_path: str     # PDF path
    pdf_text: str     # Extracted text
```

------------------------------------------------------------------------

## 🧠 The system prompt: the heart of the agent

The file `facturas_parser (system_prompt).txt` defines exactly what the
LLM must extract and how it should reason. Its pillars are:

### Fiscal fields to extract

The prompt instructs the model to identify each field by its **meaning
and tax rate**, not by exact label wording, covering all variants that
may appear in Argentine documents:

  JSON Field                 Recognized document variants
  -------------------------- ----------------------------------------
  `neto_total`               Subtotal, Net amount, Net total...
  `neto_gravado_21`          Taxable base 21%, Net taxed 21%...
  `neto_gravado_1050`        Taxable base 10.5%, Net taxed 10.5%...
  `neto_gravado_27`          Taxable base 27%, Net taxed 27%...
  `no_gravado`               Non‑taxed, Exempt operations...
  `iva_21`                   VAT 21%
  `iva_1050`                 VAT 10.5%
  `iva_27`                   VAT 27%
  `percepciones_iva`         VAT perception
  `percepciones_ganancias`   Income tax perception
  `percepciones_iibb`        Gross income perception
  `total`                    Total amount payable

### Mathematical rules the LLM must respect

The prompt teaches the model three fiscal relationships that must always
hold:

    1. neto_total = neto_gravado_21 + neto_gravado_1050 + neto_gravado_27 + no_gravado

    2. iva_21   = neto_gravado_21   × 0.21
       iva_1050 = neto_gravado_1050 × 0.105
       iva_27   = neto_gravado_27   × 0.27

    3. total = neto_total + iva_21 + iva_1050 + iva_27
               + percepciones_iva + percepciones_ganancias + percepciones_iibb

If document values do not satisfy these rules, the model must report
them exactly as shown and lower `confidence` to `"low"` --- **never
invent values to force formulas to match**.

### LLM output

The model returns strictly valid JSON (no markdown, no extra text):

``` json
{
  "moneda": "ars",
  "confidence": "high",
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

------------------------------------------------------------------------

## ✅ Validator node: second validation layer

Even though the LLM is already instructed to verify mathematical
relationships, the `verificador_matematico` node rechecks them in Python
with a **\$0.10 tolerance** to account for rounding differences. If
inconsistencies are detected, they are listed in `errores_validacion`
and `valido: False` is set.

This adds a model‑independent safety layer: even if the LLM makes a
rounding mistake or hallucination, the validator catches it.

------------------------------------------------------------------------

## 🔁 Automatic fallback between models

If a Groq model fails due to rate limits or technical issues, the agent
automatically tries the next one:

``` python
modelos = [
    "llama-3.3-70b-versatile",
    "meta-llama/llama-4-scout-17b-16e-instruct",
    "qwen/qwen3-32b",
    "llama-3.1-8b-instant"
]
```

The agent also detects rate limits embedded inside response text (not
only exceptions), searching for phrases like `"rate limit"` or
`"error code: 429"`.

------------------------------------------------------------------------

## ⚙️ Installation and usage

### 1. Install dependencies

``` bash
pip install langchain-groq pdfplumber langgraph langchain-core pandas openpyxl
```

### 2. Get a Groq API key

Free at https://console.groq.com

Configure it in the notebook:

``` python
groq_api_key = "your_api_key_here"
```

### 3. Place invoices

Create a `facts/` folder and place the PDFs inside.

### 4. Run the notebook

Run all cells. At the end, `output.xlsx` will be generated with results.

------------------------------------------------------------------------

## 📊 Output per invoice

``` python
{
  'moneda': 'ARS',
  'confidence': 'high',
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
  'errores_validacion': [],
  'valido': True,
  'FileName': '2026-02-01_..._BONVECCHIATO.pdf'
}
```

------------------------------------------------------------------------

## ⚠️ Special handled cases

  -----------------------------------------------------------------------
  Situation                           Behavior
  ----------------------------------- -----------------------------------
  PDF without text (scanned)          Returns
                                      `{"error": "pdf_sin_texto"}` and
                                      continues

  Groq rate limit                     Automatically switches to next
                                      model

  Technical error in model            Same fallback

  JSON wrapped in backticks           Validator cleans before parsing

  Real discrepancy in document        LLM reports values as‑is and lowers
                                      confidence
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 📁 Project structure

    /
    ├── FER_AGENT_COntable.ipynb
    ├── facturas_parser (system_prompt).txt
    ├── facts/
    │   ├── factura1.pdf
    │   └── factura2.pdf
    └── output.xlsx

------------------------------------------------------------------------

## 🛠️ Stack

  Tool             Role
  ---------------- ---------------------------------
  LangGraph        Graph‑based agent orchestration
  Groq             High‑speed LLM inference
  LangChain Core   Message types and abstractions
  pdfplumber       PDF text extraction
  pandas           Excel export
