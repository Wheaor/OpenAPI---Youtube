# Contrato de API — Contexto 1: Publicación y Distribución de Contenido

> Documento de referencia para el equipo. Describe qué expone este contexto, qué eventos emite y qué IDs comparte con los demás contextos.

---

## Responsabilidad del contexto

Este servicio gestiona el **ciclo de vida técnico del video**: desde que un creador inicia la subida de un archivo hasta que ese archivo está procesado y disponible para reproducción o transmisión en vivo.

**Conoce al video como archivo o stream. No decide su visibilidad pública ni gestiona su metadata editorial (título, descripción, tags).** Esa responsabilidad es exclusiva del Contexto de Catálogo.

---

## Identificadores que este contexto genera y comparte

Estos IDs son los puntos de integración entre este contexto y los demás. Cuando otro contexto los reciba (vía evento o consulta), debe referenciarlos tal cual — no crear un ID propio para el mismo concepto.

| Identificador | Tipo | Generado en | Usado por |
|---|---|---|---|
| `idAsset` | UUID | Al completar una subida | Catálogo (para vincular metadata editorial), Publicidad (para verificar elegibilidad) |
| `idCanal` | UUID | Recibido del sistema de identidad compartido | Todos los contextos |
| `idTransmision` | UUID | Al configurar un live | Catálogo (para vincular grabación resultante), Audiencia (para notificar suscriptores) |

---

## Endpoints expuestos (consultas síncronas)

Estos son los endpoints que **otros contextos pueden llamar** cuando necesiten validar o consultar datos de este servicio en tiempo real.

### Gestión de subidas (RF-P1)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/subidas` | Inicia una subida. Retorna `idSubida` y URL(s) de destino | Cliente del creador |
| `PUT` | `/subidas/{idSubida}/partes` | Envía un fragmento binario (subida por partes) | Cliente del creador |
| `GET` | `/subidas/{idSubida}` | Consulta estado y progreso de la subida | Cliente del creador |
| `DELETE` | `/subidas/{idSubida}` | Cancela una subida en curso | Cliente del creador |
| `POST` | `/subidas/{idSubida}/completar` | Valida la subida y dispara el procesamiento automáticamente | Cliente del creador |

### Procesamiento técnico (RF-P2)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `GET` | `/assets/{idAsset}/procesamiento` | Estado del transcoding: `Pendiente`, `EnCurso`, `Completado`, `Fallido` | Cliente del creador, monitoreo interno |
| `POST` | `/assets/{idAsset}/procesamiento/reintentar` | Reencola un procesamiento en estado `Fallido` | Cliente del creador |

### Reproducción (RF-P3)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `GET` | `/assets/{idAsset}/reproduccion/manifest` | Retorna URLs de streaming por calidad, adaptadas a dispositivo y región | Player del espectador |
| `POST` | `/sesiones-reproduccion` | Registra el inicio de una sesión de playback | Player del espectador |
| `PATCH` | `/sesiones-reproduccion/{idSesion}` | Actualiza progreso o marca la sesión como finalizada | Player del espectador |

### Transmisión en vivo (RF-P4)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `POST` | `/transmisiones-vivo` | Configura un live. Retorna `ingestUrl` y `claveStream` para OBS | Cliente del creador |
| `PATCH` | `/transmisiones-vivo/{idTransmision}/estado` | Controla el ciclo: `Configurada → EnVivo → Pausada → Finalizada` | Cliente del creador |

### Assets técnicos (RF-P5)

| Método | Path | Qué hace | Quién lo usaría |
|---|---|---|---|
| `GET` | `/creadores/{idCanal}/assets` | Lista paginada de assets de un creador, filtrables por estado | Cliente del creador, Catálogo (verificación) |
| `GET` | `/assets/{idAsset}` | Detalle técnico: duración, formatos disponibles, estado | Catálogo, Publicidad, cualquier contexto que necesite verificar disponibilidad |

---

## Eventos que este contexto emite (integración asíncrona)

Estos eventos se publican en el broker de mensajería. Los contextos interesados deben suscribirse; este contexto no llama directamente a ningún otro.

### `contenidoSubidoCompletamente`
**Cuándo se emite:** al validar y completar una subida de archivo.
**Principalmente interno.** El evento relevante para otros contextos es `assetListoParaPublicacion`.

```json
{
  "idEvento": "uuid",
  "idSubida": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "timestamp": "2026-06-15T21:50:00Z"
}
```

---

