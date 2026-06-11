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
    
    %% --- ENUMERADOS (Declarados, pero sin flechas que ensucien) ---
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

    %% --- CLASES DEL MODELO ---
    class Anunciante {
        - idAnunciante : UUID
        - razonSocial : String
        - rutFacturacion : String
        - direccionFacturacion : String
        - monedaPreferencia : String
        - metodoPagoId : String
        - estadoFinanciero : EstadoFinanciero
    }

    class FacturaAnunciante {
        - idFactura : UUID
        - idAnunciante : UUID
        - periodoCobro : String
        - montoTotalFacturado : Decimal
        - fechaEmision : DateTime
        - estadoPago : EstadoPago
    }

    class CampanaPublicitaria {
        - idCampana : UUID
        - idAnunciante : UUID
        - objetivo : String
        - presupuestoTotal : Decimal
        - presupuestoRestante : Decimal
        - fechaInicio : DateTime
        - fechaFin : DateTime
        - estado : EstadoCampana
    }

    class CriterioTargeting {
        - idTargeting : UUID
        - idCampana : UUID
        - paisesObjetivo : List~String~
        - edadMinima : Int
        - edadMaxima : Int
        - categoriasContenido : List~String~
        - interesesUsuario : List~String~
        + estimarInventarioDisponible() Long
    }

    class ReporteRendimiento {
        - idCampana : UUID
        - impresionesTotales : Long
        - clicksTotales : Long
        - skipsTotales : Long
        - gastoAcumulado : Decimal
        + calcularCTR() Decimal
        + calcularCPM() Decimal
    }

    class CreativoPublicitario {
        - idCreativo : UUID
        - idCampana : UUID
        - urlResource : String
        - tipoFormato : TipoFormato
        - duracionSegundos : Int
        - estadoAprobacion : EstadoAprobacion
        - motivoRechazo : String
    }

    class RegistroInteraccionAd {
        - idInteraccion : UUID
        - idCreativo : UUID
        - tipoInteraccion : TipoInteraccion
        - costoGatillado : Decimal
        - timestamp : DateTime
    }

    class InventarioContenido {
        - idItemCatalogo : UUID
        - idCanal : UUID
        - esMonetizable : Boolean
        - categoriaContenido : String
        - estadoSeguridadMarca : EstadoSeguridadMarca
    }

    class OportunidadVisualizacion {
        - idOportunidad : UUID
        - idItemCatalogo : UUID
        - paisEspectador : String
        - edadEspectador : Int
        - formatoSolicitado : TipoFormato
        + decidirAnuncioMostrar(campanasActivas: List) CreativoPublicitario
    }

    %% --- RELACIONES ESTRUCTURALES DEL NEGOCIO (Árbol limpio) ---
    Anunciante "1" o-- "0..*" CampanaPublicitaria : administra
    Anunciante "1" o-- "0..*" FacturaAnunciante : recibe
    
    CampanaPublicitaria "1" *-- "1" CriterioTargeting : segmenta
    CampanaPublicitaria "1" <-- "1" ReporteRendimiento : genera_metricas
    CampanaPublicitaria "1" o-- "1..*" CreativoPublicitario : contiene
    
    CreativoPublicitario "1" <-- "0..*" RegistroInteraccionAd : mide_rendimiento
    
    InventarioContenido "1" <-- "0..*" OportunidadVisualizacion : habilita_subasta

    %% --- PUENTES DE INTEGRACIÓN INTERNA ---
    OportunidadVisualizacion ..> CampanaPublicitaria : analiza_para_subasta
    OportunidadVisualizacion ..> CreativoPublicitario : selecciona_y_retorna
