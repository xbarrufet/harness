## Fase 1: Cimientos del Sistema, Estado Global y Capa de Persistencia Abstraída

### Rationale

Un sistema multi-agente determinista no se apoya en llamadas conversacionales desordenadas entre LLMs, sino en una **Arquitectura Guiada por Estados** (*State-Driven Architecture*). Definir un contrato de datos estricto desde el primer día previene fallos de integración, evita la pérdida de contexto y permite abstraer los componentes de almacenamiento. De este modo, la persistencia de las *Features* y la lectura de la documentación (Wiki) pueden cambiar de tecnología (de archivos Markdown a Jira o PostgreSQL) sin alterar la lógica interna de los agentes.

### Tecnologías y Librerías

* **`langgraph` (>= 0.2.0):** Para la orquestación del grafo de estados y la gestión del ciclo de vida.
* **`pydantic` (v2):** Validaciones de esquemas de datos estrictos en Python.
* **`typing` / `typing_extensions`:** Uso de `TypedDict`, `Annotated` y reducers de estado (`operator.add`).
* **`python-dotenv`:** Gestión segura de claves de API y variables de entorno.

### Pasos y Actividades

#### Paso 1.1: Configuración de la Estructura del Proyecto

Crear un repositorio modular que separe los modelos de estado, las herramientas (*tools*), los nodos (*nodes*) y la topología del grafo.

```text
multiagent_system/
├── backlog/                 # Almacenamiento local inicial de Features (.md)
├── docs/                    # Wiki local del proyecto
│   └── wiki_index.md
├── audit/                   # Informes inmutables de auditoría (RTM)
├── src/
│   ├── state.py             # Esquemas Pydantic y AgentState
│   ├── tools/               # Wiki, Sandbox y Git tools
│   ├── nodes/               # Lógica individual de cada agente
│   ├── graph.py             # Construcción del StateGraph
│   └── main.py              # CLI / Punto de entrada interactivo

```

#### Paso 1.2: Definición del Esquema Estricto de Datos (`src/state.py`)

Diseñar los modelos para la trazabilidad fina desde la Feature hasta el Criterio de Aceptación (`AC-ID`).

* *Actividad:* Implementar la jerarquía de Pydantic: `AcceptanceCriterion` -> `Feature` -> `Task` -> `TestCaseResult`.
* *Actividad:* Definir el `AgentState` con el reducer `operator.add` para logs append-only.

```python
from typing import TypedDict, List, Optional, Annotated, Literal
from pydantic import BaseModel, Field
import operator

class AcceptanceCriterion(BaseModel):
    id: str = Field(description="Identificador único del criterio. Ej: AC-FEAT-01-01")
    description: str = Field(description="Enunciado del criterio a probar")
    status: Literal["PENDING", "PASSED", "FAILED"] = "PENDING"

class Feature(BaseModel):
    id: str = Field(description="Ej: FEAT-01")
    title: str
    user_story: str
    acceptance_criteria: List[AcceptanceCriterion]
    status: Literal["BACKLOG", "IN_PROGRESS", "TESTING", "DONE"] = "BACKLOG"

class Task(BaseModel):
    id: str = Field(description="Ej: TASK-101")
    feature_id: str = Field(description="Vinculación directa con la Feature")
    assigned_agent: Literal["dev_agent", "qa_spec_agent", "qa_runner_agent"]
    description: str
    status: Literal["PENDING", "IN_DEV", "IN_TESTING", "REJECTED_BY_QA", "COMPLETED", "FAILED"] = "PENDING"
    e2e_specs: Optional[str] = None
    code_result: Optional[str] = None
    test_reports: Optional[str] = None
    dev_qa_iterations: int = 0
    max_iterations: int = 3

class AgentState(TypedDict):
    requirements: str
    raw_specs: Optional[str]
    features: List[Feature]
    plan: Optional[dict]
    current_task_id: Optional[str]
    logs: Annotated[list, operator.add]
    wiki_index_path: str
    traceability_matrix: Annotated[list, operator.add]
    human_approval_required: bool

```

#### Paso 1.3: Desarrollo de Herramientas de Contexto y Persistencia (`src/tools/`)

* *Actividad:* Escribir la herramienta `read_wiki_index` y `read_wiki_document` para permitir a los agentes consultar la documentación en `docs/` solo bajo demanda.
* *Actividad:* Crear la capa de abstracción `FeatureStorage` que guarda y lee las *Features* en archivos `.md` dentro de `backlog/`.

