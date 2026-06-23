# Proyecto Final: Diseño de Software - YouTube Bounded Contexts
* **Institución:** Universidad de O'Higgins (UOH)
* **Profesor:** Edgardo Ortiz
* **Integrantes:** Marcelo Godoy - Eduardo Olivos - Sebastián Pérez


```mermaid
sequenceDiagram
    autonumber
    actor App as Espectador (Frontend)
    participant C3 as C3: Descubrimiento
    participant C2 as C2: Catálogo
    participant C6 as C6: Publicidad
    participant C1 as C1: Publicación
    participant C4 as C4: Audiencia
    participant C5 as C5: Monetización

    Note over App, C5: ESCENARIO END-TO-END DE LA PLATAFORMA

    %% 1. Obtener recomendación
    App->>C3: GET /feeds/home/{idUsuario}
    C3-->>App: 200 OK (Retorna array de idItemCatalogo)

    %% 2. Validar visibilidad
    App->>C2: GET /contenidos/{idItemCatalogo}/visibilidad
    C2-->>App: 200 OK (visible: true)

    %% 3. Elegir anuncio
    App->>C6: POST /subastas/decidir
    C6-->>App: 200 OK (Retorna CreativoPublicitario ganador)

    %% 4. Reproducir video
    App->>C1: POST /sesiones-reproduccion
    C1-->>App: 201 Created (Retorna Manifest de streaming)

    %% 5. Registrar Like
    App->>C4: POST /reactions (tipo: Like)
    C4-->>App: 201 Created
    
    %% Evento asíncrono social
    C4-)C3: Webhook: engagementGenerado (Señal de ranking)

    %% 6. Registrar watch time
    App->>C1: PATCH /sesiones-reproduccion/{idSesion} (finalizada: true)
    C1-->>App: 200 OK
    C1-)C3: Webhook: sesion-completada (Watch Time)
    C1-)C4: Webhook: sesion-completada (Historial personal)

    %% 7. Atribuir ingreso al creador
    Note over C6, C5: Atribución financiera B2B -> B2C
    C6-)C5: Webhook: ingresoPublicitarioGenerado
    C5->>C5: Aplica Revenue Share (Regla contable interna)
```

### Justificación del Escenario Integrador
El flujo demuestra la estricta separación de responsabilidades a través del *Shared Kernel* (utilizando `idItemCatalogo` y `idUsuario` como referencias globales). Ningún servicio resuelve todo; la plataforma se comporta como un sistema distribuido y reactivo:

1. **Obtener recomendación:** El cliente contacta a **Descubrimiento (C3)** para obtener su feed personalizado. C3 retorna solo referencias, no el archivo de video.
2. **Validar que el contenido es visible:** Antes de reproducir, el cliente consulta a **Catálogo (C2)** para asegurar que el video no ha sido restringido por edad o geolocalización en los últimos milisegundos.
3. **Elegir anuncio:** Se ejecuta la subasta en tiempo real en **Publicidad (C6)**.
4. **Reproducir video:** El cliente establece la sesión de streaming con **Publicación (C1)**, el único contexto que maneja el ancho de banda y los bytes del archivo.
5. **Registrar Like:** La interacción social ocurre en **Audiencia (C4)**. C4 emite un evento asíncrono para que C3 actualice sus algoritmos de recomendación en segundo plano.
6. **Registrar watch time:** Al finalizar la reproducción, **Publicación (C1)** dispara un webhook asíncrono. **Descubrimiento (C3)** lo captura para medir la retención, y **Audiencia (C4)** lo captura para armar el historial del usuario.
7. **Atribuir ingreso al creador:** Una vez servido el comercial, **Publicidad (C6)** emite el evento financiero asíncrono hacia **Monetización (C5)**, quien aplica el *Revenue Share* sin acoplar las bases de datos de anunciantes y creadores.
