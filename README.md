# Fiscal Automation Framework (Spain · self-employed) — Showcase

**Engineered by [Kamil Slodki](https://www.linkedin.com/in/kamilslodki/)** · Valencia, España · Remote-first

**Framework de automatización fiscal para autónomo español en régimen estimación directa simplificada + IVA general: pipeline end-to-end de ingesta documental (PDF · imagen con OCR · Excel) → clasificación fiscal automatizada por categoría AEAT → cálculo de IVA (base × tipo con tolerancia 0.02 €) e IRPF (rendimiento neto acumulado YTD) usando aritmética Decimal exacta (NO float) → generación automatizada de modelos AEAT trimestrales (303 IVA · 130 IRPF pagos fraccionados) y anuales (349 intracomunitarias · 390 resumen anual IVA) → validación en 4 capas (formal RD 1619/2012 + realidad + censal + duplicados SHA-256) → reconciliación ledger ↔ extracto bancario con trazabilidad documento → asiento → línea → 6 artefactos de auditoría por trimestre (manifest.json + validity_report + OCR audit + classification_audit + reconciliation + release_report) con hash SHA-256 de integridad. Stack Python 3.11 + Pydantic 2.0 + Typer CLI + pdfplumber + pytesseract + Pillow + pandas + Decimal + JSON Schema + pytest + Rich. FSM de 6 estados (RAW → PARSED → NORMALIZED → VALIDATED → RECONCILED → APPROVED · BLOCKED) con 9 invariantes documentadas (IN-001..IN-009) validables vía grep. Diseñado como herramienta de uso propio del autor, no como SaaS multiusuario.**

[![Type](https://img.shields.io/badge/type-showcase%20%C2%B7%20producto%20propio-blue.svg)](#sobre-este-showcase)
[![Status](https://img.shields.io/badge/status-producci%C3%B3n%20activa-success.svg)](#estado)
[![Role](https://img.shields.io/badge/role-B2B%20Delivery%20Engineer%20%C2%B7%20Fiscal%20Automation-blueviolet.svg)](#contacto)
[![Engineering Doctrine](https://img.shields.io/badge/engineering%20doctrine-5%20protocols-9cf.svg)](#sistema-operativo-de-ingeniería-del-autor)
[![Domain](https://img.shields.io/badge/domain-fiscal%20compliance%20%C2%B7%20AEAT%20models-9cf.svg)](#funcionalidad)
[![Stack](https://img.shields.io/badge/stack-Python%20%7C%20Pydantic%20%7C%20Decimal%20%7C%20Tesseract%20%7C%20JSON%20Schema-informational.svg)](#stack-técnico)

<sub>Última actualización: 2026-05-18</sub>

> **Nota sobre alcance del showcase**: este repositorio describe la **arquitectura, algoritmos y patrones técnicos** del framework. Datos personales fiscales, cifras absolutas, identificadores AEAT del autor, nombres de clientes/proveedores reales y artefactos de auditoría con datos reales **NO se publican** por consideraciones de privacidad personal. El código y la documentación técnica aquí mostrados son el subset genérico defendible que demuestra la disciplina de ingeniería sin comprometer información fiscal personal.

---

## Sobre este showcase

Sistema de gestión fiscal integral para **autónomo español en régimen estimación directa simplificada + IVA general** que automatiza el ciclo completo desde la ingesta de documentos (facturas emitidas, recibidas, justificantes bancarios) hasta la generación de modelos AEAT y la auditoría pre-presentación.

**Es herramienta de uso propio del autor**, no producto SaaS multiusuario. El interés del showcase es **técnico-arquitectónico**: la disciplina de ingeniería aplicada a un dominio altamente regulado (fiscalidad española) con cero margen de error en cálculos.

**Estado**: producción activa generando modelos AEAT trimestrales en entorno real con revisión humana antes de presentar.

---

## El problema

La gestión fiscal de un autónomo en régimen estimación directa simplificada + IVA general requiere:

1. **Recopilar** facturas emitidas + recibidas + justificantes bancarios + extractos cada trimestre.
2. **Clasificar** cada gasto contra normativa (deducibilidad IRPF según art. 30-32 LIRPF · deducibilidad IVA según art. 96 LIVA · gasto vs inversión amortizable).
3. **Calcular** IVA repercutido vs deducible, IRPF acumulado YTD aplicando las reglas de exención previstas en RIRPF para pagos fraccionados, amortizaciones lineales por coeficiente.
4. **Generar** modelos AEAT (303 trimestral · 130 trimestral · 349 si aplica · 390 anual).
5. **Reconciliar** ledger contable contra extracto bancario (trazabilidad documento → asiento → línea).
6. **Auditar** antes de presentar (4 capas: formal · realidad · censal · duplicados).

Hacerlo manualmente con Excel es factible pero error-prone: un error en una base imponible puede causar diferencia con AEAT, recargo o regularización. Antipatrones sistémicos al automatizarlo:

| Antipatrón sistémico de la industria | Patrón aplicado en este framework |
|---|---|
| Aritmética con `float` para cálculos fiscales → errores de redondeo acumulados que pueden disparar regularizaciones con AEAT por diferencias de céntimos | **`Decimal` exclusivamente** en toda la math layer · tolerancia explícita de 0.02 € para reconciliación · cuotas redondeadas según directiva AEAT |
| OCR con un solo patrón regex por proveedor → falla cuando proveedor cambia formato de factura | **Doble lectura obligatoria**: 2 patrones regex distintos deben coincidir (check #1 de los 5 alto nivel) · fallback a `[OCR_INCOMPLETO]` con bloqueo del trimestre |
| Sin validación de rango por proveedor → factura inflada o duplicada pasa desapercibida | **Rango esperado por proveedor** (check #3): si una factura de proveedor X cae fuera del rango histórico ±2σ, se marca para revisión manual |
| LLM para clasificación fiscal → alucina deducibilidades dudosas | **Clasificación determinista por reglas** documentadas (NO LLM): mapping `proveedor → categoría AEAT` versionado · revisión manual obligatoria de gastos en zona gris |
| Modelo único genérico para todas las facturas → no detecta periodificación errónea | **Check de fecha archivo vs fecha interna** (check #2): factura T2 enviada en T3 con fecha real T2 se asigna correctamente al periodo de devengo |
| Sin recálculo inverso → errores tipográficos manuales en base/total pasan | **Recálculo inverso** (check #5): total / IVA debe recuperar base con tolerancia 0.02 € · si no cuadra, bloqueo |
| Sin tracking de duplicados → factura duplicada inflando deducciones | **Hash SHA-256 del documento** en cada manifiesto · validación capa 4 de las 4 obligatorias |
| Reconciliación manual ledger ↔ banco → movimientos huérfanos crónicos | **Reconciliación automatizada** con tolerancia 0.02 € · detecta movimientos en extracto sin asiento + asientos sin movimiento en extracto |
| Sin auditoría pre-presentación → modelos AEAT enviados con errores que disparan regularizaciones | **Auditoría 4 capas obligatoria**: formal (RD 1619/2012 — campos obligatorios) · realidad (¿existe el CIF?) · censal (¿está de alta?) · duplicados (SHA-256) · 6 artefactos firmados antes de release |
| Cambios en normativa fiscal → código sin versionar la regla aplicable | **9 invariantes documentadas** (IN-001..IN-009) versionadas en código · cualquier cambio normativo requiere ADR + actualización de invariantes |

---

## Funcionalidad

### Modelos AEAT cubiertos

- **303** — IVA trimestral (devengado − deducible ± compensaciones rectificativas)
- **130** — IRPF pagos fraccionados (con exenciones previstas en RIRPF art. 61.3.i)
- **349** — Operaciones intracomunitarias (grupo por NIF-IVA + país)
- **390** — Resumen anual IVA
- **100** (Renta) — capacidad opcional para integración con declaración anual

### Cálculos automatizados

- **IVA**: base × tipo (0 % · 4 % · 10 % · 21 %) con tolerancia 0.02 € · deducción por proveedor y categoría (gasto operativo vs inversión amortizable).
- **IRPF**: pagos fraccionados según LIRPF art. 109 con verificación de exenciones previstas en RIRPF · reglas de deducibilidad documentadas (art. 30-32 LIRPF).
- **Amortizaciones**: coeficientes lineales por categoría (equipos informáticos · vehículos · mobiliario) · base = mín(coste real, valor máximo permitido art. 28).

### Validación en 4 capas

| Capa | Qué valida |
|---|---|
| **Formal** | Campos obligatorios según RD 1619/2012 (fecha · concepto · base imponible · tipo IVA · cuota · CIF emisor · CIF receptor · serie · número) |
| **Realidad** | El CIF existe (formato válido · checksum) |
| **Censal** | El CIF está de alta en AEAT (referencia censal cuando aplica) |
| **Duplicados** | Hash SHA-256 del documento contra manifest histórico |

### 6 artefactos de auditoría por trimestre

1. `manifest.json` — hash SHA-256 de todos los inputs + outputs del trimestre · trazabilidad completa
2. `validity_report.json` — resultado de las 4 capas con todos los warnings/errors
3. `ocr_audit.json` — métricas del OCR (confidence por documento · failed reads · patrones que no matcharon)
4. `classification_audit.json` — clasificación fiscal aplicada por factura con regla disparada
5. `reconciliation_report.json` — diff ledger ↔ extracto bancario · huérfanos en ambos sentidos
6. `release_report.md` — resumen ejecutivo para revisión humana antes de presentar a AEAT

---

## Arquitectura

### Estructura por capas

```
┌───────────────────────────────────────────────────────┐
│  Capa 7 — Release & Audit                              │
│  (manifest.json + 5 reports + release_report.md)       │
├───────────────────────────────────────────────────────┤
│  Capa 6 — AEAT Models Generation                       │
│  (303 · 130 · 349 · 390 con plantillas oficiales)      │
├───────────────────────────────────────────────────────┤
│  Capa 5 — Reconciliation                               │
│  (ledger ↔ extracto bancario · tolerance 0.02 EUR)     │
├───────────────────────────────────────────────────────┤
│  Capa 4 — Validation (4 sub-capas)                     │
│  (formal · realidad · censal · duplicados SHA-256)     │
├───────────────────────────────────────────────────────┤
│  Capa 3 — Math Engine                                  │
│  (IVA · IRPF · amortizaciones · Decimal exclusivo)     │
├───────────────────────────────────────────────────────┤
│  Capa 2 — Classification                               │
│  (mapping proveedor → categoría AEAT · determinista)   │
├───────────────────────────────────────────────────────┤
│  Capa 1 — Intake                                       │
│  (PDF + OCR + Excel + 5 checks alto nivel)             │
└───────────────────────────────────────────────────────┘
```

### FSM del flujo

```
RAW (documento ingresado)
  ↓
PARSED (lectura exitosa por intake)
  ↓
NORMALIZED (campos canónicos extraídos)
  ↓
VALIDATED (4 capas pasadas)
  ↓
RECONCILED (cuadre con banco OK)
  ↓
APPROVED (revisión humana OK) → release a AEAT
  ↓
BLOCKED (en cualquier punto si falla validación o cuadre)
```

### 9 invariantes (IN-001..IN-009)

| Invariante | Qué garantiza |
|---|---|
| IN-001 | Cuadre IVA devengado en modelo 303 = suma de cuotas en facturas emitidas del trimestre |
| IN-002 | Cuadre IVA deducible en modelo 303 = suma de cuotas en facturas recibidas (filtradas por deducibilidad) |
| IN-003 | Rendimiento neto acumulado YTD del 130 = ingresos − gastos deducibles acumulados |
| IN-004 | NIF del autor seudonimizado en todos los outputs (nunca aparece en logs ni reports públicos) |
| IN-005 | Tolerancia 0.02 € aplicada uniformemente en cálculos y reconciliación |
| IN-006 | Aritmética con `Decimal` exclusivamente · prohibido `float` en math layer |
| IN-007 | Cada documento tiene hash SHA-256 único antes de procesar |
| IN-008 | Rectificativas sin documento original → bloqueo automático |
| IN-009 | Manifest hash recalculado en cada release · cambio post-release → flag para re-auditar |

---

## Stack técnico

| Capa | Tecnología | Rol |
|---|---|---|
| **Core** | Python 3.11 | Lenguaje base |
| **Data models** | Pydantic 2.0 + JSON Schema (4 schemas) | Validación de inputs y outputs |
| **CLI** | Typer 0.9 + Rich | Comandos + UX terminal colorizado |
| **PDF** | pdfplumber | Extracción de texto de facturas PDF |
| **OCR** | pytesseract + Pillow | Fallback para imágenes y tickets escaneados |
| **Math** | `Decimal` (stdlib) + pandas | Aritmética exacta + agregaciones |
| **Excel** | openpyxl + pandas | Lectura de extractos bancarios y export de reports |
| **Validation** | JSON Schema | 4 schemas oficiales del framework |
| **Tests** | pytest | Cobertura completa de paths fiscales críticos |
| **Storage** | YAML (config) + JSON (manifests) + CSV/SQLite (ledger) | Sin BD relacional · simple y portable |

---

## Patterns técnicos diferenciadores

### 1. OCR pipeline con 5 checks de alto nivel

Para cada factura procesada por OCR:

1. **Doble lectura**: 2 patrones regex distintos deben coincidir → si no, marcar para revisión manual.
2. **Fecha archivo vs fecha interna**: detecta periodificación errónea (factura enviada en T3 con fecha real T2).
3. **Rango por proveedor**: si el importe cae fuera del rango histórico ±2σ del proveedor, marcar para revisión.
4. **Comparativa trimestre vs trimestre anterior**: si total del trimestre infla X% sobre el anterior sin razón, alerta.
5. **Recálculo inverso**: `total / (1 + tipo_iva)` debe recuperar `base` con tolerancia 0.02 € · si no, bloqueo.

### 2. Aritmética Decimal estricta

```python
# PROHIBIDO en math layer
total = base * (1 + 0.21)  # float, redondeo impreciso

# OBLIGATORIO
from decimal import Decimal, ROUND_HALF_UP
TIPO_IVA = Decimal('0.21')
TOLERANCIA = Decimal('0.02')
total = (base * (Decimal('1') + TIPO_IVA)).quantize(Decimal('0.01'), rounding=ROUND_HALF_UP)
```

Linter custom + tests aseguran que NO se cuela `float` en la math layer.

### 3. Mapping determinista proveedor → categoría AEAT

Catálogo versionado en YAML:

```yaml
providers:
  - id: provider_a
    aliases: ["Alias A", "Alias A SLU"]
    category: gasto_telefonia
    iva_deducible: true
    expected_range: { min: 30.00, max: 80.00 }
  - id: provider_b
    category: inversion_equipo_informatico
    amortizacion_coef: 0.26
    iva_deducible: true
```

Cualquier proveedor nuevo se añade explícitamente con su clasificación · NO se infiere con LLM (decisión consciente: trazabilidad fiscal > comodidad).

### 4. Reconciliación con trazabilidad completa

Cada movimiento del extracto bancario se trackea hasta el asiento contable y la línea del documento origen. Si hay diferencia > 0.02 €, queda como huérfano para revisión.

### 5. Hash SHA-256 en cada release

El manifiesto del trimestre incluye hash SHA-256 de cada input. Si después del release alguien edita un documento, el siguiente recálculo detecta el cambio y exige re-auditar.

---

## Estado y madurez técnica

- **En producción activa** generando modelos AEAT trimestrales en entorno real con revisión humana antes de cualquier presentación.
- **Suite de tests** con cobertura completa de paths fiscales críticos (cálculo IVA · IRPF · amortizaciones · reconciliación · 4 capas de validación).
- **14 documentos técnicos** en `docs/` (arc42 resumen · ADRs · runbooks de release · checklist de auditoría pre-presentación).
- **9 invariantes** (IN-001..IN-009) documentadas y validables vía grep.
- **FSM de 6 estados** explícita en `app/models/document.py` con transiciones validadas.
- **6 artefactos de auditoría** generados por trimestre con hash SHA-256 de integridad.
- **Linter custom** que detecta uso de `float` en math layer (`decimal_only_check.py`).

---

## Compliance y seguridad

- **Aritmética Decimal exclusiva** en math layer (IN-006) · sin riesgo de errores de redondeo acumulados.
- **Validación 4 capas** obligatoria antes de release (formal · realidad · censal · duplicados).
- **Hash SHA-256** en cada manifest · integridad detectable entre releases.
- **NIF del autor seudonimizado** en todos los outputs (IN-004) · nunca aparece en logs públicos o reports compartidos.
- **`/DOCUMENTOS/` y `/data/raw/` en `.gitignore`** estricto · facturas reales y modelos AEAT presentados nunca van al repo.
- **Pre-commit hooks** recomendados: `gitleaks` + `detect-secrets` para prevenir leak accidental de datos fiscales personales.

---

## Donde se aplica este patrón

El patrón "**framework de compliance fiscal con validación multi-capa + aritmética exacta + auditoría pre-release**" es **horizontal a cualquier dominio regulatorio con cero margen de error**:

- Reporting fiscal corporativo (Impuesto Sociedades · IVA · retenciones).
- Compliance financiero (DGS reporting · Banco de España · CNMV).
- Healthcare compliance (HIPAA · facturación sanitaria con códigos CIE).
- Insurance claim processing con auditoría regulatoria.
- E-invoicing en jurisdicciones con sistema obligatorio (Italia SDI · México CFDI · Chile DTE · Brasil NFe · España VeriFactu).
- Government contractor financial reporting.
- ESG reporting cuantitativo (GRI · SASB · TCFD).

Los patterns técnicos resueltos (aritmética Decimal estricta · validación 4 capas · OCR con 5 checks alto nivel · mapping determinista · reconciliación con tolerancia · hash SHA-256 en cada release · FSM explícita · invariantes validables) son **horizontales a cualquier sistema regulado donde un error de 1 céntimo puede disparar consecuencias administrativas**.

---

## Sistema operativo de ingeniería del autor

Este proyecto se desarrolló bajo el sistema operativo de ingeniería propio del autor, replicado cross-proyecto en múltiples proyectos del ecosistema:

- **Automation Engineering Protocol** — stage-gate horizontal de cambio: change-spec → guard layer → release train → rollback playbook.
- **Prompt Engineering Protocol** — diseño modular con catálogo cerrado de patrones (no aplicable aquí, framework determinista sin LLM).
- **Post-Development Verification Protocol** — 4 niveles (estática · integración · canarios E2E con datos sintéticos · observabilidad diaria).
- **Post-Development Gates** — 94 gates pre-merge / pre-deploy / post-deploy con checklist bloqueante.
- **Frozen Zones & Regression Prevention** — CONFIG vs HEALTH, auditoría holística periódica.

Cada decisión estructural se documenta como ADR versionada. Cada cambio significativo pasa por `verify-findings` adversarial. Engineering-playbook propio con **60+ archivos doctrinales cross-proyecto**. La documentación es activo de primera clase.

Protocolo de documentación detallado (arc42 + C4 + ADR + runbooks + postmortems) en [`docs/documentation-protocol.md`](docs/documentation-protocol.md).

*Detalles operativos completos del playbook y del proyecto privado bajo consideraciones de privacidad personal.*

---

## Contacto

**Kamil Slodki** — Valencia, España

- Email: `solutions.satdata@gmail.com`
- LinkedIn: [Kamil Slodki](https://www.linkedin.com/in/kamilslodki/)
- Portfolio: [satdata-portfolio](https://github.com/KamilSlodki/satdata-portfolio)

Disponible para **founding / staff AI roles** (remote · full-time + equity) y **B2B consulting / fractional CTO engagements** (retainer · day-rate · proyectos cerrados). Para consultas técnicas concretas, briefings de proyecto o detalles de compensación, escribir directamente al email comercial.