---

## Fase 2: Toma de Requisitos (Spec Creator - GrillMe) y Validador Type-Safe

### Rationale

El principal punto de fallo en el desarrollo de software es la ambigüedad en los requisitos (*Garbage In, Garbage Out*). Aplicar el **Patrón GrillMe** en la fase inicial fuerza al sistema a "entrevistar" al usuario para destilar casos límite y requisitos no funcionales antes de escribir una sola línea de código. Además, posicionar un **Validador Type-Safe** como puerta de enlace inicial (*circuit breaker*) evita malgastar *tokens* y tiempo de procesamiento si la entrada no cumple las condiciones mínimas.

### Tecnologías y Librerías

* **`langchain-openai` / `langchain-anthropic`:** Modelos LLM avanzados con soporte nativo de llamadas a funciones y salida estructurada.
* **`pydantic.field_validator`:** Reglas de validación programáticas en Python.
* **LangGraph `interrupt`:** Para pausar el estado y esperar respuestas humanas durante la entrevista GrillMe.

### Pasos y Actividades

#### Paso 2.1: Implementación del Agente Creador de Especificaciones (`SpecCreatorAgent`)

* *Actividad:* Construir el prompt del sistema bajo el patrón GrillMe. El agente analiza los requisitos iniciales y formula preguntas de clarificación sobre errores, permisos o casos límite.
* *Actividad:* Integrar el bucle de interacción para refinar el texto de entrada hasta obtener una especificación detallada (`raw_specs`).

#### Paso 2.2: Agente de Persistencia de Features (`FeatureStorageAgent`)

* *Actividad:* Procesar `raw_specs` para extraer las User Stories individuales.
* *Actividad:* Asignar automáticamente IDs estructurados (`FEAT-01`, `FEAT-02`) y desglosar los Criterios de Aceptación asignándoles su propio ID (`AC-FEAT-01-01`, `AC-FEAT-01-02`).
* *Actividad:* Ejecutar la persistencia abstraída escribiendo el archivo en `backlog/FEAT-01.md`.

#### Paso 2.3: Nodo Evaluador Type-Safe (`SpecValidatorNode`)

* *Actividad:* Crear el modelo Pydantic `SpecValidationResult` con validadores estritos.
* *Actividad:* Invocar al LLM mediante `.with_structured_output(SpecValidationResult)` para evaluar que no existan contradicciones y que cada Feature contenga al menos un Criterio de Aceptación verificable.

```python
class SpecValidationResult(BaseModel):
    is_valid: bool = Field(description="True si la especificación está completa para desarrollo")
    has_acceptance_criteria: bool = Field(description="True si todos los ACs son explícitos")
    missing_information: List[str] = Field(default_factory=list)
    feedback_prompt: Optional[str] = Field(default=None)

def spec_validator_node(state: AgentState) -> dict:
    prompt = ChatPromptTemplate.from_messages([
        ("system", "Evalúa de forma estricta si las Features contienen toda la información técnica necesaria."),
        ("human", "Features: {features}")
    ])
    chain = prompt | llm.with_structured_output(SpecValidationResult)
    res: SpecValidationResult = chain.invoke({"features": state["features"]})
    
    if not res.is_valid:
        return {
            "human_approval_required": True,
            "logs": [f"[SpecValidator]: RECHAZADO. Informar al usuario: {res.missing_information}"]
        }
    return {"human_approval_required": False, "logs": ["[SpecValidator]: APROBADO. Requisitos listos."]}

```

---

## Fase 3: Arquitectura, Planificación e Intervención Humana (Human-in-the-Loop)

### Rationale

Garantizar la alineación estratégica del software requiere supervisión ejecutiva antes de ejecutar tareas intensivas. Un agente Arquitecto/Planificador traduce las *Features* en un plan de trabajo acoplado. Sin embargo, para evitar que el agente tome decisiones de diseño erróneas o costosas, introducimos un **Punto de Interrupción (Breakpoint)** en LangGraph. Esto permite a un ingeniero humano pausar la ejecución, revisar la arquitectura en memoria y modificar o aprobar las tareas antes de liberarlas a los desarrolladores.

### Tecnologías y Librerías

* **`langgraph.checkpoint.memory.MemorySaver`:** Persistencia de estado en memoria (o Redis/PostgreSQL en producción) para permitir la pausa e inspección del grafo.
* **LangGraph Breakpoints (`interrupt_before`):** Interrupción nativa del flujo antes de la ejecución del enrutador dinámico.

