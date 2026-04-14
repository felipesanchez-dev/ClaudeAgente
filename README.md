# Flujo del Sistema Multiagente BrontoByte

> Mapa conceptual + diagramas de flujo del harness multiagente instalado en `.claude/`.
> Todos los diagramas están en Mermaid (render nativo en VS Code / GitHub).

---

## 1. Mapa conceptual — Componentes del sistema

```mermaid
mindmap
  root((Sistema Multiagente<br>BrontoByte))
    Dirección
      orchestrator
        Clasifica S/M/L
        Delega con Agent tool
        Consolida salida
        Self-check 7 puntos
    Diseño
      optimizer
        Simplificación previa
        Detecta reuso
      planner
        Plan por fases
        Dependencias
        Criterios de éxito
      researcher
        Docs oficiales
        Breaking changes
        Patrones internos
    Ejecución
      implementer
        Cambios mínimos
        Respeta arquitectura
      validator
        Build+test+lint
        Veredicto APTO/NO APTO
      reviewer
        Alineación objetivo
        Mantenibilidad
    Seguridad
      secure-code-auditor
        Caja blanca OWASP
      dependency-and-build-auditor
        Supply chain + CVEs
      secrets-and-leak-detector
        Fingerprints AIza/AKIA
      blackbox-security-tester
        Headers + cookies + TLS
      threat-modeler
        STRIDE + controles
    Infra
      Hooks
        SessionStart
        PreToolUse Write/Edit/Bash
        PostToolUse
        Stop
      Slash Commands
        /plan
        /implement
        /audit
        /sec
        /ship
      MCPs
        filesystem
        github
        playwright
      Scripts
        validate-handoff.py
        install-security-tools.sh
        stop-persist-log.sh
```

---

## 2. Flujo de los 7 pasos obligatorios

```mermaid
flowchart TD
    U[👤 Usuario envía tarea] --> ORCH{orchestrator<br>clasifica}
    ORCH -->|S: trivial| DIRECT[Ejecución directa<br>pasos 2-3-7 colapsados]
    ORCH -->|M/L| FLOW[Flujo completo]

    FLOW --> P1[1️⃣ ANALIZAR<br>contexto, arquitectura, riesgos]
    P1 --> P2[2️⃣ OPTIMIZAR + INVESTIGAR<br>en paralelo]
    P2 --> P3[3️⃣ PLANEAR<br>planner → plan por fases]
    P3 --> P4[4️⃣ IMPLEMENTAR<br>implementer por fase]
    P4 --> P5[5️⃣ SEGURIDAD<br>fan-out paralelo]
    P5 --> P6[6️⃣ VALIDAR<br>builds + tests + lint]
    P6 --> DEC{¿validator<br>APTO?}
    DEC -->|no| P4
    DEC -->|sí| P7[7️⃣ CONSOLIDAR<br>reviewer + orchestrator]
    P7 --> SELF{Self-check<br>7 preguntas}
    SELF -->|falla| P6
    SELF -->|ok| OUT[📤 Salida única<br>al usuario]
    DIRECT --> OUT

    classDef step fill:#e1f5ff,stroke:#0066cc,stroke-width:2px
    classDef sec fill:#fff0e1,stroke:#cc6600,stroke-width:2px
    classDef gate fill:#f0f0f0,stroke:#666,stroke-width:2px
    class P1,P2,P3,P4,P6,P7 step
    class P5 sec
    class ORCH,DEC,SELF gate
```

---

## 3. Fan-out paralelo de seguridad (Paso 5 en detalle)

```mermaid
flowchart LR
    ORCH[orchestrator<br>📨 un solo mensaje<br>con múltiples Agent]
    ORCH -.-> A1[🔍 secure-code-auditor<br>OWASP caja blanca]
    ORCH -.-> A2[📦 dependency-and-build-auditor<br>CVEs + Docker + CI]
    ORCH -.-> A3[🔐 secrets-and-leak-detector<br>Fingerprints AKIA/AIza]
    ORCH -.-> A4[🌐 blackbox-security-tester<br>Headers + TLS + CORS]
    ORCH -.-> A5[🎯 threat-modeler<br>STRIDE + controles]

    A1 --> HO1[YAML handoff]
    A2 --> HO2[YAML handoff]
    A3 --> HO3[YAML handoff]
    A4 --> HO4[YAML handoff]
    A5 --> HO5[YAML handoff]

    HO1 --> VAL[validate-handoff.py<br>auto-normaliza<br>sinónimos]
    HO2 --> VAL
    HO3 --> VAL
    HO4 --> VAL
    HO5 --> VAL

    VAL --> CONS[orchestrator<br>consolida findings<br>renumera G-001..]

    style ORCH fill:#ff9999,stroke:#cc0000,stroke-width:3px
    style VAL fill:#ccffcc,stroke:#006600,stroke-width:2px
    style CONS fill:#ff9999,stroke:#cc0000,stroke-width:2px
```

