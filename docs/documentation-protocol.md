# Protocolo de documentación

Este proyecto se desarrolló bajo el protocolo de documentación propio del autor, replicado cross-proyecto en múltiples proyectos del ecosistema. **La documentación es activo de primera clase, no afterthought.**

> Este showcase es **producto propio del autor con sanitización agresiva**. Datos personales fiscales (NIF · cifras absolutas · clientes/proveedores reales · gestoría · banca · domicilio fiscal · artefactos AEAT con datos reales) NO se publican por consideraciones de privacidad personal. El subset técnico mostrado es la arquitectura, algoritmos y patterns genéricos que demuestran la disciplina de ingeniería sin comprometer información fiscal personal.

---

## 5 capas documentales

### 1. arc42 — Marco de arquitectura del software

Marco estandarizado de 12 secciones. En este repo público se proyectan las secciones genéricas (introducción · estrategia de solución · vista de bloques · vista de runtime · conceptos cross-cutting · requisitos de calidad técnica). Secciones con datos personales (despliegue · riesgos específicos · glosario fiscal personal) viven en repo privado.

### 2. C4 — Modelo visual de arquitectura

Notación Simon Brown: Context · Container · Component · Code. En este showcase se publica solo la estructura conceptual de capas (no diagramas con identificadores reales de proveedores ni cuentas bancarias).

### 3. ADR — Architecture Decision Records

Decisiones estructurales documentadas con plantilla fija (contexto · decisión · alternativas · consecuencias · plan rollout · plan rollback). Las **9 invariantes** (IN-001..IN-009) son los compromisos técnicos versionados en código y validables vía grep.

### 4. Runbooks operativos

Procedimientos de release · auditoría pre-presentación · backup · recovery. NO se publican por contener referencias a credenciales y rutas locales del autor.

### 5. Postmortems formales

Cada error fiscal detectado en producción cierra con postmortem: cronología · causa raíz · acción correctiva (que escala a invariante nueva si aplica) · métrica observable. Los aprendizajes transferibles (no específicos) alimentan el engineering-playbook cross-proyecto del autor.

---

## Disciplina cross-proyecto

- **Engineering-playbook propio** con 60+ archivos doctrinales cross-proyecto.
- **`verify-findings` adversarial**: auditoría con segundo agente independiente. Ratio de ruido objetivo < 15 %.
- **Convenciones**: español · sin emojis decorativos · tono operativo y directo.
- **Aritmética Decimal exclusiva** en math layer (IN-006).
- **Hash SHA-256** en cada release con manifiesto firmado.
- **Validación 4 capas** obligatoria pre-presentación AEAT.

---

*Detalles operativos completos bajo consideraciones de privacidad personal.*