### Pasos y Actividades

#### Paso 3.1: Nodo Planificador (`PlannerNode`)

* *Actividad:* Diseñar el prompt del Arquitecto Software para convertir las *Features* en una lista de tareas (`Task`).
* *Actividad:* Garantizar que cada `Task` mantenga una referencia explícita al `feature_id` correspondiente (ej: `TASK-101` para `FEAT-01`), asignando inicialmente la fase a `qa_spec_agent`.

#### Paso 3.2: Configuración del Checkpointer e Interrupción

* *Actividad:* Instanciar `MemorySaver()` e integrar la opción de interrupción en la compilación del grafo.
* *Actividad:* Crear la interfaz por consola o API para inspeccionar el estado actual (`app.get_state(config)`), mostrar el plan sugerido por el LLM y permitir que el usuario teclee "Aprobar" o ingrese feedback de corrección.

```python
# Compilación del grafo con interrupción antes del enrutador principal
app = builder.compile(
    checkpointer=memory,
    interrupt_before=["router"]
)

# En la CLI de control:
snapshot = app.get_state(config)
if snapshot.next and "router" in snapshot.next:
    print("--- PLAN GENERADO PARA REVISIÓN ---")
    print(snapshot.values["plan"])
    user_input = input("¿Aprobar plan? (s/n/feedback): ")
    if user_input.lower() != 's':
        app.update_state(config, {"human_feedback": user_input}, as_node="planner")

```

---

## Fase 4: Bucle Colaborativo Dev/QA y Quality Gate Determinista (Fail-Fast Sandbox)

### Rationale

Evaluar la calidad del código mediante la revisión conversacional de otro LLM genera "alucinaciones cruzadas" e imperfecciones sintácticas. En esta fase, la validación se vuelve **100% determinista**. Aplicamos el principio **Fail-Fast** en un Sandbox aislado ordenado por capas de coste y tiempo:

1. *Análisis Estático (Linter/SonarQube/Mypy):* Ejecución instantánea y económica.
2. *Tests Unitarios (`pytest`):* Ejecución rápida de la lógica del componente.
3. *Tests E2E (`pytest` / Playwright):* Pruebas complejas de integración.

Si una capa falla, el bucle se detiene inmediatamente y devuelve el reporte exacto al `DevAgent` sin gastar recursos en las capas superiores.

```text
[ DevAgent ] ──► Genera Código y Tests (Etiquetados con AC-ID)
                       │
                       ▼
┌─────────────────────────────────────────────────────────────┐
│                   QA RUNNER (FAIL-FAST)                     │
│                                                             │
│  [1. Linter / Sonar Rules] ── (Falla) ──┐                   │
│         │ Pasa                          │                   │
│         ▼                               │                   │
│  [2. Unit Tests (Pytest)]  ── (Falla) ──┼─► [REJECTED_BY_QA]│
│         │ Pasa                          │   (Dev reintenta) │
│         ▼                               │                   │
│  [3. E2E Tests (Integration)]─(Falla)──┘                   │
│         │ Pasa                                              │
└─────────┼───────────────────────────────────────────────────┘
          ▼
   [ Task COMPLETED ]

```

### Tecnologías y Librerías

* **`subprocess` / Docker:** Entorno de ejecución Sandbox para aislamiento de código.
* **`ruff` / `flake8` / `mypy` / `bandit`:** Análisis estático de tipo, estilo, complejidad y seguridad (SonarQube Standards).
* **`pytest`:** Framework para la ejecución de pruebas unitarias e integración.

### Pasos y Actividades

#### Paso 4.1: QA Spec Agent Node (`qa_spec_agent_node`)

* *Actividad:* Leer la `Task` y sus Criterios de Aceptación (`AC-FEAT-01-01`, etc.).
* *Actividad:* Diseñar el plan de pruebas E2E y declarar los nombres esperados de los casos de prueba.

#### Paso 4.2: Dev Agent Node con Quality Profile (`dev_agent_node`)

* *Actividad:* Inyectar en el prompt el perfil de calidad estricto (SonarQube Standards: typings obligatorios, complejidad ciclomática < 5, ausencia de `eval()`).
* *Actividad:* Escribir el código de producción, las pruebas unitarias y la implementación E2E utilizando **marcadores explícitos de pytest** para enlazar cada prueba con su `AC-ID`.