**Nota:** los 5 agentes corren **concurrentemente** en procesos aislados (duración real medida: ~32s para 3 agentes, no ~95s secuencial).

---

## 4. Ciclo de vida de un handoff

```mermaid
sequenceDiagram
    participant O as orchestrator
    participant A as subagente
    participant V as validate-handoff.py
    participant L as _pending.md
    participant H as hook Stop

    O->>A: Agent(subagent_type, prompt)
    Note over A: lee archivos,<br>genera hallazgos
    A-->>O: Markdown + bloque YAML<br>---agent-handoff---
    O->>V: pipe handoff
    V->>V: normaliza sinónimos<br>(completed→ok)
    alt handoff válido
        V-->>O: exit 0 + JSON
        O->>L: actualiza _pending.md
    else handoff inválido
        V-->>O: exit 1 + errores
        O->>A: re-solicita (1 intento)
    end
    Note over O: Al cerrar turno
    H->>L: lee _pending.md
    H->>H: renombra a<br>logs/<ts>_<session>.md
    Note over H: Rotación 200 logs máx
```

---

## 5. Paralelismo — Matriz de co-ejecución

```mermaid
gantt
    title Ejecución de una tarea M/L en el tiempo
    dateFormat  s
    axisFormat %Ss

    section Bloque A (paralelo)
    researcher          :a1, 0, 30s
    optimizer           :a2, 0, 20s
    threat-modeler*     :a3, 0, 35s

    section Bloque B
    planner             :b1, after a1 a2 a3, 15s

    section Bloque C (secuencial por fases)
    implementer fase 1  :c1, after b1, 20s
    implementer fase 2  :c2, after c1, 20s

    section Bloque D Seguridad (fan-out paralelo)
    secure-code-auditor :d1, after c2, 35s
    dep-and-build       :d2, after c2, 30s
    secrets-leak        :d3, after c2, 25s
    blackbox*           :d4, after c2, 40s

    section Bloque E
    validator           :e1, after d1 d2 d3 d4, 60s
    reviewer            :e2, after e1, 20s

    section Bloque F
    consolidación       :f1, after e2, 5s
```

`*` threat-modeler y blackbox son condicionales (si cambia auth / si hay URL).

---

## 6. Flujo de hooks — ¿cuándo se disparan?

```mermaid
flowchart TD
    START[Inicio sesión Claude Code] --> SS[🪝 SessionStart<br>inyecta contrato multiagente]
    SS --> WORK[Trabajo en curso]

    WORK --> TOOL{Tool a invocar}
    TOOL -->|Write/Edit| PW[🛡️ pre-write-guard.sh<br>bloquea .env, *.pem, *.key]
    PW -->|exit 0| EDIT[ejecuta Write/Edit]
    PW -->|exit 2| STOP1[❌ bloqueado]

    EDIT --> POST[🔍 post-edit-scan.sh<br>escanea archivo editado]
    POST -->|secreto detectado| WARN[⚠️ aviso al agente]
    POST --> NEXT[siguiente acción]
    WARN --> NEXT

    TOOL -->|Bash| PB[🛡️ pre-bash-guard.sh<br>bloquea rm -rf /, curl\|bash]
    PB -->|exit 0| RUN[ejecuta comando]
    PB -->|exit 2| STOP2[❌ bloqueado]

    RUN --> NEXT
    NEXT --> WORK

    WORK -->|fin de turno| SE[🪝 Stop hook<br>stop-persist-log.sh]
    SE --> PROM[promueve _pending.md<br>→ logs/ts_session.md]
    PROM --> END[Cierre]

    classDef hook fill:#ffcccc,stroke:#990000,stroke-width:2px
    classDef block fill:#ff6666,stroke:#990000,stroke-width:3px
    class SS,PW,POST,PB,SE hook
    class STOP1,STOP2 block
```

---

## 7. Slash commands — Atajos disponibles

