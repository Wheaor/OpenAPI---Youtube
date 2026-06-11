# Bounded Contexts 6: Publicidad y Marketplace de Anunciantes

## 1. Descripción de Responsabilidad
Gestionar el negocio publicitario B2B de la plataforma. Permite a cuentas empresariales (Anunciantes) registrar sus datos de facturación, administrar el ciclo de vida de sus Campañas Publicitarias (presupuestos, vigencia y objetivos), subir sus comerciales (Creativos) y definir reglas de segmentación (Targeting). Además, cuenta con un motor de subastas en tiempo real que evalúa las oportunidades de visualización en base a la elegibilidad del inventario (Brand Safety) y calcula las métricas de rendimiento (CTR, CPM) para emitir las facturas (Invoices) correspondientes.

---

## 2. Especificación OpenAPI
El contrato formal de la API con sus endpoints, estructuras de datos (schemas), paginación y gestión de errores se encuentra en el siguiente archivo:
* 📄 [Ver Especificación OpenAPI](./openapi.yaml)

---

## 3. Diagrama de Clases Conceptual
A continuación se presenta el modelo de datos canónico y las relaciones estricta para el contexto publicitario:

```mermaid
classDiagram
    direction TB
    
    %% --- ENUMERADOS (Tipos de datos estrictos) ---
    class EstadoFinanciero {
        <<enumeration>>
        AlDia
        Moroso
        Suspendido
    }
    class EstadoCampana {
        <<enumeration>>
        Creada
        Activa
        Pausada
        Agotada
        Finalizada
    }
    class TipoFormato {
        <<enumeration>>
        PreRoll
        MidRoll
        Bumper
        Display
    }
    class EstadoAprobacion {
        <<enumeration>>
        Pendiente
        Aprobado
        Rechazado
    }
    class EstadoSeguridadMarca {
        <<enumeration>>
        Seguro
        Restringido
        Inapropiado
    }
    class TipoInteraccion {
        <<enumeration>>
        Impresion
        Click
        Skip
    }
    class EstadoPago {
        <<enumeration>>
        Pendiente
        Pagada
        Vencida
    }

    %% --- CLASES DEL MODELO CANÓNICO ---
    class Anunciante {
        -UUID idAnunciante
        -String razonSocial
        -String rutFacturacion
        -String direccionFacturacion
        -String monedaPreferencia
        -String metodoPagoId
        -EstadoFinanciero estadoFinanciero
    }

    class CampanaPublicitaria {
        -UUID idCampana
        -UUID idAnunciante
        -String objetivo
        -Decimal presupuestoTotal
        -Decimal presupuestoRestante
        -DateTime fechaInicio
        -DateTime fechaFin
        -EstadoCampana estado
    }

    class CreativoPublicitario {
        -UUID idCreativo
        -UUID idCampana
        -String urlResource
        -TipoFormato tipoFormato
        -Int duracionSegundos
        -EstadoAprobacion estadoAprobacion
        -String motivoRechazo
    }

    class CriterioTargeting {
        -UUID idTargeting
        -UUID idCampana
        -List~String~ paisesObjetivo
        -Int edadMinima
        -Int edadMaxima
        -List~String~ categoriasContenido
        -List~String~ interesesUsuario
        +estimarInventarioDisponible() Long
    }

    class InventarioContenido {
        -UUID idItemCatalogo
        -UUID idCanal
        -Boolean esMonetizable
        -String categoriaContenido
        -EstadoSeguridadMarca estadoSeguridadMarca
    }

    class OportunidadVisualizacion {
        -UUID idOportunidad
        -UUID idItemCatalogo
        -String paisEspectador
        -Int edadEspectador
        -TipoFormato formatoSolicitado
        +decidirAnuncioMostrar(campanasActivas: List) CreativoPublicitario
    }

    class RegistroInteraccionAd {
        -UUID idInteraccion
        -UUID idCreativo
        -TipoInteraccion tipoInteraccion
        -Decimal costoGatillado
        -DateTime timestamp
    }

    class ReporteRendimiento {
        -UUID idCampana
        -Long impresionesTotales
        -Long clicksTotales
        -Long skipsTotales
        -Decimal gastoAcumulado
        +calcularCTR() Decimal
        +calcularCPM() Decimal
    }

    class FacturaAnunciante {
        -UUID idFactura
        -UUID idAnunciante
        -String periodoCobro
        -Decimal montoTotalFacturado
        -DateTime fechaEmision
        -EstadoPago estadoPago
    }

    %% --- RELACIONES Y ARQUITECTURA DEL SISTEMA ---
    Anunciante "1" o-- "0..*" CampanaPublicitaria : administra
    Anunciante "1" o-- "0..*" FacturaAnunciante : recibe
    CampanaPublicitaria "1" o-- "1..*" CreativoPublicitario : aloja
    CampanaPublicitaria "1" *-- "1" CriterioTargeting : compone
    CampanaPublicitaria "1" <-- "1" ReporteRendimiento : analiza
    CreativoPublicitario "1" <-- "0..*" RegistroInteraccionAd : trackea
    InventarioContenido "1" <-- "0..*" OportunidadVisualizacion : evalua