```python
# Ejemplo de código generado por el DevAgent:
import pytest

@pytest.mark.feature("FEAT-01")
@pytest.mark.ac("AC-FEAT-01-01")
def test_validar_descuento_aplicado():
    assert calcular_descuento(100, 10) == 90

```

#### Paso 4.3: QA Runner Node con Pipeline Fail-Fast (`qa_runner_agent_node`)

* *Actividad:* Guardar el código generado en un archivo temporal en el Sandbox (`/tmp/task_TASK-101.py`).
* *Actividad:* Ejecutar `run_static_analysis` (Linter, Mypy, Bandit). Si hay violaciones, marcar `status = REJECTED_BY_QA` e incluir los logs del Linter.
* *Actividad:* Si la fase estática aprueba, ejecutar `pytest`. Si hay fallos de `assert`, marcar `REJECTED_BY_QA` con los rastreos de error (*stacktraces*).
* *Actividad:* Controlar el contador `dev_qa_iterations`. Si supera el límite ($N \ge 3$), cambiar el estado a `FAILED` y escalar la tarea al `PlannerNode`.

---

## Fase 5: Agente de Auditoría y Matriz de Trazabilidad de Requisitos (RTM)

### Rationale

Para cumplir con marcos normativos, ISO/IEC 25010 o auditorías de clientes corporativos, es indispensable demostrar que **el 100% de los Criterios de Aceptación han sido probados con éxito**. Un agente de auditoría tradicional no debe "opinar" sobre si el software está listo; en su lugar, analiza de forma determinista los reportes estructurados emitidos por el framework de pruebas (`pytest-json-report`) durante la ejecución en el Sandbox, construyendo una **Matriz de Trazabilidad de Requisitos (RTM)** inmutable.

### Tecnologías y Librerías

* **`pytest-json-report`:** Plugin de Pytest para volcar el resultado completo de la suite en formato JSON estructurado.
* **`json` / `datetime`:** Procesamiento determinista de logs y estancado de marcas de tiempo.
* **Markdown Table Generator:** Creación automatizada de artefactos auditables.

### Pasos y Actividades

#### Paso 5.1: Extracción de Metadatos de Ejecución

* *Actividad:* Configurar la ejecución de `pytest` en el Sandbox para generar el archivo `/tmp/report.json` con la bandera `--json-report`.
* *Actividad:* Crear un parser que lea el JSON y extraiga el nombre de la prueba, los marcadores `@pytest.mark.ac`, el estado final (`passed`/`failed`), la duración y el timestamp de ejecución.

#### Paso 5.2: Agente de Auditoría (`AuditAgentNode`)

* *Actividad:* Iterar sobre todas las `Features` del estado y cruzar sus `AcceptanceCriteria` contra los resultados del JSON parser.
* *Actividad:* Generar el documento inmutable `audit/Requirements_Traceability_Matrix.md`.
* *Actividad:* Si algún criterio carece de test o está en estado `FAILED`, marcar el informe global como `REJECTED_FOR_PRODUCTION`.

```python
def audit_agent_node(state: AgentState) -> dict:
    features = state["features"]
    results = state.get("traceability_matrix", [])
    
    rtm_md = "# Matriz de Trazabilidad de Requisitos (RTM)\n\n"
    rtm_md += "| Feature ID | Criterio de Aceptación (AC-ID) | Test Case | Estado | Timestamp |\n"
    rtm_md += "| :--- | :--- | :--- | :---: | :--- |\n"
    
    for feat in features:
        for ac in feat.acceptance_criteria:
            matched = next((r for r in results if r["ac_id"] == ac.id), None)
            if matched and matched["status"] == "PASSED":
                rtm_md += f"| `{feat.id}` | `{ac.id}`: {ac.description} | `{matched['test_id']}` | ✅ PASSED | {matched['timestamp']} |\n"
            else:
                rtm_md += f"| `{feat.id}` | `{ac.id}`: {ac.description} | N/A | ❌ FAILED | N/A |\n"
                
    with open("audit/RTM.md", "w") as f:
        f.write(rtm_md)
        
    return {"logs": ["[AuditAgent]: Matriz RTM escrita en 'audit/RTM.md'."]}

```

---

## Fase 6: Publicación en Git y Pipeline CI/CD Híbrido (Estrategia Zero-Trust)

### Rationale

