# Bounded Contexts 4: Audiencia, Comunidad y Engagement

## 1. Descripción de Responsabilidad

Gestionar la interacción social entre los usuarios, creadores y los ítems del catálogo de la plataforma. Permite a los espectadores suscribirse a canales, reaccionar a ítems de contenido mediante likes o dislikes, participar en conversaciones a través de comentarios y respuestas, y administrar la relación social con los creadores.

Además, administra las publicaciones de comunidad realizadas por los creadores, las notificaciones generadas por eventos relevantes y las señales de engagement derivadas de la interacción de los usuarios con el contenido.

Este contexto constituye la principal fuente de eventos sociales del sistema, los cuales son consumidos por el contexto de Descubrimiento para personalización y ranking.

Asimismo, es responsable de registrar y emitir eventos de interacción como suscripciones, comentarios, reacciones y engagement, los cuales son utilizados por otros contextos para análisis y recomendación.

Su responsabilidad se limita a la gestión de la capa social y de interacción. No administra la reproducción de video, la indexación de contenido, los algoritmos de recomendación, los derechos editoriales ni la monetización.

---

## 2. Especificación OpenAPI

El contrato formal de la API con sus endpoints, estructuras de datos y eventos de dominio se encuentra en:

📄 Ver Especificación OpenAPI (./openapi_audiencia.yml)

---

## 3. Modelo de Dominio (Clases)

```mermaid
classDiagram
    direction TB

    class TipoReaccion {
        <<enumeration>>
        Like
        Dislike
    }

    class EstadoNotificacion {
        <<enumeration>>
        NoLeida
        Leida
    }

    class Suscripcion {
        - idSuscripcion : UUID
        - idUsuario : UUID
        - idCanal : UUID
        - fechaSuscripcion : DateTime
    }

    class Reaccion {
        - idReaccion : UUID
        - idUsuario : UUID
        - idItemCatalogo : UUID
        - tipo : TipoReaccion
        - fechaRegistro : DateTime
    }

    class Comentario {
        - idComentario : UUID
        - idItemCatalogo : UUID
        - idUsuario : UUID
        - texto : String
        - fechaCreacion : DateTime
        - fijado : Boolean
    }

    class ReporteComentario {
        - idReporte : UUID
        - motivo : String
        - estado : String
        - fechaReporte : DateTime
    }

    class PublicacionComunidad {
        - idPublicacion : UUID
        - idCanal : UUID
        - contenido : String
        - fechaPublicacion : DateTime
    }

    class Notificacion {
        - idNotificacion : UUID
        - idUsuario : UUID
        - mensaje : String
        - estado : EstadoNotificacion
        - fechaCreacion : DateTime
    }

    class HistorialVisualizacion {
        - idHistorial : UUID
        - idUsuario : UUID
        - idItemCatalogo : UUID
        - fechaVisualizacion : DateTime
    }

    class WatchLater {
        - idRegistro : UUID
        - idUsuario : UUID
        - idItemCatalogo : UUID
        - fechaAgregado : DateTime
    }

    Comentario "1" o-- "*" ReporteComentario
    PublicacionComunidad "1" o-- "*" Comentario
    Suscripcion --> Notificacion
    Comentario --> Notificacion
    PublicacionComunidad --> Notificacion
```

---

## 4. Diagrama de Secuencia (Flujo Integrador)

```mermaid
sequenceDiagram
    autonumber

    actor User as Espectador
    participant C4 as Audiencia, Comunidad y Engagement
    participant C3 as Descubrimiento
    participant C1 as Publicación
    participant Creator as Creador

    User->>C4: POST /subscriptions
    C4->>C4: Registrar suscripción
    C4-->>User: 201 Created
    C4-->>C3: UsuarioSuscritoEvent

    User->>C4: POST /reactions
    C4->>C4: Registrar reacción
    C4-->>User: 201 Created
    C4-->>C3: ReaccionRegistradaEvent

    User->>C4: Watch / engagement
    C4->>C4: Registrar engagement
    C4-->>C3: EngagementRegistradoEvent

    User->>C4: POST /comments
    C4->>C4: Guardar comentario
    C4-->>User: 201 Created
    C4-->>C3: ComentarioPublicadoEvent

    Creator->>C4: POST /community-posts
    C4->>C4: Crear publicación
    C4-->>Creator: 201 Created
```

