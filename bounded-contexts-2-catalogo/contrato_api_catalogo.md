# Contrato de API — Contexto 2: Catálogo Editorial y Derechos

> Documento de referencia para el equipo. Describe qué expone este contexto, qué eventos emite, qué eventos consume y qué IDs comparte con los demás contextos.

---

## Responsabilidad del contexto

Este servicio gestiona la **identidad pública y editorial del contenido**: desde que un asset técnico está listo en Publicación hasta que ese contenido tiene título, descripción, categoría, visibilidad y reglas de acceso definidas.

**Conoce al video como ítem editorial con metadata, visibilidad y política de derechos. No gestiona el archivo técnico ni su reproducción.** Esa responsabilidad es exclusiva del Contexto de Publicación.

---

## Identificadores que este contexto genera y comparte

Estos IDs son los puntos de integración entre este contexto y los demás. Cuando otro contexto los reciba (vía evento o consulta), debe referenciarlos tal cual — no crear un ID propio para el mismo concepto.

| Identificador | Tipo | Generado en | Usado por |
|---|---|---|---|
| `idCanal` | UUID | Al registrar un canal editorial | Todos los contextos que necesiten agrupar contenido por creador |
| `idItemCatalogo` | UUID | Al crear un ítem de catálogo | Descubrimiento (indexación), Monetización (elegibilidad de ingresos), Publicidad (inventario) |
| `idRestriccion` | UUID | Al aplicar una restricción | Descubrimiento y Publicidad (para ajustar comportamiento de forma inmediata) |
| `idReclamacion` | UUID | Al registrar una reclamación | Sistemas de moderación y auditoría |

> **Para todos los equipos:** el `idCanal` que genera este contexto es el mismo que circula por el sistema. No generen un ID propio de canal en otro contexto.

---

## Eventos que este contexto consume (integración asíncrona entrante)

Este contexto se suscribe al broker para recibir señales de Publicación. No llama directamente a Publicación en ningún flujo normal.

| Evento | Emitido por | Efecto en Catálogo |
|---|---|---|
| `assetListoParaPublicacion` | Contexto de Publicación | Habilita la creación de un `ItemCatalogo` vinculado a ese `idAsset` |
| `transmisionVivoCambioEstado` | Contexto de Publicación | Cuando `estado = Finalizada` e `idAssetGrabacion` está presente, habilita la vinculación del VOD resultante |

---

## Endpoints expuestos (consultas síncronas)

Estos son los endpoints que **otros contextos o el cliente pueden llamar** cuando necesiten consultar o modificar datos editoriales en tiempo real.

### Gestión de canales (RF-C1)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/canales` | Registra un canal editorial asociado a un creador | Cliente del creador |
| `GET` | `/canales/{idCanal}` | Retorna el perfil editorial público del canal | Cliente del espectador, Descubrimiento |
| `PATCH` | `/canales/{idCanal}` | Actualiza nombre, descripción o imagen del canal | Cliente del creador |
| `GET` | `/canales/{idCanal}/contenidos` | Lista paginada de ítems publicados del canal, filtrable por categoría | Cliente del espectador, Descubrimiento |

### Contenido editorial (RF-C2, RF-C3)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/contenidos` | Crea un ítem de catálogo en estado `Borrador` vinculando un `idAsset` | Cliente del creador |
| `GET` | `/contenidos/{idItemCatalogo}` | Retorna la metadata editorial completa del ítem | Cliente del creador, Descubrimiento, Publicidad |
| `PATCH` | `/contenidos/{idItemCatalogo}` | Actualiza metadata editorial parcialmente (título, descripción, tags, categoría, miniatura) | Cliente del creador |
| `PATCH` | `/contenidos/{idItemCatalogo}/estado` | Cambia el estado de visibilidad del ítem (`Publicado`, `Despublicado`) | Cliente del creador |
| `GET` | `/contenidos/{idItemCatalogo}/visibilidad` | Evalúa si el contenido es visible para un espectador dado (`?pais=CL&edad=25`) | Descubrimiento, Publicación, Publicidad |