Bajo el principio de **Confianza Cero (*Zero-Trust*)**, no debemos confiar ciegamente en que el Sandbox local simulado por el agente sea idéntico al entorno de producción final. El Sandbox local realiza las validaciones rápidas para ahorrar costes de CI. Una vez aprobadas, el **Git Agent** abre formalmente una Pull Request en el repositorio remoto. Esto desencadena el pipeline oficial en el servidor de CI/CD (GitHub Actions / GitLab CI). Si la CI remota detecta algún fallo, envía un **Webhook** que despierta al `DevAgent` en la rama correspondiente para aplicar un parche automático.

```text
[ Agente Local ] ──► (Quality Gate Sandbox OK) ──► [ GitAgent ] ──► Abre Pull Request
                                                                           │
                                                                           ▼
                                                             [ Pipeline GitHub Actions ]
                                                               (SonarQube Server / Full Matrix)
                                                                           │
                                                                 ¿Resultado de la CI?
                                                                 ┌─────────┴─────────┐
                                                         FALLO   │                   │ ÉXITO
                                                                 ▼                   ▼
                                                        [ Webhook Listener ]   [ Ready for Human ]
                                                                 │              [ Merge to Main  ]
                                                                 ▼
                                                        [ DevAgent Corrige ]

```

### Tecnologías y Librerías

* **`GitPython` / GitHub CLI (`gh`):** Automatización de operaciones de control de versiones.
* **GitHub Actions / GitLab CI YAML:** Definición del pipeline de integración continua oficial.
* **`fastapi` / `uvicorn`:** Servidor web ligero para escuchar Webhooks de eventos de fallo de CI/CD.

### Pasos y Actividades

#### Paso 6.1: Agente de Git (`GitAgentNode`)

* *Actividad:* Crear la rama remota nombrada con el ID de la Feature (`feat/FEAT-01-login-jwt`).
* *Actividad:* Realizar el commit asociando el ID en el mensaje (`feat(FEAT-01): implementada lógica de auth según AC-FEAT-01-01`).
* *Actividad:* Abrir la Pull Request automáticamente invocando la CLI oficial (`gh pr create --title "[FEAT-01] Implementación Auth API" --body "..."`).

#### Paso 6.2: Definición del Workflow Oficial de CI (`.github/workflows/ci.yml`)

* *Actividad:* Configurar la ejecución oficial de SonarScanner CLI conectado al servidor oficial de SonarQube/SonarCloud.
* *Actividad:* Ejecutar la suite completa de pruebas en contenedores aislados.

```yaml
name: Production Quality Gate
on:
  pull_request:
    branches: [ main ]

jobs:
  verify-code:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.11'
      - name: Install dependencies
        run: |
          pip install ruff mypy pytest pytest-json-report
      - name: Run Quality Gate Estático
        run: |
          ruff check .
          mypy .
      - name: Run Test Suite
        run: |
          pytest --json-report

```

#### Paso 6.3: Cierre del Bucle mediante Webhook Listener

* *Actividad:* Crear un servicio API en `FastAPI` que escuche el evento `workflow_run.completed` de GitHub/GitLab.
* *Actividad:* Si la propiedad `conclusion` del evento es `failure`, extraer los logs de fallo del API de Git, cargar el hilo correspondiente en `LangGraph` y reactivar al `DevAgent` en la rama activa para que aplique la corrección y realice un nuevo `git push`.

---

## Matriz Resumen de Fases y Entregables

| Fase | Componente Principal | Tecnología Clave | Entregable/Artefacto |
| --- | --- | --- | --- |
| **Fase 1** | Cimientos y Estado | `langgraph`, `pydantic` v2 | `AgentState`, Wiki local y `FeatureStorage` abstraído |
| **Fase 2** | Spec Creator y Type-Safe Validator | Patrón GrillMe, Structured Output | Features persistidas en `backlog/FEAT-XX.md` con `AC-ID` |
| **Fase 3** | Planner y Human-in-the-Loop | LangGraph `MemorySaver`, Breakpoints | Plan de tareas revisado y aprobado por el supervisor |
| **Fase 4** | Bucle Dev/QA Fail-Fast Sandbox | `ruff`, `mypy`, `pytest` en Sandbox | Código de producción y tests etiquetados aprobados localmente |
| **Fase 5** | Audit Agent y RTM | `pytest-json-report`, Parser JSON | Matriz de Trazabilidad inmutable `audit/RTM.md` |
| **Fase 6** | Git Agent y CI/CD Zero-Trust | `gh` CLI, GitHub Actions, FastAPI | Pull Request creada y verificada en el CI remoto |