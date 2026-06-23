# Contrato de API — Contexto 4: Audiencia, Comunidad y Engagement

> Documento de referencia para el equipo. Describe qué expone este contexto social, cómo consume eventos de otros dominios y qué identificadores compartidos requiere bajo el estándar RNF-2.

---

## Responsabilidad del contexto

Este servicio opera como el **motor de interacción social y generación de señales de engagement** de la plataforma. Es responsable de gestionar las relaciones entre usuarios, canales y contenido a través de suscripciones, reacciones, comentarios, historial de visualización y publicaciones de comunidad.

También es responsable de generar y emitir eventos sociales que representan el comportamiento de los usuarios, los cuales son consumidos por el contexto de Descubrimiento para personalización, ranking y análisis de comportamiento.

**Frontera estricta:** Este contexto conoce la interacción social, pero **no conoce el ranking, no recomienda contenido, no procesa video y no gestiona monetización ni pagos**.

---

## Identificadores Compartidos (Puntos de Integración)

Este contexto depende de identificadores globales definidos por otros dominios, pero mantiene su propio modelo interno.

| Identificador | Tipo | Origen (Dueño) | Uso en este contexto (Audiencia) |
|---|---|---|---|
| `idUsuario` | UUID | Plataforma (Identidad) | Identifica al usuario que interactúa (comenta, reacciona, se suscribe). |
| `idCanal` | UUID | Catálogo (C2) | Identifica el canal al que el usuario se suscribe o del cual recibe contenido social. |
| `idItemCatalogo` | UUID | Catálogo (C2) | Identifica el contenido sobre el cual ocurren reacciones, comentarios o engagement. |
| `idComentario` | UUID | Audiencia (C4) | Identificador interno de comentarios dentro del contexto social. |

---

## Endpoints expuestos (Operaciones síncronas)

### 1. Suscripciones (RF-A1)

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/subscriptions` | Suscribirse a un canal | App Usuario |
| `GET` | `/subscriptions/{idUsuario}` | Listar canales suscritos | App Usuario |
| `DELETE` | `/subscriptions/{idSuscripcion}` | Cancelar suscripción | App Usuario |

### 2. Reacciones (RF-A2)

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/reactions` | Crear o actualizar like/dislike | App Usuario |
| `DELETE` | `/reactions/{idUsuario}/{idItemCatalogo}` | Eliminar reacción | App Usuario |
| `GET` | `/reactions/{idItemCatalogo}/stats` | Obtener estadísticas de reacciones | App / Catálogo |

### 3. Comentarios (RF-A3)

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/comments` | Crear comentario | App Usuario |
| `GET` | `/comments` | Listar comentarios por ítem | App Usuario |
| `PUT` | `/comments/{idComentario}` | Editar comentario | Autor |
| `DELETE` | `/comments/{idComentario}` | Eliminar comentario | Autor |
| `POST` | `/comments/{idComentario}/report` | Reportar comentario | Usuario |

### 4. Comunidad (RF-A6)

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/community-posts` | Crear post de comunidad | Creador |
| `GET` | `/community-posts/{idCanal}` | Listar posts de canal | App Usuario |

### 5. Notificaciones e Historial (RF-A4 / RF-A5)

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `GET` | `/notifications/{idUsuario}` | Obtener notificaciones | App Usuario |
| `GET` | `/history/{idUsuario}` | Obtener historial de visualización | App Usuario |

---

## Integración por Eventos (Asíncrona)

### Qué consumimos de otros contextos:

- Desde Publicación (C1):
  - SesionVisualizacionCompletadaEvent
  - ContenidoPublicadoEvent

- Desde Catálogo (C2):
  - ContenidoPublicadoEvent
  - ContenidoDespublicadoEvent

### Qué emitimos hacia otros contextos:

#### UsuarioSuscritoEvent
Emitido cuando un usuario se suscribe a un canal.

#### UsuarioDesuscritoEvent
Emitido cuando un usuario deja de seguir un canal.

#### ReaccionRegistradaEvent
Emitido cuando un usuario da like o dislike a un ítem.

#### ComentarioPublicadoEvent
Emitido cuando un comentario es creado exitosamente.

#### ComentarioEliminadoEvent
Emitido cuando un comentario es eliminado.

#### EngagementRegistradoEvent
Evento agregado que consolida señales para consumo por Descubrimiento.

### Ejemplo de evento

```json
{
  "idEvento": "a12c4d90-11ab-4c3d-9a01-9d2b8f4c1111",
  "idUsuario": "e4e0a812-c224-4b93-84b2-3c22119d8541",
  "idItemCatalogo": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "tipo": "Like",
  "timestamp": "2026-06-23T12:00:00Z"
}
```

---

## Lo que este contexto NO HACE (Prevención de Acoplamiento)

| Responsabilidad prohibida | Bounded Context responsable |
|---|---|
| Recomendar videos o construir feeds | Descubrimiento (C3) |
| Reproducir o transmitir video | Publicación (C1) |
| Definir metadata editorial | Catálogo (C2) |
| Procesar pagos o ingresos | Monetización (C5) |
| Gestionar campañas publicitarias | Publicidad (C6) |
| Almacenar archivos de video o streaming | Publicación (C1) |

---

## Cierre conceptual

Este contexto representa la capa social del ecosistema, donde se generan todas las señales de interacción del usuario. Estas señales alimentan los sistemas de recomendación y análisis, pero no contienen lógica de negocio editorial ni financiera.
