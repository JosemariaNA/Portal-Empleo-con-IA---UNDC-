# Documento de Diseño — UNDC-Empleo

## Índice

1. [Overview](#overview)
2. [Architecture](#architecture)
3. [Data Models](#data-models)
4. [Components and Interfaces](#components-and-interfaces)
5. [Error Handling](#error-handling)
6. [Testing Strategy](#testing-strategy)
7. [Correctness Properties](#correctness-properties)

---

## Overview

UNDC-Empleo es un **monolito modular** sin microservicios, colas de mensajes ni frameworks de agentes. Todos los módulos residen en el mismo repositorio, se comunican mediante importaciones directas de Python o llamadas HTTP locales, y comparten una única base de datos PostgreSQL 16.

El sistema expone cuatro portales web (Estudiante, Egresado, Empresa, OSE) sobre una API FastAPI que orquesta un pipeline determinista de afinidad laboral. El pipeline aplica un filtro duro, calcula un puntaje `Decimal` y genera una explicación textual (LLM con fallback a plantilla). Un Servidor MCP proporciona acceso estructurado a la base de datos y un Módulo RAG local genera rutas de cierre de brechas a partir de sílabos universitarios.

### Principios Rectores

- **Sin LLM en etapas críticas**: el Filtro Duro y el Puntaje usan solo aritmética `Decimal`; el LLM solo interviene en la redacción de explicaciones.
- **Privacidad por diseño**: ninguna tabla almacena atributos protegidos (edad, género, foto, distrito, colegio, estado civil); el LLM nunca recibe nombre ni código universitario.
- **Determinismo verificable**: el mismo par Perfil-Vacante siempre produce la misma Huella SHA-256 y el mismo Puntaje, desde cualquier entorno.
- **Consentimiento como prerequisito**: ninguna operación comparte datos de un egresado sin consentimiento `compartir_con_empresas` activo.

### Variables de Entorno Requeridas

```
DATABASE_URL=postgresql+psycopg2://...
JWT_SECRET=<clave aleatoria larga>
WEBHOOK_SECRET=<clave aleatoria larga>
GEMINI_API_KEY=<opcional; si no está, se usa plantilla>
GROQ_API_KEY=<opcional; respaldo del LLM>
CHROMA_DB_PATH=./data/chroma
CORPUS_PATH=./packages/rag/corpus
```

---

## Architecture

### Diagrama General

```
┌─────────────────────────────────────────────────────────────────────┐
│                       NAVEGADOR / CLIENTE                           │
│   Portal Estudiante · Portal Egresado · Portal Empresa · Portal OSE │
│            (Next.js 15 + React 19 + Tailwind v4 + Recharts)         │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ HTTPS / REST + JSON
┌──────────────────────────────▼──────────────────────────────────────┐
│                          API FastAPI                                 │
│              apps/api · Python 3.11+ · Pydantic v2                  │
│  ┌──────────┐ ┌───────────┐ ┌──────────────┐ ┌───────────────────┐  │
│  │ Auth JWT │ │ Afinidad  │ │  Consentim.  │ │  Vacantes/Ranking │  │
│  │ Router   │ │ Router    │ │  Router      │ │  Router           │  │
│  └──────────┘ └──────┬────┘ └──────────────┘ └───────────────────┘  │
│                      │                                               │
│  ┌───────────────────▼──────────────────────────────────────────┐   │
│  │              Pipeline Determinista                            │   │
│  │   packages/agent/pipeline/                                    │   │
│  │   Filtro Duro → Puntaje (Decimal) → Explicación              │   │
│  │                              ↕ Validador ↕ Plantilla         │   │
│  └───────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌──────────────────┐    ┌───────────────────┐                       │
│  │  Servidor MCP    │    │   Módulo RAG       │                       │
│  │  packages/mcp    │    │   packages/rag     │                       │
│  │  Solo lectura    │    │   ChromaDB local   │                       │
│  │  +1 escritura    │    │   sentence-transf. │                       │
│  └──────────────────┘    └───────────────────┘                       │
└──────────────────────────────┬──────────────────────────────────────┘
                               │ SQLAlchemy 2 / Alembic
┌──────────────────────────────▼──────────────────────────────────────┐
│               PostgreSQL 16 con pgvector · Docker                   │
└─────────────────────────────────────────────────────────────────────┘
```

### Tres Entornos Virtuales

Las dependencias son incompatibles entre módulos, por lo que se usan tres entornos virtuales separados:

| Entorno virtual | Módulos que lo usan | Dependencias clave |
|---|---|---|
| `.venv` | API FastAPI, generador de datos | FastAPI, SQLAlchemy 2, Pydantic v2, bcrypt, PyJWT, httpx |
| `.venv-mcp` | Servidor MCP, agente/pipeline | SDK MCP (Python), psycopg2, estándar de Python |
| `.venv-rag` | Módulo RAG, indexador | sentence-transformers, chromadb, torch (CPU) |

> El pipeline (`packages/agent/pipeline/`) usa **solo la biblioteca estándar de Python** para poder ser importado por los tres entornos.

Para importarlo desde cualquier entorno:

```python
import sys
sys.path.insert(0, str(Path(__file__).resolve().parents[2] / "packages"))
from agent.pipeline import ejecutar_pipeline
```

### Flujo de una Consulta de Afinidad (secuencia)

```
Estudiante           API FastAPI         Servidor MCP       Módulo RAG          LLM
    │                    │                    │                   │               │
    │─── GET /afinidad──►│                    │                   │               │
    │                    │─── get_perfil() ──►│                   │               │
    │                    │◄── perfil JSON ────│                   │               │
    │                    │─── get_vacante()──►│                   │               │
    │                    │◄── vacante JSON ───│                   │               │
    │                    │                    │                   │               │
    │                    │── filtro_duro() ───►(puro Python, sin MCP ni RAG)      │
    │                    │── calcular_puntaje()►(Decimal, sin LLM)                │
    │                    │── consultar_rag() ─────────────────►  │               │
    │                    │◄── fragmentos + similitud ────────────│               │
    │                    │── generar_ruta_cierre()               │               │
    │                    │                    │                   │               │
    │                    │─── redactar_explicacion() ─────────────────────────►  │
    │                    │◄── texto explicación ──────────────────────────────── │
    │                    │── validar_explicacion()               │               │
    │                    │  (si falla → plantilla_determinista)  │               │
    │                    │── registrar en match_registro         │               │
    │◄── JSON afinidad ──│                    │                   │               │
```

### Reglas de Integración

- El **LLM nunca recibe** nombre ni código universitario del estudiante.
- El **Validador** se ejecuta **siempre** antes de entregar texto al usuario, aunque el LLM responda correctamente.
- El **Módulo RAG** nunca llama a servicios externos de embeddings; opera exclusivamente en local.
- La **OSE** accede solo a cifras agregadas; el router de indicadores rechaza con 422 cualquier filtro de identificación individual.

### Estructura de Directorios

```
Proyecto-Empleo/
├── apps/
│   ├── api/                        # FastAPI principal
│   │   ├── main.py
│   │   ├── config.py               # variables de entorno (pydantic-settings)
│   │   ├── deps.py                 # dependencias FastAPI (get_db, get_current_user)
│   │   ├── routers/
│   │   │   ├── auth.py
│   │   │   ├── afinidad.py
│   │   │   ├── consentimiento.py
│   │   │   ├── postulacion.py
│   │   │   ├── vacantes.py
│   │   │   ├── ranking.py
│   │   │   ├── indicadores.py
│   │   │   └── webhook.py
│   │   ├── schemas/                # Pydantic v2 – contratos REST
│   │   │   ├── auth.py
│   │   │   ├── afinidad.py
│   │   │   ├── consentimiento.py
│   │   │   ├── postulacion.py
│   │   │   ├── vacante.py
│   │   │   ├── ranking.py
│   │   │   └── indicadores.py
│   │   ├── models/                 # SQLAlchemy 2 – ORM
│   │   │   ├── base.py
│   │   │   ├── usuario.py
│   │   │   ├── perfil.py
│   │   │   ├── vacante.py
│   │   │   ├── postulacion.py
│   │   │   ├── consentimiento.py
│   │   │   ├── match_registro.py
│   │   │   ├── taller_evidencia.py
│   │   │   └── auditoria.py
│   │   └── services/               # lógica de negocio, orquesta pipeline y MCP
│   │       ├── afinidad_service.py
│   │       ├── consentimiento_service.py
│   │       ├── indicadores_service.py
│   │       └── webhook_service.py
│   └── web/                        # Next.js 15 (frontend)
│       ├── app/
│       │   ├── (estudiante)/
│       │   ├── (egresado)/
│       │   ├── (empresa)/
│       │   └── (ose)/
│       ├── components/
│       └── package.json
│
├── packages/
│   ├── agent/
│   │   └── pipeline/               # SOLO biblioteca estándar de Python
│   │       ├── __init__.py
│   │       ├── filtro_duro.py
│   │       ├── puntaje.py
│   │       ├── explicacion.py      # LLM + Validador + Plantilla
│   │       ├── validador.py
│   │       ├── plantilla.py
│   │       └── huella.py           # SHA-256 canónico
│   ├── mcp-server/                 # Servidor MCP semana 4
│   │   ├── server.py
│   │   ├── tools/                  # 9 herramientas
│   │   ├── resources/              # 3 recursos
│   │   └── postulacion_writer.py   # única escritura permitida
│   └── rag/                        # Módulo RAG local
│       ├── indexador.py
│       ├── consultor.py
│       └── corpus/                 # sílabos y reglamentos en PDF/TXT
│
├── data/
│   └── sintetica/
│       ├── generador.py            # semilla fija SEED=42
│       ├── perfiles.json
│       └── vacantes.json
│
├── tests/
│   ├── unit/
│   ├── property/                   # pytest + hypothesis
│   └── casos-agente/               # reglas duras ejecutables
│
├── scripts/
│   ├── verificar.ps1               # TODO EN VERDE
│   └── verificar_determinismo.py
│
├── alembic/                        # migraciones de BD
├── docker-compose.yml
├── pyproject.toml
└── README.md
```

### Docker Compose (esquema)

```yaml
services:
  db:
    image: pgvector/pgvector:pg16
    environment:
      POSTGRES_DB: undc_empleo
      POSTGRES_USER: dev
      POSTGRES_PASSWORD: dev
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    env_file: .env
    ports:
      - "8000:8000"
    depends_on:
      - db

volumes:
  pgdata:
```

---

## Data Models

### Principios

- **Sin Atributos Protegidos** como columnas: ninguna tabla almacena edad, género, foto, distrito, colegio ni estado civil.
- Las tablas de usuario usan identificadores UUID v4 generados en la aplicación.
- Toda columna de texto libre que provenga de entrada externa se inspecciona por el middleware de Atributos Protegidos antes de persistirse.
- El tipo `JSONB` de PostgreSQL almacena las tres etapas del pipeline en `match_registro`.

### Diagrama Entidad-Relación (simplificado)

```
usuario (id, codigo_univ, correo, rol, hash_contrasena, activo)
    │
    ├──< perfil (id, usuario_id, competencias JSONB, cursos_aprobados JSONB,
    │            talleres_verificados JSONB, actualizado_en)
    │
    ├──< consentimiento (id, egresado_id→usuario, tipo, estado,
    │                    creado_en_utc, revocado_en_utc, version)
    │
    ├──< postulacion (id, egresado_id→usuario, vacante_id→vacante,
    │                 estado, creado_en_utc, huella_sha256)
    │
    └──< taller_evidencia (id, usuario_id→usuario, taller_id→taller,
                           competencia_acreditada, registrado_en_utc)

vacante (id, empresa_id→usuario, titulo, descripcion,
         requisitos_duros JSONB, requisitos_deseables JSONB,
         estado, creado_en_utc, cerrado_en_utc)

match_registro (id, perfil_id→perfil, vacante_id→vacante,
                version_pipeline, huella_sha256,
                resultado_filtro JSONB, resultado_puntaje JSONB,
                resultado_explicacion JSONB, explicacion_origen,
                creado_en_utc)  -- INMUTABLE: sin UPDATE ni DELETE

taller (id, nombre, descripcion, competencia_asociada, activo)

auditoria_atributos_protegidos (id, endpoint, tipo_atributo,
                                 detectado_en_utc)

auditoria_webhook (id, taller_id_recibido, motivo_rechazo,
                   intento_en_utc)
```

### Tabla: `usuario`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID PK | generado en app |
| `codigo_univ` | VARCHAR(20) | nulo para empresa/OSE |
| `correo` | VARCHAR(254) UNIQUE | nulo para estudiante/egresado |
| `rol` | ENUM(`estudiante`,`egresado`,`empresa`,`ose`) | |
| `hash_contrasena` | VARCHAR(72) | bcrypt, costo ≥12 |
| `activo` | BOOLEAN | default TRUE |
| `creado_en_utc` | TIMESTAMPTZ | |

> No existen columnas de edad, género, foto, distrito, colegio ni estado civil.

### Tabla: `match_registro`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID PK | |
| `perfil_id` | UUID FK → perfil | |
| `vacante_id` | UUID FK → vacante | |
| `version_pipeline` | VARCHAR(20) | formato MAJOR.MINOR.PATCH |
| `huella_sha256` | CHAR(64) | hash canónico de la entrada |
| `resultado_filtro` | JSONB | salida de filtro_duro() |
| `resultado_puntaje` | JSONB | valor Decimal serializado como string |
| `resultado_explicacion` | JSONB | texto + origen |
| `explicacion_origen` | ENUM(`llm`,`plantilla`) | |
| `creado_en_utc` | TIMESTAMPTZ | |

> Esta tabla **no tiene sentencias UPDATE ni DELETE**; es append-only. Una regla de base de datos (trigger o política RLS) puede reforzarlo.

### Tabla: `consentimiento`

| Columna | Tipo | Notas |
|---|---|---|
| `id` | UUID PK | |
| `egresado_id` | UUID FK → usuario | |
| `tipo` | VARCHAR(50) | e.g. `compartir_con_empresas` |
| `estado` | ENUM(`activo`,`revocado`) | |
| `version` | INTEGER | incremental por egresado y tipo |
| `creado_en_utc` | TIMESTAMPTZ | |
| `revocado_en_utc` | TIMESTAMPTZ | nulo si activo |

> Retención mínima: 365 días desde `creado_en_utc`. Sin borrado físico.

### Índices Relevantes

```sql
-- Búsqueda de consentimiento activo (operación crítica en cada postulación)
CREATE INDEX ix_consentimiento_egresado_tipo_estado
    ON consentimiento (egresado_id, tipo, estado);

-- Determinismo: verificar duplicados de huella
CREATE UNIQUE INDEX ix_match_registro_huella
    ON match_registro (huella_sha256);

-- Ranking por vacante, ordenado por puntaje
CREATE INDEX ix_postulacion_vacante
    ON postulacion (vacante_id, estado);
```

---

## Components and Interfaces

### Pipeline Determinista

#### Principios de Diseño

- **Funciones puras de Python**: ninguna etapa tiene efectos secundarios; dado el mismo input, producen siempre el mismo output.
- **Sin LLM en Filtro Duro ni en Puntaje**: solo aritmética `Decimal` y comparaciones booleanas.
- **Importable por los tres entornos virtuales**: `packages/agent/pipeline/` no depende de ningún paquete externo.

#### Interfaz del Pipeline

```python
# packages/agent/pipeline/__init__.py
from .filtro_duro import ejecutar_filtro_duro   # → ResultadoFiltro
from .puntaje import calcular_puntaje           # → Decimal (0.00–100.00)
from .explicacion import generar_explicacion    # → ResultadoExplicacion
from .huella import calcular_huella             # → str (hex SHA-256)

def ejecutar_pipeline(perfil: dict, vacante: dict) -> ResultadoPipeline:
    entrada_canonica = _canonicalizar(perfil, vacante)
    huella = calcular_huella(entrada_canonica)
    filtro = ejecutar_filtro_duro(perfil, vacante)
    puntaje = calcular_puntaje(perfil, vacante, filtro)
    explicacion = generar_explicacion(filtro, puntaje, perfil, vacante)
    return ResultadoPipeline(
        huella_sha256=huella,
        filtro=filtro,
        puntaje=puntaje,
        explicacion=explicacion,
    )
```

#### Etapa 1 — Filtro Duro

```python
# packages/agent/pipeline/filtro_duro.py
from dataclasses import dataclass

@dataclass(frozen=True)
class ResultadoFiltro:
    cumple_global: bool
    resultados: tuple[dict, ...]  # {requisito, cumple, motivo}

def ejecutar_filtro_duro(perfil: dict, vacante: dict) -> ResultadoFiltro:
    """
    Para cada requisito duro de la vacante, evalúa si el perfil lo satisface.
    Devuelve True solo si todos los requisitos duros se cumplen.
    Función pura: sin IO, sin LLM, sin efectos secundarios.
    """
    requisitos_duros = vacante.get("requisitos_duros", [])
    if not requisitos_duros:
        return ResultadoFiltro(cumple_global=True, resultados=())
    
    resultados = []
    for req in requisitos_duros:
        cumple = _evaluar_requisito(req, perfil)
        resultados.append({
            "requisito": req["nombre"],
            "cumple": cumple,
            "motivo": None if cumple else f"No se encontró '{req['nombre']}' en el perfil",
        })
    
    cumple_global = all(r["cumple"] for r in resultados)
    return ResultadoFiltro(cumple_global=cumple_global, resultados=tuple(resultados))

def _evaluar_requisito(requisito: dict, perfil: dict) -> bool:
    competencias = {c.lower() for c in perfil.get("competencias", [])}
    return requisito["nombre"].lower() in competencias
```

#### Etapa 2 — Puntaje

```python
# packages/agent/pipeline/puntaje.py
from decimal import Decimal, ROUND_HALF_UP

PESO_DURO = Decimal("0.60")
PESO_DESEABLE = Decimal("0.40")
DOS_DECIMALES = Decimal("0.01")

def calcular_puntaje(
    perfil: dict,
    vacante: dict,
    filtro: "ResultadoFiltro",
) -> Decimal | None:
    """
    Devuelve un Decimal de 0.00 a 100.00, o None si la vacante no declara requisitos.
    Función pura: aritmética Decimal, sin LLM, sin IO.
    """
    req_duros = vacante.get("requisitos_duros", [])
    req_deseables = vacante.get("requisitos_deseables", [])
    
    if not req_duros and not req_deseables:
        return None  # vacante sin requisitos declarados
    
    puntaje_duro = _calcular_cobertura(perfil, req_duros)
    puntaje_deseable = _calcular_cobertura(perfil, req_deseables)
    
    total = (puntaje_duro * PESO_DURO + puntaje_deseable * PESO_DESEABLE)
    return (total * Decimal("100")).quantize(DOS_DECIMALES, rounding=ROUND_HALF_UP)

def _calcular_cobertura(perfil: dict, requisitos: list[dict]) -> Decimal:
    if not requisitos:
        return Decimal("0")
    competencias = {c.lower() for c in perfil.get("competencias", [])}
    cumplidos = sum(1 for r in requisitos if r["nombre"].lower() in competencias)
    return Decimal(str(cumplidos)) / Decimal(str(len(requisitos)))
```

#### Etapa 3 — Explicación (LLM + Validador + Plantilla)

```python
# packages/agent/pipeline/explicacion.py
import os
from .validador import validar_texto_explicacion
from .plantilla import generar_plantilla_determinista

def generar_explicacion(filtro, puntaje, perfil_anonimizado, vacante) -> dict:
    """
    Intenta redactar con el LLM. Si falla la validación o no hay clave,
    cae en la plantilla determinista. El LLM nunca recibe nombre ni código.
    """
    estructura = _armar_estructura(filtro, puntaje, vacante)
    
    if not _hay_clave_llm():
        return {"texto": generar_plantilla_determinista(estructura),
                "origen": "plantilla", "motivo_rechazo": None}
    
    texto_llm = _invocar_llm(estructura)  # sin nombre, sin código
    es_valido, motivo = validar_texto_explicacion(texto_llm, estructura)
    
    if es_valido:
        return {"texto": texto_llm, "origen": "llm", "motivo_rechazo": None}
    else:
        return {"texto": generar_plantilla_determinista(estructura),
                "origen": "plantilla", "motivo_rechazo": motivo}

def _hay_clave_llm() -> bool:
    return bool(os.environ.get("GEMINI_API_KEY") or os.environ.get("GROQ_API_KEY"))
```

#### Huella SHA-256

```python
# packages/agent/pipeline/huella.py
import hashlib
import json

def calcular_huella(entrada_canonica: str) -> str:
    """Devuelve el hex digest SHA-256 de la entrada canónica UTF-8."""
    return hashlib.sha256(entrada_canonica.encode("utf-8")).hexdigest()

def canonicalizar_par(perfil: dict, vacante: dict) -> str:
    """
    Serializa el par perfil-vacante de forma determinista:
    claves ordenadas alfabéticamente, sin espacios, separadores mínimos.
    """
    datos = {"perfil": perfil, "vacante": vacante}
    return json.dumps(datos, sort_keys=True, separators=(",", ":"), ensure_ascii=True)
```

---

### Servidor MCP

#### Rol y Restricciones

El Servidor MCP (`packages/mcp-server/`) construido en la semana 4 es el único punto de acceso estructurado a la base de datos desde el pipeline y la API. Es **de solo lectura** con una única excepción: el registro de Postulación, que requiere consentimiento verificado.

Toda operación de escritura distinta a esa excepción devuelve **HTTP 405**.

#### Las 9 Herramientas

| # | Nombre | Descripción | Parámetros |
|---|---|---|---|
| 1 | `get_perfil` | Devuelve el perfil de un usuario | `usuario_id: str` |
| 2 | `get_vacante` | Devuelve una vacante por ID | `vacante_id: str` |
| 3 | `listar_vacantes_activas` | Lista todas las vacantes en estado `activa` | `empresa_id: str \| None` |
| 4 | `get_postulacion` | Devuelve una postulación por ID | `postulacion_id: str` |
| 5 | `listar_postulaciones_vacante` | Lista postulantes con consentimiento activo | `vacante_id: str` |
| 6 | `get_consentimiento_activo` | Verifica si un egresado tiene consentimiento activo | `egresado_id: str, tipo: str` |
| 7 | `listar_cursos` | Lista cursos disponibles con códigos | `—` |
| 8 | `get_taller` | Devuelve un taller por ID | `taller_id: str` |
| 9 | `registrar_postulacion` | **Única escritura**: crea postulación con consentimiento verificado | `egresado_id, vacante_id, timestamp_iso` |

#### Los 3 Recursos

| # | URI | Descripción |
|---|---|---|
| 1 | `mcp://undc-empleo/catalogo-cursos` | Catálogo completo de cursos UNDC con código y nombre |
| 2 | `mcp://undc-empleo/catalogo-talleres` | Catálogo de talleres con competencia asociada |
| 3 | `mcp://undc-empleo/version-pipeline` | Versión actual del pipeline en formato MAJOR.MINOR.PATCH |

#### Flujo de `registrar_postulacion`

```python
# packages/mcp-server/postulacion_writer.py
def registrar_postulacion(egresado_id: str, vacante_id: str, timestamp_iso: str) -> dict:
    # 1. Verificar consentimiento activo (SIEMPRE, antes de cualquier escritura)
    consentimiento = db.query_consentimiento_activo(egresado_id, "compartir_con_empresas")
    if not consentimiento:
        raise MCPError(403, "Consentimiento 'compartir_con_empresas' no activo")
    
    # 2. Verificar vacante activa
    vacante = db.query_vacante(vacante_id)
    if vacante["estado"] != "activa":
        raise MCPError(422, f"Vacante en estado '{vacante['estado']}'")
    
    # 3. Verificar duplicados
    existente = db.query_postulacion_activa(egresado_id, vacante_id)
    if existente:
        raise MCPError(409, f"Postulación existente: {existente['id']}")
    
    # 4. Calcular huella SHA-256 canónica
    huella = calcular_huella(f"{egresado_id}|{vacante_id}|{timestamp_iso}")
    
    # 5. Persistir (única operación de escritura)
    return db.insert_postulacion(egresado_id, vacante_id, timestamp_iso, huella)
```

---

### Módulo RAG

#### Arquitectura

```
corpus/                          ChromaDB (local)
  sílabos/*.pdf  ──► indexador ──► colección "silabos"
  reglamentos/*.txt               fragmentos + metadatos
                                      │
                    consultor ◄───────┘
                        │
                 [ (fragmento, score, metadatos) ]
                        │
               umbral coseno ≥ 0.50 (configurable)
               umbral Ruta de Cierre ≥ 0.70
```

#### Indexador

```python
# packages/rag/indexador.py
from sentence_transformers import SentenceTransformer
import chromadb

MODELO_EMBEDDING = "paraphrase-multilingual-MiniLM-L12-v2"
TAMAÑO_FRAGMENTO = 400        # caracteres
SOLAPAMIENTO = 80             # caracteres de solapamiento entre fragmentos

def indexar_corpus(ruta_corpus: str, ruta_chroma: str) -> None:
    modelo = SentenceTransformer(MODELO_EMBEDDING)
    cliente = chromadb.PersistentClient(path=ruta_chroma)
    coleccion = cliente.get_or_create_collection(
        name="silabos",
        metadata={"hnsw:space": "cosine"},
    )
    for doc_path in _listar_documentos(ruta_corpus):
        fragmentos = _fragmentar(doc_path)
        for frag in fragmentos:
            embedding = modelo.encode(frag["texto"]).tolist()
            coleccion.add(
                ids=[frag["id"]],
                embeddings=[embedding],
                documents=[frag["texto"]],
                metadatas=[{
                    "documento": frag["documento"],
                    "codigo_curso": frag["codigo_curso"],
                    "seccion": frag["seccion"],
                }],
            )
```

#### Consultor

```python
# packages/rag/consultor.py
UMBRAL_DEFAULT = 0.50
UMBRAL_RUTA_CIERRE = 0.70
MAX_FRAGMENTO_CHARS = 300

def consultar(brecha: str, umbral: float = UMBRAL_DEFAULT, top_k: int = 5) -> list[dict]:
    """
    Devuelve fragmentos ordenados por similitud coseno descendente.
    Solo incluye fragmentos con score >= umbral.
    """
    modelo = _get_modelo()  # singleton cargado una vez al inicio
    coleccion = _get_coleccion()
    
    embedding = modelo.encode(brecha).tolist()
    resultados = coleccion.query(
        query_embeddings=[embedding],
        n_results=top_k,
        include=["documents", "metadatas", "distances"],
    )
    
    fragmentos = []
    for doc, meta, dist in zip(
        resultados["documents"][0],
        resultados["metadatas"][0],
        resultados["distances"][0],
    ):
        score = 1.0 - dist  # ChromaDB devuelve distancia coseno; convertir a similitud
        if score >= umbral:
            fragmentos.append({
                "texto": doc[:MAX_FRAGMENTO_CHARS],
                "similitud": round(score, 4),
                "codigo_curso": meta["codigo_curso"],
                "nombre_curso": meta.get("nombre_curso", ""),
                "seccion": meta["seccion"],
                "documento": meta["documento"],
            })
    
    return sorted(fragmentos, key=lambda x: x["similitud"], reverse=True)
```

#### Generación de Ruta de Cierre

La ruta de cierre se construye en el servicio de afinidad de la API:

1. Para cada brecha detectada por el Filtro Duro o el Puntaje, se consulta el RAG con `umbral=0.70`.
2. Se cruza el código de curso devuelto contra el Servidor MCP (herramienta `listar_cursos`).
3. Si el MCP no confirma el código, se excluye y se declara en `brechas_sin_cobertura`.
4. El orden final de la lista es descendente por `similitud` (RAG); el LLM **no** reordena.
5. Cada ítem incluye: `codigo_curso`, `nombre_curso`, `seccion`, `fragmento` (≤300 chars).

---

### Autenticación JWT

#### Flujo de Tokens

```
POST /auth/login
  ├── Verificar credenciales (bcrypt, costo ≥12)
  ├── Emitir access_token  (exp: 8h, campo: rol)
  └── Emitir refresh_token (exp: 7d)

POST /auth/refresh
  ├── Validar refresh_token (no vencido, no en lista negra)
  └── Emitir nuevo access_token (exp: 8h)

Todas las rutas protegidas:
  └── Bearer {access_token} → middleware verifica firma + expiración + rol
```

#### Payload del JWT

```json
{
  "sub": "uuid-del-usuario",
  "rol": "estudiante" | "egresado" | "empresa" | "ose",
  "iat": 1700000000,
  "exp": 1700028800
}
```

> `sub` es el UUID interno; **no** es el código universitario ni el correo.

#### Implementación (esquema)

```python
# apps/api/routers/auth.py
import bcrypt
import jwt
from datetime import datetime, timedelta, timezone

ACCESS_EXP  = timedelta(hours=8)
REFRESH_EXP = timedelta(days=7)
ALGORITMO   = "HS256"

def crear_access_token(usuario_id: str, rol: str) -> str:
    ahora = datetime.now(timezone.utc)
    payload = {"sub": usuario_id, "rol": rol,
               "iat": ahora, "exp": ahora + ACCESS_EXP}
    return jwt.encode(payload, _secret(), algorithm=ALGORITMO)

def verificar_hash(contrasena_plana: str, hash_almacenado: str) -> bool:
    return bcrypt.checkpw(
        contrasena_plana.encode("utf-8"),
        hash_almacenado.encode("utf-8"),
    )

def crear_hash(contrasena_plana: str) -> str:
    salt = bcrypt.gensalt(rounds=12)
    return bcrypt.hashpw(contrasena_plana.encode("utf-8"), salt).decode("utf-8")
```

#### Control de Acceso por Rol

| Endpoint | Roles permitidos |
|---|---|
| `GET /afinidad/{vacante_id}` | `estudiante`, `egresado` |
| `POST /postulacion` | `egresado` |
| `GET /ranking/{vacante_id}` | `empresa` |
| `POST /vacantes` | `empresa` |
| `GET /indicadores` | `ose` |
| `POST /webhook/asistencia` | sin autenticación JWT (HMAC-SHA256) |

---

### Frontend — Cuatro Portales

#### Tecnologías

- **Next.js 15** con App Router
- **React 19**
- **Tailwind CSS v4**
- **Recharts** para gráficos de indicadores (portal OSE)

#### Estructura de Rutas (App Router)

```
apps/web/app/
├── (estudiante)/
│   ├── layout.tsx
│   ├── dashboard/page.tsx          # vacantes disponibles + buscador
│   └── afinidad/[vacante_id]/
│       └── page.tsx                # resultado pipeline: puntaje, brechas, ruta
│
├── (egresado)/
│   ├── layout.tsx
│   ├── dashboard/page.tsx          # vacantes + postulaciones activas
│   ├── afinidad/[vacante_id]/
│   │   └── page.tsx
│   ├── postular/[vacante_id]/
│   │   └── page.tsx                # confirmación con estado consentimiento
│   └── consentimiento/page.tsx     # activar / revocar / historial
│
├── (empresa)/
│   ├── layout.tsx
│   ├── vacantes/page.tsx           # listado + crear vacante
│   ├── vacantes/[id]/page.tsx      # editar / cerrar vacante
│   └── ranking/[vacante_id]/
│       └── page.tsx                # ranking explicado con aviso de no-descarte
│
├── (ose)/
│   ├── layout.tsx
│   └── indicadores/page.tsx        # gráficos Recharts + aviso datos sintéticos
│
└── login/page.tsx                  # único punto de entrada por rol
```

#### Componentes Clave

| Componente | Descripción |
|---|---|
| `ResultadoPipeline` | Muestra puntaje, coincidencias, brechas y ruta de cierre; incluye badge `llm` o `plantilla` |
| `RankingTable` | Tabla ordenada con aviso de no-descarte en cada fila |
| `ConsentimientoPanel` | Toggle activar/revocar + historial cronológico |
| `IndicadoresOSE` | Gráficos de barras (Recharts) por cohorte + aviso datos sintéticos |
| `PrivacyGuard` | HOC que verifica rol y consentimiento antes de renderizar datos sensibles |

#### Aviso R5 en el Ranking

Cada fila del ranking renderiza este aviso fijo:

> "La decisión de contratación corresponde exclusivamente a una persona; este sistema no descarta candidatos."

---

### Webhook de Asistencia

#### Flujo

```
POST /webhook/asistencia
  Body: { usuario_id, taller_id, timestamp_iso, firma_hmac }
  
  1. Verificar firma HMAC-SHA256:
       esperada = HMAC(WEBHOOK_SECRET, f"{usuario_id}|{taller_id}|{timestamp_iso}")
       si no coincide → 401, registrar en auditoria_webhook
  
  2. Verificar existencia del taller en catálogo:
       si no existe → 404, registrar en auditoria_webhook
  
  3. Registrar en taller_evidencia (ÚNICA tabla que se modifica)
  
  4. Marcar competencia como acreditada_por_asistencia en el perfil del usuario
  
  5. Disparar recálculo del pipeline para todas las afinidades previas
     que incluyan la competencia acreditada
  
  6. Devolver 200 con { postulacion_id, competencia_acreditada, recalculos }
```

#### Restricción de Tablas

- `taller_evidencia`: se **inserta** el registro de asistencia.
- `perfil`: se **actualiza** `talleres_verificados` con la nueva competencia.
- `historia_academica`: **no se toca bajo ninguna circunstancia**.
- `cv_usuario`: **no se toca bajo ninguna circunstancia**.

---

### Contratos de API REST

#### Autenticación

##### `POST /auth/login`

```python
# Request
class LoginRequest(BaseModel):
    identificador: str  # codigo_univ (estudiante/egresado) o correo (empresa/ose)
    contrasena: str

# Response 200
class LoginResponse(BaseModel):
    access_token: str
    refresh_token: str
    token_type: Literal["bearer"]
    rol: Literal["estudiante", "egresado", "empresa", "ose"]
    expira_en: int  # segundos hasta expiración del access_token

# Response 401
class ErrorAutenticacion(BaseModel):
    detail: str  # mensaje genérico, sin distinguir campo erróneo
```

##### `POST /auth/refresh`

```python
class RefreshRequest(BaseModel):
    refresh_token: str

class RefreshResponse(BaseModel):
    access_token: str
    token_type: Literal["bearer"]
    expira_en: int
```

#### Afinidad

##### `GET /afinidad/{vacante_id}`

```python
# Response 200
class ItemRutaCierre(BaseModel):
    codigo_curso: str
    nombre_curso: str
    seccion: str
    fragmento: str  # max 300 chars
    similitud: float

class ResultadoAfinidad(BaseModel):
    vacante_id: str
    puntaje: Decimal | None            # None si vacante sin requisitos
    coincidencias: list[str]
    brechas: list[str]
    brechas_sin_cobertura: list[str]   # brechas sin respaldo RAG
    ruta_cierre: list[ItemRutaCierre]
    explicacion: str
    explicacion_origen: Literal["llm", "plantilla"]
    huella_sha256: str
    version_pipeline: str
```

#### Consentimiento

##### `POST /consentimiento/activar`

```python
class ConsentimientoActivarRequest(BaseModel):
    tipo: Literal["compartir_con_empresas"]

class ConsentimientoResponse(BaseModel):
    id: str
    tipo: str
    estado: Literal["activo", "revocado"]
    version: int
    creado_en_utc: datetime
    revocado_en_utc: datetime | None
```

##### `POST /consentimiento/revocar`

```python
# Request: mismo body que activar
# Response 200: ConsentimientoResponse con estado="revocado"
# Response 409: {"detail": "No existe consentimiento activo que revocar"}
```

##### `GET /consentimiento/historial`

```python
class HistorialConsentimientoResponse(BaseModel):
    registros: list[ConsentimientoResponse]  # orden cronológico descendente
```

#### Postulación

##### `POST /postulacion`

```python
class PostulacionRequest(BaseModel):
    vacante_id: str

class PostulacionResponse(BaseModel):
    id: str
    vacante_id: str
    estado: Literal["pendiente"]
    creado_en_utc: datetime
    huella_sha256: str
```

#### Vacantes

##### `POST /vacantes`

```python
class RequisitoVacante(BaseModel):
    nombre: str
    descripcion: str | None = None

class VacanteRequest(BaseModel):
    titulo: str
    descripcion: str
    requisitos_duros: list[RequisitoVacante] = []
    requisitos_deseables: list[RequisitoVacante] = []

class VacanteResponse(BaseModel):
    id: str
    titulo: str
    estado: Literal["activa", "incompleta"]
    advertencia: str | None = None  # presente cuando estado = "incompleta"
    creado_en_utc: datetime
```

#### Ranking

##### `GET /ranking/{vacante_id}`

```python
class EntradaRanking(BaseModel):
    egresado_id: str              # UUID; sin nombre, sin código
    puntaje: Decimal
    coincidencias: list[str]
    brechas: list[str]
    ruta_cierre_resumida: list[ItemRutaCierre]  # max 3 ítems
    cumple_filtro_duro: bool
    motivos_incumplimiento: list[str]  # vacío si cumple_filtro_duro=True
    explicacion: str
    aviso: str = "La decisión de contratación corresponde exclusivamente a una persona; este sistema no descarta candidatos."

class RankingResponse(BaseModel):
    vacante_id: str
    total: int
    postulantes: list[EntradaRanking]
    nota: str | None = None  # presente cuando total=0
```

#### Indicadores OSE

##### `GET /indicadores`

```python
class IndicadoresCohorte(BaseModel):
    anio_egreso: int
    carrera: str
    egresados_activos: int
    postulaciones: int
    tasa_postulaciones: float
    consentimientos_activos: int
    talleres_verificados: int

class IndicadoresResponse(BaseModel):
    cohortes: list[IndicadoresCohorte]
    cohortes_omitidas: int              # cohortes con <5 egresados
    nota_datos_sinteticos: str | None   # presente cuando origen=sintético
```

#### Webhook de Asistencia

##### `POST /webhook/asistencia`

```python
class WebhookAsistenciaRequest(BaseModel):
    usuario_id: str
    taller_id: str
    timestamp_iso: str  # ISO 8601 UTC
    firma_hmac: str     # HMAC-SHA256 hex

class WebhookAsistenciaResponse(BaseModel):
    asistencia_id: str
    competencia_acreditada: str
    recalculos: int  # número de afinidades recalculadas
    registrado_en_utc: datetime
```

---

## Error Handling

### Validador de Explicaciones LLM

El validador rechaza el texto del LLM si detecta alguna de estas condiciones y activa el fallback a plantilla determinista:

| Condición | Método de detección |
|---|---|
| Valores numéricos no presentes en la estructura calculada | regex + comparación con `estructura["puntaje"]` |
| Códigos de curso no devueltos por el Servidor MCP | set lookup contra `estructura["codigos_validos"]` |
| Promesas de empleo | regex de frases prohibidas: `"garantiz", "asegur", "obtendr", "conseguir"` |
| Descarte de candidatos | regex: `"no clasificó", "eliminado", "descartado", "rechazado"` |
| Mención de Atributos Protegidos | lista de términos: edad, género, foto, distrito, colegio, estado civil |

Cuando el validador rechaza el texto del LLM, el campo `explicacion_origen` de la respuesta es siempre `"plantilla"` y `motivo_rechazo` describe la condición que falló.

### Middleware de Atributos Protegidos

```python
# apps/api/middleware/atributos_protegidos.py
ATRIBUTOS_PROHIBIDOS = frozenset([
    "edad", "genero", "género", "foto", "imagen",
    "distrito", "colegio", "estado_civil",
])

def inspeccionar_y_descartar(datos: dict, endpoint: str) -> tuple[dict, list[str]]:
    """
    Elimina recursivamente cualquier clave prohibida del diccionario.
    Devuelve (datos_limpios, lista_de_atributos_descartados).
    """
    descartados = []
    datos_limpios = _limpiar_recursivo(datos, descartados)
    if descartados:
        _registrar_auditoria(endpoint, descartados)
    return datos_limpios, descartados
```

### Verificación HMAC del Webhook

```python
# apps/api/services/webhook_service.py
import hmac, hashlib, os

def verificar_firma_webhook(
    usuario_id: str, taller_id: str, timestamp_iso: str, firma_recibida: str
) -> bool:
    secreto = os.environ["WEBHOOK_SECRET"].encode("utf-8")
    mensaje = f"{usuario_id}|{taller_id}|{timestamp_iso}".encode("utf-8")
    firma_esperada = hmac.new(secreto, mensaje, hashlib.sha256).hexdigest()
    return hmac.compare_digest(firma_esperada, firma_recibida)
```

> Se usa `hmac.compare_digest` para evitar ataques de temporización.

### Códigos de Error por Escenario

| Escenario | Código HTTP | Comportamiento |
|---|---|---|
| Credenciales inválidas en login | 401 | Mensaje genérico; no distingue campo erróneo |
| Token JWT vencido o inválido | 401 | Rechaza con `detail` genérico |
| Rol sin permiso para el endpoint | 403 | Rechaza antes de ejecutar lógica de negocio |
| Consentimiento no activo al postular | 403 | No transmite ningún dato del egresado |
| Postulación duplicada | 409 | Devuelve `id` de la postulación existente |
| Atributos protegidos en vacante | 422 | Identifica los campos que deben corregirse |
| Vacante no activa al postular | 422 | Mensaje con estado actual de la vacante |
| Firma HMAC inválida en webhook | 401 | Registra en `auditoria_webhook` con marca de tiempo |
| Taller no encontrado en webhook | 404 | Registra en `auditoria_webhook` |
| Escritura no permitida en MCP | 405 | Rechaza toda operación de escritura distinta a postulación |
| Cohorte con menos de 5 egresados | — | Omite la cohorte; incrementa `cohortes_omitidas` |

---

## Testing Strategy

### Estructura de Tests

```
tests/
├── unit/           # pruebas unitarias por módulo
├── property/       # pytest + hypothesis (pruebas basadas en propiedades)
└── casos-agente/   # reglas duras ejecutables contra el pipeline
```

Herramientas: **pytest** + **hypothesis** para property-based testing.

### Enfoque por Capa

#### Pipeline (unit + property)

Las tres etapas del pipeline son funciones puras, lo que las hace ideales para property-based testing:

- **Filtro Duro**: para cualquier perfil y vacante generados aleatoriamente, `ejecutar_filtro_duro` devuelve `cumple_global=True` si y solo si todos los requisitos duros están en el perfil.
- **Puntaje**: para cualquier par válido, el resultado es `Decimal` en `[0.00, 100.00]` con exactamente dos decimales; o `None` cuando no hay requisitos.
- **Determinismo**: ejecutar el pipeline dos veces con el mismo input produce la misma huella SHA-256 y el mismo puntaje (propiedad de idempotencia).

#### Servidor MCP (unit)

- Verificar que todas las herramientas de solo lectura rechazan escrituras con HTTP 405.
- Verificar que `registrar_postulacion` comprueba consentimiento antes de persistir.
- Verificar detección de postulaciones duplicadas (HTTP 409).

#### Módulo RAG (unit + integración)

- Verificar que todos los fragmentos retornados tienen `similitud >= umbral`.
- Verificar que la lista está ordenada de mayor a menor similitud.
- Verificar que ningún fragmento excede `MAX_FRAGMENTO_CHARS`.

#### API FastAPI (integración)

- Verificar control de acceso por rol para cada endpoint.
- Verificar que el ranking no incluye egresados sin consentimiento activo.
- Verificar que los indicadores omiten cohortes con menos de 5 egresados.
- Verificar que ninguna respuesta de ranking contiene atributos protegidos.

#### Verificación de Determinismo Cruzado

```bash
# scripts/verificar_determinismo.py
# Ejecuta el mismo par perfil-vacante por dos caminos:
#   1. API FastAPI  (POST /afinidad)
#   2. Servidor MCP (tool: get_perfil + get_vacante + pipeline directo)
# Compara huella_sha256 y puntaje; falla si difieren.
python scripts/verificar_determinismo.py --perfil-id perfil-sintetico-0001 --vacante-id vacante-sintetica-0005
```

### Datos Sintéticos para Tests

```python
# data/sintetica/generador.py
import random
from decimal import Decimal

SEED = 42  # NUNCA MODIFICAR; garantiza reproducibilidad

def generar_dataset(n_perfiles: int = 200, n_vacantes: int = 50) -> dict:
    rng = random.Random(SEED)
    perfiles = [_generar_perfil(rng, i) for i in range(n_perfiles)]
    vacantes = [_generar_vacante(rng, i) for i in range(n_vacantes)]
    return {"perfiles": perfiles, "vacantes": vacantes}

def _generar_perfil(rng: random.Random, idx: int) -> dict:
    # SIN Atributos Protegidos en el objeto generado
    return {
        "id": f"perfil-sintetico-{idx:04d}",
        "competencias": rng.sample(COMPETENCIAS_DISPONIBLES, k=rng.randint(3, 12)),
        "cursos_aprobados": rng.sample(CURSOS_UNDC, k=rng.randint(10, 40)),
        "talleres_verificados": [],
        # No contiene: edad, género, foto, distrito, colegio, estado_civil
    }
```

#### Reglas de Privacidad en Datos Sintéticos

| Regla | Implementación |
|---|---|
| Sin datos reales | Solo `generador.py` con semilla fija; no hay ingesta de CSV reales |
| Sin Atributos Protegidos | El generador no crea esas columnas; Pydantic v2 rechaza si aparecen |
| Aviso en Indicadores | `nota_datos_sinteticos` en `IndicadoresResponse` siempre presente en desarrollo |
| Semilla declarada | El aviso incluye: "correlación sembrada con SEED=42; no refleja datos reales" |
| Ley N.° 29733 | Ningún dato personal real se almacena ni transmite en el entorno de desarrollo |

---

## Correctness Properties

*Una propiedad es una característica o comportamiento que debe mantenerse verdadero en todas las ejecuciones válidas del sistema. Las propiedades sirven como puente entre las especificaciones legibles por humanos y las garantías de corrección verificables automáticamente.*

---

### Property 1: Determinismo del Pipeline

*Para cualquier* par Perfil-Vacante válido, ejecutar el pipeline en dos momentos distintos, en cualquier orden y desde cualquier entorno de ejecución, produce exactamente el mismo Puntaje (Decimal) y la misma Huella SHA-256.

**Validates: Requirements 2.3**

---

### Property 2: Puntaje con dos decimales exactos

*Para cualquier* par Perfil-Vacante con al menos un requisito declarado, el Puntaje resultante es un valor `Decimal` con exactamente dos posiciones decimales, comprendido en el intervalo cerrado [0.00, 100.00].

**Validates: Requirements 2.2**

---

### Property 3: Registro inmutable del pipeline

*Para cualquier* ejecución del pipeline, el registro creado en `match_registro` contiene la Huella SHA-256, la versión del pipeline, las tres etapas en JSONB y la marca de tiempo; y ese registro no puede ser modificado ni eliminado posteriormente.

**Validates: Requirements 2.4, 12.6**

---

### Property 4: El Validador rechaza texto prohibido

*Para cualquier* texto generado por el LLM que contenga al menos uno de los elementos prohibidos (cifras no presentes en la estructura calculada, códigos de curso inexistentes en el MCP, promesas de empleo, descarte de candidatos, o mención de Atributos Protegidos), el Validador rechaza el texto y el campo `explicacion_origen` de la respuesta es `"plantilla"`.

**Validates: Requirements 2.5**

---

### Property 5: Estructura completa de la respuesta de afinidad

*Para cualquier* consulta de afinidad sobre una Vacante con requisitos declarados, la respuesta JSON contiene todos los campos obligatorios: `puntaje`, `coincidencias`, `brechas`, `ruta_cierre`, `explicacion`, `explicacion_origen`, `huella_sha256` y `version_pipeline`.

**Validates: Requirements 2.7**

---

### Property 6: Ruta de cierre ordenada por similitud

*Para cualquier* Ruta de Cierre con más de un ítem, la lista está ordenada de mayor a menor puntuación de similitud coseno, y ningún ítem tiene similitud inferior a 0.70.

**Validates: Requirements 3.5, 3.6**

---

### Property 7: Cobertura parcial declarada explícitamente

*Para cualquier* perfil con N brechas donde el Módulo RAG solo encuentra respaldo (similitud ≥ 0.70) para M < N brechas, la respuesta incluye exactamente M ítems en `ruta_cierre` y exactamente N−M brechas en `brechas_sin_cobertura`.

**Validates: Requirements 3.2, 3.4**

---

### Property 8: Fragmentos del RAG ordenados por similitud descendente

*Para cualquier* consulta al Módulo RAG con un texto de brecha y un umbral configurado, la lista devuelta está ordenada de mayor a menor similitud coseno y todos sus elementos tienen similitud estrictamente mayor o igual al umbral solicitado.

**Validates: Requirements 11.2**

---

### Property 9: Idempotencia de activación de consentimiento

*Para cualquier* egresado que ya tiene un consentimiento `compartir_con_empresas` activo, activar el mismo consentimiento nuevamente devuelve el identificador del registro existente sin crear un duplicado; el número de registros activos para ese egresado y ese tipo permanece exactamente en 1.

**Validates: Requirements 4.1**

---

### Property 10: Invariante de consentimiento en operaciones de compartición

*Para cualquier* operación que comparte datos de un egresado con una empresa (postular, listar en ranking), si el egresado no tiene consentimiento `compartir_con_empresas` activo al momento de la operación, el sistema la bloquea con HTTP 403 sin transmitir ningún dato del egresado.

**Validates: Requirements 4.3, 4.6**

---

### Property 11: Historial de consentimientos en orden cronológico descendente

*Para cualquier* egresado con N registros de consentimiento, la respuesta a `GET /consentimiento/historial` contiene exactamente N registros ordenados por `creado_en_utc` de más reciente a más antiguo.

**Validates: Requirements 4.5**

---

### Property 12: Postulación con huella SHA-256 canónica

*Para cualquier* postulación exitosa, la Huella SHA-256 almacenada es igual a `SHA256("{egresado_id}|{vacante_id}|{timestamp_iso}")` calculada de forma independiente; y la respuesta incluye `id`, `creado_en_utc` y el estado inicial `"pendiente"`.

**Validates: Requirements 5.1, 5.5**

---

### Property 13: Rechazo de postulación duplicada

*Para cualquier* egresado que ya tiene una postulación con estado `pendiente` o `en_proceso` para la misma vacante, intentar crear una nueva postulación resulta siempre en HTTP 409 con el identificador de la postulación existente.

**Validates: Requirements 5.2**

---

### Property 14: Solo-lectura del Servidor MCP

*Para cualquier* operación de escritura enviada al Servidor MCP que no sea el registro de postulación con consentimiento verificado, el servidor responde siempre con HTTP 405.

**Validates: Requirements 5.4, 10.3**

---

### Property 15: Vacante con requisitos inicia como activa

*Para cualquier* solicitud de creación de vacante con al menos un requisito (duro o deseable), la vacante persiste con estado `"activa"`; para cualquier solicitud sin ningún requisito, la vacante persiste con estado `"incompleta"` y la respuesta contiene una advertencia.

**Validates: Requirements 6.1, 6.2**

---

### Property 16: Atributos Protegidos rechazados en vacantes

*Para cualquier* solicitud de creación o actualización de vacante que contenga al menos un Atributo Protegido como requisito, la API responde con HTTP 422 e identifica los campos que deben corregirse.

**Validates: Requirements 6.6**

---

### Property 17: Ranking incluye solo egresados con consentimiento activo

*Para cualquier* consulta del ranking de una vacante, la lista de postulantes nunca contiene a un egresado cuyo consentimiento `compartir_con_empresas` esté revocado o no exista al momento de la consulta.

**Validates: Requirements 7.1, 7.5**

---

### Property 18: Ranking ordenado y sin eliminaciones

*Para cualquier* ranking con N postulantes con consentimiento activo, la lista contiene exactamente N entradas ordenadas de mayor a menor puntaje; los postulantes que no cumplen el Filtro Duro aparecen al final con motivos explícitos, sin ser eliminados.

**Validates: Requirements 7.2, 7.3**

---

### Property 19: Cada entrada del ranking está libre de Atributos Protegidos

*Para cualquier* entrada del ranking, ningún campo contiene valores de edad, género, foto, distrito, colegio ni estado civil; y el campo `aviso` contiene siempre la cadena de no-descarte.

**Validates: Requirements 7.4, 7.7**

---

### Property 20: k-anonimato en indicadores OSE (k ≥ 5)

*Para cualquier* consulta de indicadores, la respuesta nunca incluye cifras de una cohorte con menos de 5 egresados; y la respuesta nunca contiene identificadores individuales de egresados.

**Validates: Requirements 8.1, 8.4**

---

### Property 21: Métricas completas por cohorte

*Para cualquier* cohorte con al menos 5 egresados activos en el sistema, la respuesta de indicadores incluye las cinco métricas definidas: `egresados_activos`, `postulaciones`, `tasa_postulaciones`, `consentimientos_activos` y `talleres_verificados`.

**Validates: Requirements 8.2**

---

### Property 22: Aviso de datos sintéticos siempre presente

*Para cualquier* respuesta de indicadores generada sobre datos sintéticos, el campo `nota_datos_sinteticos` está presente y no es nulo.

**Validates: Requirements 8.3**

---

### Property 23: Webhook registra en taller_evidencia y no modifica historia académica

*Para cualquier* llamada al webhook con firma HMAC-SHA256 válida y taller existente, la asistencia se registra exclusivamente en `taller_evidencia`; las tablas `historia_academica` y `cv_usuario` permanecen sin cambios antes y después de la operación.

**Validates: Requirements 9.1, 9.3**

---

### Property 24: Recálculo determinista tras asistencia verificada

*Para cualquier* asistencia verificada que acredita una competencia C, el recálculo del pipeline para todas las afinidades previas que incluyan C produce exactamente el mismo puntaje que produciría una ejecución nueva del pipeline con el perfil actualizado.

**Validates: Requirements 9.2**

---

### Property 25: Webhook rechaza firmas inválidas con auditoría

*Para cualquier* llamada al webhook con firma HMAC-SHA256 incorrecta, el sistema responde con HTTP 401 y registra el intento en `auditoria_webhook` con marca de tiempo, identificador de taller recibido y motivo del rechazo.

**Validates: Requirements 9.4**

---

### Property 26: Descarte y auditoría de Atributos Protegidos

*Para cualquier* entrada a la API que contenga al menos un Atributo Protegido, el sistema descarta todos esos atributos antes de procesar la solicitud y registra en `auditoria_atributos_protegidos` el tipo de atributo, el endpoint y la marca de tiempo ISO 8601 UTC.

**Validates: Requirements 12.2, 12.3**

---

### Property 27: El LLM nunca recibe nombre ni código universitario

*Para cualquier* invocación del LLM, el payload enviado al modelo externo no contiene el nombre del estudiante ni su código universitario; cualquier campo con esa información es eliminado antes de la llamada.

**Validates: Requirements 12.5**

---

### Property 28: Equivalencia API ↔ Servidor MCP para afinidad y puntaje

*Para cualquier* par Perfil-Vacante, la Huella SHA-256 y el Puntaje calculados por la API FastAPI (con su SQL propio) son exactamente iguales a los calculados por el Servidor MCP; el script `verificar_determinismo.py` confirma esta equivalencia.

**Validates: Requirements 10.5**

---

*Content was rephrased for compliance with licensing restrictions where applicable.*
