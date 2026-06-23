# Bounded Contexts 1: Publicación y Distribución de Contenido

## 1. Descripción de Responsabilidad
Gestionar el ciclo de vida técnico del contenido de video en la plataforma. Permite a los creadores iniciar una Subida de su archivo (completa o por partes), monitoreando su progreso hasta su validación. Una vez finalizada la subida, el sistema dispara automáticamente el Procesamiento del archivo, generando múltiples calidades del mismo Asset (360p, 720p, 1080p) y permitiendo reintentar procesamientos fallidos. Además, administra las Sesiones de Reproducción, entregando la información necesaria para reproducir un asset según el dispositivo, ancho de banda y región del espectador, y capturando métricas técnicas de visualización (duración vista, calidad, interrupciones). El contexto también soporta Transmisiones en Vivo, proveyendo los datos de conexión para emitir desde software externo y permitiendo iniciar, pausar y finalizar la transmisión, con la posibilidad de generar una versión grabada al término del live. Notifica a otros contextos cuando un asset está técnicamente listo para ser publicado, sin tomar ninguna decisión sobre su visibilidad pública ni gestionar su metadata editorial (título, descripción, tags).

---

## 2. Especificación OpenAPI
El contrato formal de la API de este contexto junto con sus endpoints, estructuras de datos (schemas), paginación y gestión de errores, se encuentra en el siguiente archivo:
* 📄 [Ver Especificación OpenAPI](./openapi_publicacion.yaml)

---

## 3. Diagrama de Clases Conceptual 
A continuación se presenta el modelo de datos canónico y las relaciones estrictas para el contexto de Publicación y Distribución de Contenido:

```mermaid
classDiagram
    direction LR

    class Asset {
        -idAsset: UUID
        -idCanal: UUID
        -idSubidaOrigen: UUID
        -duracionSegundos: int
        -estado: EstadoProcesamiento
        -formatosDisponibles: string[] 
        -tamanoBytesOriginal: long
        -fechaCreacion: DateTime
        +iniciarProcesamiento() void
        +obtenerFormatosDisponibles() string[]
        +estaListoParaPublicacion() boolean
        +generarManifest(dispositivo, anchoBanda, region) ManifestReproduccion
    }

    class Subida {
        -idSubida: UUID
        -idCanal: UUID
        -nombreArchivo: string 
        -tamanoBytesTotal: long
        -tipoSubida: TipoSubida
        -bytesRecibidos: long
        -estado: EstadoSubida
        -idAssetResultante :UUID 
        +recibirParte(numeroParte, bytes) void
        +calcularProgreso() float
        +cancelar() void
        +completarYValidar() Asset
    }

    class Procesamiento {
        -idAsset :UUID
        -estado: EstadoProcesamiento
        -calidadesGeneradas :string[]
        -motivoFallo: string
        -intentos: int
        +generarCalidades() void
        +reintentar() void
        +marcarComoFinalizado(exito) void
    }

    class SesionReproduccion {
        -idSesion: UUID
        -idAsset: UUID
        -idUsuario: UUID
        -dispositivo: Dispositivo
        -segundoActual: int
        -calidadEnUso: string
        -interrupciones: int
        -estado: EstadoSesionReproduccion
        +registrarProgreso(segundo, calidad) void
        +finalizar() void
        +calcularSegundosVistos() int
    }

    class ManifestReproduccion {
        <<value object>>
        -idAsset: UUID
        -calidadesDisponibles: string[] 
        -calidadRecomendada: string
        -urlsReproduccion: MapaCalidadUrl 
        +seleccionarCalidadOptima(dispositivo, anchoBanda, region) string
    }

    class TransmisionVivo {
        -idTransmision: UUID
        -idCanal: UUID
        -titulo: string
        -estado: EstadoTransmisionVivo
        -ingestUrl: URI
        -claveStream: string
        -idAssetGrabacion: UUID
        +iniciar() void
        +pausar() void
        +finalizar(generarGrabacion) Asset
    }

    class Canal {
        <<external>>
        +UUID idCanal
    }

    class EstadoSubida {
        <<enumeration>>
        EnProgreso
        Completada
        Cancelada
        Fallida
    }

    class EstadoProcesamiento {
        <<enumeration>>
        Pendiente
        EnCurso
        Completado
        Fallido
    }

    class EstadoTransmisionVivo {
        <<enumeration>>
        Configurada
        EnVivo
        Pausada
        Finalizada
    }

    class EstadoSesionReproduccion {
        <<enumeration>>
        Activa
        Finalizada
    }

    class TipoSubida {
        <<enumeration>>
        Completa
        PorPartes
    }

    class Dispositivo {
        <<enumeration>>
        Movil
        Tablet
        Desktop
        SmartTV
        Consola
    }

    Subida "1" --> "0..1" Asset : origina
    TransmisionVivo "1" --> "0..1" Asset : puede generar
    Asset "1" *-- "1" Procesamiento : tiene
    Asset "1" --> "0..*" SesionReproduccion : es reproducido en
    Asset ..> ManifestReproduccion : genera

    Asset "*" --> "1" Canal : pertenece a
    Subida "*" --> "1" Canal : pertenece a
    TransmisionVivo "*" --> "1" Canal : pertenece a

    Subida ..> EstadoSubida : usa
    Subida ..> TipoSubida : usa
    Asset ..> EstadoProcesamiento : usa
    Procesamiento ..> EstadoProcesamiento : usa
    TransmisionVivo ..> EstadoTransmisionVivo : usa
    SesionReproduccion ..> EstadoSesionReproduccion : usa
    SesionReproduccion ..> Dispositivo : usa

```

---

## 4. Diagrama de Secuencia

A continuación se modela el comportamiento dinámico del sistema durante el escenario integrador exigido por la cátedra, en la porción donde Publicación y Distribución de Contenido (Contexto 1) actúa como protagonista. El diagrama detalla cómo Publicación valida de forma síncrona la visibilidad del contenido junto a Catálogo editorial y derechos (Contexto 2) antes de servir el manifest de reproducción, gestiona el ciclo completo de la sesión de playback, y finalmente notifica de manera asíncrona y desacoplada a Descubrimiento y personalización (Contexto 3) las métricas de visualización (watch time, calidad, interrupciones), sin que Publicación dependa de la disponibilidad ni del procesamiento de ese contexto consumidor.
