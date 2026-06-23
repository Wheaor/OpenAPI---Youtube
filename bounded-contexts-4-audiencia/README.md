# Bounded Contexts 4: Audiencia, Comunidad y Engagement

## 1. Descripción de Responsabilidad

Gestionar la interacción social entre los usuarios, creadores y contenidos de la plataforma. Permite a los espectadores suscribirse a canales, reaccionar a videos mediante likes o dislikes, participar en conversaciones a través de comentarios y respuestas, gestionar listas de reproducción personales y mantener historiales de visualización.

Además, administra las publicaciones de comunidad realizadas por los creadores, las notificaciones generadas por actividades relevantes y las métricas de engagement derivadas de la interacción social de los usuarios. Este contexto constituye la principal fuente de señales sociales para los sistemas de recomendación y análisis de comportamiento de la plataforma.

Asimismo, es responsable de mantener las relaciones entre audiencia y creadores, registrar eventos de participación y proveer mecanismos para moderar contenido generado por usuarios, incluyendo reportes, fijación de comentarios y configuración de preferencias de notificaciones.

Su responsabilidad se limita a la gestión de la comunidad y las interacciones sociales. No administra la reproducción de contenido multimedia, la indexación de videos, los algoritmos de recomendación, los derechos editoriales ni la monetización de creadores o anunciantes.

---

## 2. Especificación OpenAPI

El contrato formal de la API con sus endpoints, estructuras de datos (schemas), paginación y gestión de errores se encuentra en el siguiente archivo:

* 📄 Ver Especificación OpenAPI (./openapi_audiencia.yaml)

---

## 3. Diagrama de Clases Conceptual

A continuación se presenta el modelo de datos canónico y las relaciones estrictas para el contexto de Audiencia, Comunidad y Engagement:

```mermaid
classDiagram
    direction TB

    %% --- ENUMERACIONES ---
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

    class EstadoReporte {
        <<enumeration>>
        Pendiente
        Revisado
        Resuelto
        Rechazado
    }

    %% --- DOMINIO DE SUSCRIPCIONES ---
    class Suscripcion {
        - idSuscripcion : UUID
        - idUsuario : UUID
        - idCanal : UUID
        - fechaSuscripcion : DateTime
        + cancelar()
    }

    %% --- DOMINIO DE REACCIONES ---
    class Reaccion {
        - idReaccion : UUID
        - idUsuario : UUID
        - idContenido : UUID
        - tipo : TipoReaccion
        - fechaRegistro : DateTime
    }

    %% --- DOMINIO DE COMENTARIOS ---
    class Comentario {
        - idComentario : UUID
        - idContenido : UUID
        - idAutor : UUID
        - texto : String
        - fechaCreacion : DateTime
        - fijado : Boolean
        + editar()
        + eliminar()
    }

    class RespuestaComentario {
        - idRespuesta : UUID
        - idComentarioPadre : UUID
        - idAutor : UUID
        - texto : String
        - fechaCreacion : DateTime
    }

    class ReporteComentario {
        - idReporte : UUID
        - motivo : String
        - estado : EstadoReporte
        - fechaReporte : DateTime
    }

    %% --- DOMINIO DE COMUNIDAD ---
    class PublicacionComunidad {
        - idPublicacion : UUID
        - idCanal : UUID
        - contenido : String
        - fechaPublicacion : DateTime
    }

    %% --- DOMINIO DE NOTIFICACIONES ---
    class Notificacion {
        - idNotificacion : UUID
        - idUsuario : UUID
        - mensaje : String
        - estado : EstadoNotificacion
        - fechaCreacion : DateTime
        + marcarLeida()
    }

    %% --- DOMINIO DE HISTORIAL ---
    class HistorialVisualizacion {
        - idHistorial : UUID
        - idUsuario : UUID
        - idContenido : UUID
        - fechaVisualizacion : DateTime
    }

    class WatchLater {
        - idRegistro : UUID
        - idUsuario : UUID
        - idContenido : UUID
        - fechaAgregado : DateTime
    }

    %% --- RELACIONES ---
    Comentario "1" o-- "*" RespuestaComentario : contiene

    Comentario "1" o-- "*" ReporteComentario : reportado_por

    PublicacionComunidad "1" o-- "*" Comentario : recibe

    Suscripcion --> Notificacion : genera

    Reaccion --> Comentario : engagement

    Reaccion --> PublicacionComunidad : engagement

    HistorialVisualizacion --> WatchLater : referencia

    Comentario --> Notificacion : genera

    PublicacionComunidad --> Notificacion : genera
```

---

## 4. Diagrama de Secuencia

A continuación se modela el comportamiento dinámico y asíncrono del sistema ante un escenario de participación de la audiencia. El diagrama muestra cómo una interacción social genera engagement y eventos consumidos posteriormente por otros contextos.

```mermaid
sequenceDiagram
    autonumber

    actor User as Espectador
    participant C4 as C4: Audiencia
    participant C3 as C3: Descubrimiento
    participant Creator as Canal/Creador

    %% --- SUSCRIPCIÓN ---
    Note over User,C4: RF-A1 Suscripción a canal

    User->>C4: POST /subscriptions
    C4->>C4: Registra suscripción
    C4-->>User: 201 Created

    %% --- NOTIFICACIÓN ---
    C4->>C4: Genera notificación de nuevas publicaciones

    %% --- COMENTARIO ---
    Note over User,C4: RF-A3 Crear comentario

    User->>C4: POST /comments
    C4->>C4: Almacena comentario
    C4-->>User: 201 Created

    %% --- LIKE ---
    Note over User,C4: RF-A2 Reacción

    User->>C4: POST /reactions
    C4->>C4: Actualiza métricas engagement
    C4-->>User: 201 Created

    %% --- EVENTO HACIA DESCUBRIMIENTO ---
    Note over C4,C3: Integración asíncrona

    C4-)C3: Evento EngagementRegistrado
    C3->>C3: Actualiza ranking del contenido

    %% --- PUBLICACIÓN DE COMUNIDAD ---
    Note over Creator,C4: RF-A6 Publicación comunidad

    Creator->>C4: POST /community-posts
    C4->>C4: Registra publicación

    %% --- NOTIFICACIÓN A SUSCRIPTORES ---
    C4->>C4: Genera notificaciones masivas

    C4-->>Creator: 201 Created

    %% --- CONSULTA DE NOTIFICACIONES ---
    User->>C4: GET /notifications
    C4-->>User: Lista de notificaciones
```

