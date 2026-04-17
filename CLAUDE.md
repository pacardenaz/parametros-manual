# Base de Conocimiento OBT — KontrolTravel / Ideas Fractal

Repositorio del manual de parámetros de KontrolTravel OBT publicado como sitio GitHub Pages, con pipeline de generación de base de conocimiento para Dify RAG.

## Estructura del repositorio

```
parametros_site/
├── params/           # 368 páginas HTML — una por parámetro
├── docs/             # 10 páginas HTML — manuales técnicos completos
├── media/            # Imágenes y recursos del sitio
├── index.html        # Índice principal del sitio
├── index.csv         # Catálogo de parámetros (código, título, módulo, URL)
├── parametros_dify.txt       # (obsoleto) catálogo plano de parámetros
├── base_conocimiento_obt.txt # ← ARCHIVO PRINCIPAL para Dify RAG
└── base_conocimiento_qa.txt  # (no usado — Q&A requiere LLM en Dify)
```

## Sitio publicado

URL base: `https://pacardenaz.github.io/parametros-manual`

- Parámetros: `/params/{nombre}.html`
- Documentación: `/docs/{nombre}.html`

## Scripts de generación (en la sesión de trabajo)

### `build_knowledge_base.py` ← el más importante

Genera `base_conocimiento_obt.txt` extrayendo contenido completo desde los HTML fuente.

```bash
python3 build_knowledge_base.py
```

**Qué hace:**
- Parsea las 368 páginas de `params/*.html` con `ParamPageExtractor`
- Parsea las 10 páginas de `docs/*.html` con `DocsPageExtractor`
- Clasifica parámetros por módulo (hoteles, vuelos, autos, asistencias, corporativo, markup, udids, general)
- Genera 460 chunks separados por `\n\n`, sin dobles saltos internos
- Cada chunk de documentación incluye: `Documento:`, `Tema:` (alias semántico), `URL documentación:` y el contenido completo
- Cada chunk de parámetro incluye: `Módulo:`, `Código:`, `Título:`, `Descripción:`, `URL documentación:`

**Salida:** `parametros_site/base_conocimiento_obt.txt` (~461k chars, 460 chunks)

### `build_qa.py`

Genera `base_conocimiento_qa.txt` en formato Q&A a partir de `base_conocimiento_obt.txt`. No se usa actualmente porque Dify requiere LLM para indexar Q&A y esa opción está deshabilitada.

### `consolidate_dify.py`

Script anterior que consolidaba `parametros_dify.txt` + `docs/docs_dify.txt`. Reemplazado por `build_knowledge_base.py`.

## Configuración de Dify RAG

### Base de conocimiento activa
- Archivo: `base_conocimiento_obt.txt`
- Modo de índice: **Alta calidad**

### Configuración de fragmentos (crítica)
| Parámetro | Valor |
|-----------|-------|
| Identificador de segmento | `\n\n` |
| Longitud máxima | 3,000 chars |
| Superposición | 100 chars |
| **Reemplazar espacios/saltos consecutivos** | **☐ DESACTIVADO** (crítico) |
| Eliminar URLs | ☐ desactivado |

> ⚠️ Si "Reemplazar espacios, saltos de línea y tabulaciones consecutivas" queda **activado**, destruye los separadores `\n\n` y el archivo completo queda como un solo bloque sin chunks. El chatbot dejará de encontrar resultados.

### Configuración de recuperación
| Parámetro | Valor |
|-----------|-------|
| Modo | Búsqueda híbrida (semántica + keyword) |
| TopK | 8 |
| Umbral de puntuación | Desactivado |

## Flujo para actualizar el conocimiento

Cuando se agrega o modifica un parámetro o documento en el sitio:

1. Actualizar/agregar el HTML en `params/` o `docs/`
2. Hacer push al repositorio (GitHub Pages se actualiza automáticamente)
3. Ejecutar `python3 build_knowledge_base.py`
4. Subir el nuevo `base_conocimiento_obt.txt` a Dify:
   - Eliminar la base de conocimiento actual
   - Crear una nueva con el archivo actualizado
   - Aplicar la configuración de fragmentos descrita arriba
   - Esperar que indexe (~1-2 min)

## Módulos de parámetros

| Módulo | Descripción |
|--------|-------------|
| `hoteles` | Búsqueda, disponibilidad, precios y reserva de hoteles |
| `vuelos` | Búsqueda aérea, GDS (Amadeus/Sabre/Galileo), tarifas, equipaje, cabinas, emisión |
| `autos` | Búsqueda y reserva de autos / renta de vehículos |
| `asistencias` | Tarjetas de asistencia al viajero y seguros |
| `corporativo` | Flujo de aprobación, políticas de viaje, centros de costo (CECO), herencia entre entidades |
| `markup` | Configuración de markup y tarifación |
| `udids` | Campos gerenciales personalizados |
| `general` | Parámetros generales del sistema |

## Documentos técnicos incluidos

| Archivo | Contenido |
|---------|-----------|
| `02_flujocorporativokt.html` | Flujo Corporativo KT — aprobaciones, políticas, CECOs (12 secciones) |
| `01_configuracion-jeeves.html` | Tarjetas de crédito virtuales Jeeves (16 secciones) |
| `03_manual-ventana-de-precios-1-1.html` | Ventana de precios / vuelos económicos (49 secciones) |
| `04_manual-de-udids-1.html` | UDIDs — campos gerenciales (4 secciones) |
| `05_reemisión-reservas.html` | Reemisión de reservas (2 secciones) |
| `06_configuración-del-markup-por-el-gestor.html` | Markup por gestor (6 secciones) |
| `07_configuración-de-hoteles.html` | Configuración módulo hoteles (7 secciones) |
| `08_configuración-de-tarjetas-de-aisstencia..html` | Asistencias / seguros de viaje (8 secciones) |
| `09_configuración-de-autos..html` | Configuración módulo autos (8 secciones) |
| `10_split-latam.html` | Split LATAM — pago dividido (6 secciones) |

## Decisiones técnicas importantes

**¿Por qué no usar `\n\n` dentro de los chunks?**
Dify usa `\n\n` como separador de fragmentos. Cualquier doble salto de línea dentro de una sección hace que el encabezado (`Documento:`, `Tema:`, `URL:`) quede en un chunk separado del contenido. Eso produce chunks de ~284 chars sin información útil que ganan la búsqueda semántica pero no responden la pregunta.

**¿Por qué el campo `Tema:`?**
Los títulos técnicos (`FlujoCorporativoKT`) no coinciden en el espacio de embeddings con consultas naturales ("flujo corporativo"). El campo `Tema:` agrega el alias en lenguaje natural dentro del mismo chunk para mejorar la recuperación semántica.

**¿Por qué no usar el modo Q&A de Dify?**
Dify requiere una llamada LLM por chunk para extraer preguntas en modo Q&A. En esta instancia esa opción está deshabilitada. El archivo `base_conocimiento_qa.txt` existe pero no se usa.
