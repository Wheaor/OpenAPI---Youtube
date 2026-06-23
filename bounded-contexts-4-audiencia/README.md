# 📌 Bounded Contexts 4: Audiencia, Comunidad y Engagement

## 1. Descripción de Responsabilidad

Gestionar la interacción social entre los usuarios, creadores y contenidos de la plataforma. Permite a los espectadores suscribirse a canales, reaccionar a videos mediante likes o dislikes, participar en conversaciones a través de comentarios y respuestas, gestionar listas de contenido para ver más tarde y mantener historiales de visualización.

Además, administra las publicaciones de comunidad realizadas por los creadores, las notificaciones generadas por actividades relevantes y las métricas de engagement derivadas de la interacción social de los usuarios. Este contexto constituye la principal fuente de señales sociales para los sistemas de recomendación y análisis de comportamiento de la plataforma.

Asimismo, es responsable de mantener las relaciones entre audiencia y creadores, registrar eventos de participación y proveer mecanismos para moderar contenido generado por usuarios, incluyendo reportes, fijación de comentarios y configuración de preferencias de notificaciones.

Su responsabilidad se limita a la gestión de la comunidad y las interacciones sociales. No administra la reproducción de contenido multimedia, la indexación de videos, los algoritmos de recomendación, los derechos editoriales ni la monetización de creadores o anunciantes.

---

## 2. Especificación OpenAPI

El contrato formal de la API con sus endpoints, estructuras de datos (schemas), paginación y gestión de errores se encuentra en el siguiente archivo:

* 📄 Ver Especificación OpenAPI (./openapi_audiencia.yaml)

---

## 3. Diagrama de Clases Conceptual

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

class EstadoReporte {
  <<enumeration>>
  Pendiente
  Revisado
  Resuelto
  Rechazado
}

class Suscripcion {
  - idSuscripcion : UUID
  - idUsuario : UUID
  - idCanal : UUID
  - fechaSuscripcion : DateTime
  + cancelar()
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
  + editar()
  + eliminar()
}

class RespuestaComentario {
  - idRespuesta : UUID
  - idComentarioPadre : UUID
  - idUsuario : UUID
  - texto : String
  - fechaCreacion : DateTime
}

class ReporteComentario {
  - idReporte : UUID
  - motivo : String
  - estado : EstadoReporte
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
  + marcarLeida()
}

class HistorialVisualizacion {
  - idHistorial : UUID
  - idUsuario : UUID
  - idItemCatalogo : UUID
  - fechaVisualizacion : DateTime
}

class VerMasTarde {
  - idRegistro : UUID
  - idUsuario : UUID
  - idItemCatalogo : UUID
  - fechaAgregado : DateTime
}

Comentario "1" o-- "*" RespuestaComentario : contiene
Comentario "1" o-- "*" ReporteComentario : reportado_por
PublicacionComunidad "1" o-- "*" Comentario : recibe
Suscripcion --> Notificacion : genera
Reaccion --> Comentario : engagement
Reaccion --> PublicacionComunidad : engagement
HistorialVisualizacion --> VerMasTarde : referencia
Comentario --> Notificacion : genera
PublicacionComunidad --> Notificacion : genera