### Restricciones de política (RF-C6)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `GET` | `/contenidos/{idItemCatalogo}/restricciones` | Retorna restricciones activas e historial completo | Cliente del creador, sistemas de moderación |
| `POST` | `/contenidos/{idItemCatalogo}/restricciones` | Aplica una nueva restricción de política sobre el ítem | Sistema / Moderador |
| `DELETE` | `/contenidos/{idItemCatalogo}/restricciones/{idRestriccion}` | Levanta una restricción activa | Sistema / Moderador |

### Reclamaciones de derechos (RF-C5)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/contenidos/{idItemCatalogo}/reclamaciones` | Registra una reclamación de derechos sobre el ítem | Titular externo / sistema de derechos |
| `GET` | `/contenidos/{idItemCatalogo}/reclamaciones` | Lista todas las reclamaciones del ítem con su estado | Cliente del creador, sistemas de auditoría |
| `PATCH` | `/contenidos/{idItemCatalogo}/reclamaciones/{idReclamacion}` | Resuelve una reclamación (`Aceptada`, `Disputada`, `Retirada`) | Sistema / Moderador |

### Playlists (RF-C4)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/canales/{idCanal}/playlists` | Crea una nueva playlist en el canal | Cliente del creador |
| `GET` | `/canales/{idCanal}/playlists` | Lista todas las playlists del canal | Cliente del creador, cliente del espectador |
| `GET` | `/playlists/{idPlaylist}` | Retorna el detalle de la playlist con sus ítems ordenados | Cliente del espectador, Descubrimiento |
| `PATCH` | `/playlists/{idPlaylist}` | Actualiza título, descripción o visibilidad | Cliente del creador |
| `DELETE` | `/playlists/{idPlaylist}` | Elimina la playlist (no elimina los ítems de catálogo) | Cliente del creador |
| `POST` | `/playlists/{idPlaylist}/items` | Agrega un ítem de catálogo a la playlist | Cliente del creador |
| `DELETE` | `/playlists/{idPlaylist}/items/{idItemCatalogo}` | Quita un ítem de la playlist | Cliente del creador |
| `PUT` | `/playlists/{idPlaylist}/items/orden` | Reordena todos los ítems de la playlist de una vez | Cliente del creador |

---

## Eventos que este contexto emite (integración asíncrona saliente)

Estos eventos se publican en el broker de mensajería. Los contextos interesados deben suscribirse; este contexto no llama directamente a ningún otro.

### `contenidoPublicado`
**Cuándo se emite:** cuando un ítem transiciona al estado `Publicado`.
**Consumidores:** **Descubrimiento** (indexación para búsqueda y feeds), **Monetización** (habilitar elegibilidad de ingresos), **Publicidad** (incorporar al inventario publicitario disponible).

```json
{
  "idEvento": "uuid",
  "idItemCatalogo": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "titulo": "Mi primer video de viaje",
  "categoria": "Viajes",
  "tags": ["viaje", "sudamerica", "mochilero"],
  "restriccionesActivas": [],
  "timestamp": "2026-06-15T22:10:00Z"
}
```

> **Para los equipos de Descubrimiento, Monetización y Publicidad:** el campo `restriccionesActivas` incluye las restricciones vigentes al momento de la publicación. Úsenlo para filtrar correctamente desde el primer momento sin necesidad de una consulta adicional a Catálogo.

---

### `contenidoDespublicado`
**Cuándo se emite:** cuando un ítem pasa a `Despublicado` o `Restringido` (por decisión del creador, reclamación de derechos o política de moderación).
**Consumidores:** **Descubrimiento** (remover de índices y feeds), **Publicidad** (retirar del inventario).

```json
{
  "idEvento": "uuid",
  "idItemCatalogo": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "estadoVisibilidad": "Despublicado",
  "motivo": "DECISION_CREADOR | RECLAMACION_DERECHOS | POLITICA_MODERACION",
  "timestamp": "2026-06-17T12:00:00Z"
}
```

---

### `restriccionAplicada`
**Cuándo se emite:** cuando se aplica una nueva restricción de política sobre un ítem.
**Consumidores:** **Descubrimiento** y **Publicidad** (para excluir el contenido de ciertos territorios o del inventario publicitario de forma inmediata).