```mermaid
flowchart LR
    subgraph "Planificación"
        PLAN[/plan tarea/] --> OR1[orchestrator] --> BLOCKA[researcher ∥ optimizer] --> PLAN2[planner]
    end

    subgraph "Implementación"
        IMPL[/implement plan/] --> OR2[orchestrator] --> IMP[implementer por fase] --> FAN[fan-out seguridad] --> VAL2[validator] --> REV[reviewer]
    end

    subgraph "Auditoría"
        AUDIT[/audit scope/] --> OR3[orchestrator] --> FAN2[fan-out completo<br>5 auditores en paralelo]
    end

    subgraph "Caja blanca focal"
        SEC[/sec ruta/] --> OR4[orchestrator] --> SCA[secure-code-auditor<br>+ secrets + threat si aplica]
    end

    subgraph "Pre-deploy"
        SHIP[/ship entorno/] --> OR5[orchestrator] --> ALL[validator + deps + secrets + blackbox + reviewer] --> VER{GO / NO-GO}
    end
```

---

## 8. Arquitectura de archivos

```
.claude/
├── CLAUDE.md                    ← contrato operativo (7 pasos, reglas)
├── AGENT_HANDOFF.md             ← schema YAML de hand-off + ⚠️ formato literal
├── flujo.md                     ← este archivo
├── settings.local.json          ← permisos + hooks + env
├── .gitignore
│
├── agents/                      ← 12 subagentes especializados
│   ├── orchestrator.md          [opus]
│   ├── planner.md               [opus]
│   ├── reviewer.md              [opus]
│   ├── secure-code-auditor.md   [opus]
│   ├── threat-modeler.md        [opus]
│   ├── optimizer.md             [sonnet]
│   ├── researcher.md            [sonnet]
│   ├── implementer.md           [sonnet]
│   ├── validator.md             [sonnet]
│   ├── blackbox-security-tester.md   [sonnet]
│   ├── dependency-and-build-auditor.md [sonnet]
│   └── secrets-and-leak-detector.md  [sonnet]
│
├── commands/                    ← slash commands
│   ├── plan.md
│   ├── implement.md
│   ├── audit.md
│   ├── sec.md
│   └── ship.md
│
├── scripts/                     ← hooks + validación + instalador
│   ├── pre-write-guard.sh       (hook PreToolUse Write/Edit)
│   ├── pre-bash-guard.sh        (hook PreToolUse Bash)
│   ├── post-edit-scan.sh        (hook PostToolUse)
│   ├── stop-persist-log.sh      (hook Stop)
│   ├── validate-handoff.py      (validador YAML tolerante)
│   └── install-security-tools.sh
│
└── logs/                        ← trazabilidad automática
    ├── _pending.md              (buffer del turno actual, gitignored)
    └── <timestamp>_<session>.md (archivados por el hook Stop)
```

Raíz del proyecto:

```
BrontoByte/
├── .gitignore                   ← ignora .env, dist, node_modules, logs
├── .mcp.json                    ← MCPs compartidos (filesystem, github, playwright, fetch)
├── frontend-brontobyte/
├── backend-brontobyte/
└── .claude/                     (arriba)
```

Memoria global de usuario:

```
~/.claude/projects/c--Users-Felipe-Documents-BrontoByte/memory/
├── MEMORY.md                    ← índice
├── project_brontobyte.md        (proyecto)
├── project_agent_system.md      (sistema multiagente)
└── feedback_work_style.md       (preferencias del usuario)
```

---

## 9. Modelo de decisión del orchestrator

```mermaid
flowchart TD
    T[Tarea entrante] --> C{Clasificar}
    C -->|1-2 pasos triviales| S[Tipo S]
    C -->|3-8 pasos, ≤3 módulos| M[Tipo M]
    C -->|multi-módulo / auditoría / feature| L[Tipo L]

    S --> SE[orchestrator ejecuta directo]
    SE --> SEND[entrega]

    M --> MA[Analizar] --> MB[optimizer + researcher paralelo]
    MB --> MC[planner] --> MD[implementer] --> ME[fan-out seguridad]
    ME --> MF[validator] --> MG[reviewer] --> MH[consolidar + log]
    MH --> SEND

    L --> LA[Analizar profundo] --> LB[optimizer + researcher + threat-modeler]
    LB --> LC[planner por fases] --> LD[implementer fase a fase]
    LD --> LE[fan-out completo 5 agentes]
    LE --> LF[validator] --> LG[reviewer] --> LH[self-check 7 pts]
    LH --> LI[consolidar + log completo]
    LI --> SEND

    classDef simple fill:#ccffcc
    classDef medium fill:#ffffcc
    classDef large fill:#ffcccc
    class S,SE simple
    class M,MA,MB,MC,MD,ME,MF,MG,MH medium
    class L,LA,LB,LC,LD,LE,LF,LG,LH,LI large
```

