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
    direction TB

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

    

    Subida "1" --> "0..1" Asset : origina
    TransmisionVivo "1" --> "0..1" Asset : puede generar
    Asset "1" *-- "1" Procesamiento : tiene
    Asset "1" --> "0..*" SesionReproduccion : es reproducido en
    Asset ..> ManifestReproduccion : genera

    Asset "*" --> "1" Canal : pertenece a
    Subida "*" --> "1" Canal : pertenece a
    TransmisionVivo "*" --> "1" Canal : pertenece a

    

```

---

## 4. Diagrama de Secuencia

A continuación se modela el comportamiento dinámico del sistema durante el escenario integrador exigido por la cátedra, en la porción donde Publicación y Distribución de Contenido (Contexto 1) actúa como protagonista. El diagrama detalla cómo Publicación valida de forma síncrona la visibilidad del contenido junto a Catálogo editorial y derechos (Contexto 2) antes de servir el manifest de reproducción, gestiona el ciclo completo de la sesión de playback, y finalmente notifica de manera asíncrona y desacoplada a Descubrimiento y personalización (Contexto 3) las métricas de visualización (watch time, calidad, interrupciones), sin que Publicación dependa de la disponibilidad ni del procesamiento de ese contexto consumidor.

```mermaid
%%{init: {"theme": "dark"}}%%
sequenceDiagram
    autonumber
    actor Viewer as Viewer (Espectador)
    participant Pub as Publicacion API
    participant Asset as Asset / SesionReproduccion
    participant Cat as Catalogo API
    participant Bus as Bus de Eventos
    participant Desc as Descubrimiento API

    rect rgb(30, 45, 70)
    Note over Viewer, Cat: Paso 1 - Validar visibilidad (consulta sincrona, RF-C3)
    Viewer->>Pub: GET /assets/idAsset/reproduccion/manifest (dispositivo, anchoBanda, region)
    Pub->>Cat: GET /catalogo/idAsset/visibilidad (idUsuario, pais, edad)
    alt contenido visible
        Cat-->>Pub: visible = true
    else contenido no visible o despublicado
        Cat-->>Pub: visible = false (motivo)
        Pub-->>Viewer: 403 contenido no disponible
    end
    end

    rect rgb(30, 60, 45)
    Note over Pub, Asset: Paso 2 - Generar manifest de reproduccion (RF-P3)
    Pub->>Asset: estaListoParaPublicacion()
    Asset-->>Pub: true
    Pub->>Asset: generarManifest(dispositivo, anchoBanda, region)
    Asset->>Asset: seleccionarCalidadOptima(dispositivo, anchoBanda, region)
    Asset-->>Pub: ManifestReproduccion (calidades, urls)
    Pub-->>Viewer: 200 OK (manifest de reproduccion)
    end

    rect rgb(65, 60, 30)
    Note over Viewer, Asset: Paso 3 - Sesion de reproduccion (RF-P3)
    Viewer->>Pub: POST /sesiones-reproduccion (idAsset, idUsuario, dispositivo)
    Pub->>Asset: crear SesionReproduccion()
    Asset-->>Pub: idSesion
    Pub-->>Viewer: 201 Created (idSesion)

    loop Mientras dura la reproduccion
        Viewer->>Pub: PATCH /sesiones-reproduccion/idSesion (segundoActual, calidadEnUso)
        Pub->>Asset: registrarProgreso(segundo, calidad)
    end

    Viewer->>Pub: POST /sesiones-reproduccion/idSesion/finalizar
    Pub->>Asset: finalizar()
    Asset->>Asset: calcularSegundosVistos()
    Asset-->>Pub: segundosVistos, interrupciones
    Pub-->>Viewer: 200 OK (sesion finalizada)
    end

    rect rgb(55, 35, 65)
    Note over Pub, Desc: Paso 4 - Notificacion asincrona (RNF-4, RNF-5)
    Pub->>Bus: publicar evento SesionReproduccionCompletada (idAsset, idUsuario, segundosVistos, calidadEnUso, interrupciones)
    Bus-->>Pub: ack de publicacion
    Bus->>Desc: entregar evento SesionReproduccionCompletada
    Note right of Desc: Consumo eventual, no bloqueante.
    end
```