```json
{
  "idEvento": "uuid",
  "idItemCatalogo": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "idRestriccion": "uuid",
  "tipoRestriccion": "BloqueGeografico | RestriccionEtaria | LimitacionMonetizacion | SuspensionModeracion",
  "edadMinima": 18,
  "territoriosBloqueados": ["CL", "AR"],
  "timestamp": "2026-06-18T10:00:00Z"
}
```

---

### `restriccionLevantada`
**Cuándo se emite:** cuando una restricción activa es eliminada. Si el ítem queda sin restricciones bloqueantes y vuelve a estado `Publicado`, se emite también `contenidoPublicado`.
**Consumidores:** **Descubrimiento** y **Publicidad**.

```json
{
  "idEvento": "uuid",
  "idItemCatalogo": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "idRestriccion": "uuid",
  "tipoRestriccion": "BloqueGeografico | RestriccionEtaria | LimitacionMonetizacion | SuspensionModeracion",
  "timestamp": "2026-06-19T08:00:00Z"
}
```

---

### `reclamacionCreada`
**Cuándo se emite:** cuando se registra una reclamación de derechos sobre un ítem.
**Consumidores:** sistemas de moderación y auditoría internos.

```json
{
  "idEvento": "uuid",
  "idReclamacion": "uuid",
  "idItemCatalogo": "uuid",
  "idCanal": "uuid",
  "tipoReclamacion": "DerechosAutor | MarcaRegistrada | Privacidad | Otro",
  "timestamp": "2026-06-17T14:00:00Z"
}
```

---

### `reclamacionResuelta`
**Cuándo se emite:** cuando una reclamación es aceptada, disputada o retirada. Puede encadenar los eventos `contenidoDespublicado`, `restriccionAplicada` o `restriccionLevantada`.
**Consumidores:** sistemas de moderación y auditoría internos.

```json
{
  "idEvento": "uuid",
  "idReclamacion": "uuid",
  "idItemCatalogo": "uuid",
  "idCanal": "uuid",
  "resolucion": "Aceptada | Disputada | Retirada",
  "timestamp": "2026-06-18T09:00:00Z"
}
```

---

## Flujo principal de integración: asset listo hasta publicación

```
Contexto Publicación
  │
  └─ evento: assetListoParaPublicacion ──────→ CATÁLOGO recibe idAsset
                                                    │
                                                    ├─ POST /contenidos               → Creador vincula metadata editorial
                                                    │    └─ ItemCatalogo {Borrador}
                                                    │
                                                    ├─ PATCH /contenidos/{id}         → Creador edita metadata (opcional)
                                                    │
                                                    └─ PATCH /contenidos/{id}/estado  → Creador publica
                                                         │  [valida restricciones bloqueantes]
                                                         └─ ItemCatalogo {Publicado}
                                                              │
                                                              ├─ evento: contenidoPublicado ──→ Descubrimiento
                                                              ├─ evento: contenidoPublicado ──→ Monetización
                                                              └─ evento: contenidoPublicado ──→ Publicidad
```

---

## Autenticación

Todas las operaciones requieren JWT en header `Authorization: Bearer <token>`.
El token debe incluir los claims de rol: `Creador`, `Espectador`, `Sistema` o `Moderador` según la operación.

---

## Formato de error (uniforme en toda esta API)

```json
{
  "codigo": "ASSET_NO_DISPONIBLE",
  "mensaje": "El asset indicado no existe o aún no completó su procesamiento técnico.",
  "timestamp": "2026-06-15T22:07:00Z",
  "detalles": ["idAsset: d3b07384-d113-4956-a5db-2d1b331a0f2c no registrado como listo"]
}
```

---

## Lo que este contexto NO hace (para evitar confusión en el diseño)

| Responsabilidad | Contexto correcto |
|---|---|
| Subida, procesamiento y transcoding del video | Publicación y distribución de contenido |
| Servir el manifest de reproducción | Publicación y distribución de contenido |
| Recomendaciones, búsqueda y feeds personalizados | Descubrimiento y personalización |
| Likes, comentarios y notificaciones a suscriptores | Audiencia, comunidad y engagement |
| Pagos y liquidación de ingresos al creador | Monetización del ecosistema creador |
| Campañas publicitarias y targeting | Publicidad y marketplace de anunciantes |
| Autenticación e identidad de usuarios | Sistema de identidad compartido |