### `procesamientoFinalizado`
**Cuándo se emite:** al terminar el transcoding de un asset, con éxito o fallo.
**Consumidores:** monitoreo interno, sistemas de alertas.

```json
{
  "idEvento": "uuid",
  "idAsset": "uuid",
  "resultado": "Exito | Fallo",
  "calidadesGeneradas": ["360p", "720p", "1080p"],
  "motivoFallo": null,
  "timestamp": "2026-06-15T22:05:00Z"
}
```

---

### `assetListoParaPublicacion` 
**Cuándo se emite:** cuando el procesamiento finaliza con éxito. **Este es el evento más importante para Catálogo.**
**Consumidores:** **Catálogo** (para poder vincular metadata editorial y eventualmente publicar el contenido).

```json
{
  "idEvento": "uuid",
  "idAsset": "uuid",
  "idCanal": "uuid",
  "duracionSegundos": 842,
  "calidadesGeneradas": ["360p", "720p", "1080p"],
  "timestamp": "2026-06-15T22:06:00Z"
}
```

> **Para el equipo de Catálogo:** el campo `idAsset` de este evento es el `videoAssetId` que deberán usar al vincular un ítem editorial con su asset técnico (RF-C2). No generen un ID propio para el mismo concepto.

---

### `sesionReproduccionCompletada`
**Cuándo se emite:** cuando una sesión de playback se marca como finalizada.
**Consumidores:** **Descubrimiento** (como señal de watch time), **Audiencia** (para registrar historial de visualización).

```json
{
  "idEvento": "uuid",
  "idSesion": "uuid",
  "idAsset": "uuid",
  "idUsuario": "uuid",
  "segundosVistos": 780,
  "duracionSegundos": 842,
  "calidadPredominante": "1080p",
  "interrupciones": 2,
  "timestamp": "2026-06-15T22:20:00Z"
}
```

---

### `transmisionVivoCambioEstado`
**Cuándo se emite:** cuando un live pasa a `EnVivo` o `Finalizada`.
**Consumidores:** **Audiencia** (para notificar suscriptores que el live comenzó), **Catálogo** (para conocer el `idAssetGrabacion` cuando el live termina y se genera VOD).

```json
{
  "idEvento": "uuid",
  "idTransmision": "uuid",
  "idCanal": "uuid",
  "estado": "EnVivo | Finalizada",
  "idAssetGrabacion": null,
  "timestamp": "2026-06-15T20:00:00Z"
}
```

> **Para el equipo de Catálogo:** cuando `estado` es `Finalizada` e `idAssetGrabacion` está presente, ese ID es el `videoAssetId` del VOD generado. Pueden vincularlo con metadata editorial igual que cualquier otro asset.

---

## Flujo principal de integración: subida hasta publicación

```
Creador
  │
  ├─ POST /subidas                         → Inicia subida, obtiene idSubida
  ├─ PUT  /subidas/{id}/partes             → Envía fragmentos binarios
  ├─ POST /subidas/{id}/completar          → Valida y dispara procesamiento
  │
  └─ [interno: transcoding automático]
       │
       ├─ evento: contenidoSubidoCompletamente    → (interno)
       ├─ evento: procesamientoFinalizado          → (monitoreo)
       └─ evento: assetListoParaPublicacion  ──────→ CATÁLOGO recibe idAsset
                                                         y puede crear ítem editorial
```

---

## Autenticación

Todas las operaciones requieren JWT en header `Authorization: Bearer <token>`.
El token debe incluir los claims de rol: `Creador`, `Espectador` o `Sistema` según la operación.

---

## Formato de error (uniforme en toda esta API)

```json
{
  "codigo": "SUBIDA_INCOMPLETA",
  "mensaje": "No se han recibido todas las partes declaradas para esta subida.",
  "timestamp": "2026-06-15T21:57:08Z",
  "detalles": ["Parte 3 de 5 no recibida"]
}
```

---

## Lo que este contexto NO hace (para evitar confusión en el diseño)

| Responsabilidad | Contexto correcto |
|---|---|
| Título, descripción, tags del video | Catálogo editorial y derechos |
| Decidir si el video es público o privado | Catálogo editorial y derechos |
| Recomendaciones, búsqueda | Descubrimiento y personalización |
| Likes, comentarios | Audiencia, comunidad y engagement |
| Pagos al creador | Monetización del ecosistema creador |
| Campañas publicitarias | Publicidad y marketplace de anunciantes |