---

## 10. Ciclo de retroalimentación y autocorrección

```mermaid
stateDiagram-v2
    [*] --> Recibido
    Recibido --> Analizando: orchestrator lee
    Analizando --> Planeando: contexto suficiente
    Analizando --> Preguntando: ambigüedad crítica
    Preguntando --> Analizando: respuesta usuario
    Planeando --> Implementando: plan aprobado
    Implementando --> Seguridad: fase completa
    Seguridad --> Validando: handoffs válidos
    Seguridad --> Implementando: critical/high abierto
    Validando --> Revisando: APTO
    Validando --> Implementando: NO APTO
    Revisando --> Consolidando: alineado
    Revisando --> Implementando: desviado
    Consolidando --> SelfCheck
    SelfCheck --> Cerrado: 7/7 OK
    SelfCheck --> Seguridad: gap detectado
    Cerrado --> [*]

    note right of SelfCheck
        1. ¿Plan cubrió todo?
        2. ¿Fan-out seguridad OK?
        3. ¿Handoffs válidos?
        4. ¿Critical/high cerrados?
        5. ¿validator APTO?
        6. ¿reviewer APROBADO?
        7. ¿_pending.md escrito?
    end note
```

---

## 11. Ejemplo real — Tarea M paso a paso

**Usuario:** *"Agrega endpoint `/api/users/me` que devuelva el perfil autenticado."*

```mermaid
journey
    title Ejecución de tarea M — endpoint /api/users/me
    section Análisis
      Orchestrator clasifica M: 5: orchestrator
      Lee backend existente: 4: orchestrator
    section Paralelo inicial
      researcher docs FastAPI deps: 5: researcher
      optimizer busca endpoints similares: 5: optimizer
    section Plan
      planner produce 3 fases: 5: planner
    section Implementación
      Fase 1 router + schema: 5: implementer
      Fase 2 hook frontend: 5: implementer
    section Seguridad (paralelo)
      secure-code-auditor: 5: secure-code-auditor
      secrets-and-leak-detector: 5: secrets-and-leak-detector
      threat-modeler checks authz: 4: threat-modeler
    section Cierre
      validator build+test: 5: validator
      reviewer alineación: 5: reviewer
      orchestrator consolida: 5: orchestrator
      log persistido: 5: hook-Stop
```

---

## 12. Glosario rápido

| Término | Definición |
|---|---|
| **Orchestrator** | Agente director único punto de entrada. Nunca implementa en M/L, siempre delega. |
| **Hand-off** | Bloque YAML entre `---agent-handoff---` y `---end-handoff---` que todo subagente emite al terminar. |
| **Fan-out** | Lanzamiento paralelo de múltiples subagentes en un solo mensaje. |
| **S/M/L** | Clasificación de tamaño: Small (trivial), Medium (3-8 pasos), Large (auditoría/feature). |
| **STRIDE** | Marco de amenazas: Spoofing, Tampering, Repudiation, Information disclosure, DoS, Elevation of privilege. |
| **Self-check 7 puntos** | Preguntas internas del orchestrator antes de cerrar M/L. |
| **`_pending.md`** | Buffer del turno actual; el hook Stop lo archiva con timestamp. |
| **BFF** | Backend-for-Frontend; patrón usado para ocultar IA/servicios externos al cliente. |
| **Harness** | Infraestructura ejecutable (no solo doc) que fuerza el flujo: hooks + validador + logs. |

---

## 13. Resumen ejecutivo del flujo

1. **Usuario** escribe tarea (o usa `/audit`, `/plan`, etc.).
2. **Orchestrator** clasifica S/M/L y ejecuta el flujo de 7 pasos.
3. **Optimizer + researcher** en paralelo aportan contexto.
4. **Planner** produce plan verificable por fases.
5. **Implementer** ejecuta fase a fase.
6. **Fan-out seguridad** (hasta 5 agentes paralelos) audita sin bloquearse entre sí.
7. **validate-handoff.py** normaliza y valida cada YAML de subagente.
8. **Validator** certifica build/tests/lint.
9. **Reviewer** confirma alineación con el objetivo.
10. **Orchestrator** consolida, responde `Self-check 7 puntos`, emite salida final.
11. **Hook Stop** archiva `_pending.md` → `logs/<timestamp>_<session>.md`.
12. **Hooks PreTool** bloquean en vivo escrituras a `.env` y comandos destructivos.

**Resultado:** equipo técnico coordinado, trazable, defensivo y reproducible.
