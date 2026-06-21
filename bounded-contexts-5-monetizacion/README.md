# Bounded Contexts 5: Monetización del Ecosistema Creador

## 1. Descripción de Responsabilidad
Gestionar el modelo de negocio, recaudación y distribución de ingresos de los creadores de contenido de la plataforma de manera desacoplada. Permite administrar el proceso de postulación y evaluación de requisitos de elegibilidad de los canales para el programa de socios (RF-M1), configurar y procesar productos de apoyo directo del espectador como membresías pagadas de canal y propinas por video (RF-M2), y registrar de forma centralizada los flujos brutos de ingresos provenientes tanto de publicidad como de transacciones directas (RF-M3). Además, se encarga de aplicar las reglas de división de ganancias de la plataforma (Revenue Share) (RF-M3), consolidar paneles de analítica financiera avanzada desglosados por video y periodos de tiempo (RPM) (RF-M4), y gestionar las cuentas contables internas de saldo acumulado para procesar las solicitudes de retiro (Payouts) bajo la documentación fiscal correspondiente (RF-M5).

---

```mermaid
classDiagram
    direction TB

    %% --- ENUMERADOS ---
    class EstadoElegibilidad {
        <<enumeration>>
        Pendiente
        Apto
        Rechazado
        Suspendido
    }
    class TipoOrigenIngreso {
        <<enumeration>>
        Publicidad
        Membresia
        Propina
    }
    class EstadoIngreso {
        <<enumeration>>
        Pendiente
        Confirmado
        Pagado
    }

    %% --- DOMINIO: CUENTA Y ELEGIBILIDAD (RF-M1) ---
    class CuentaCreador {
        - idCanal : UUID
        - estadoElegibilidad : EstadoElegibilidad
        - monetizacionActivaGlobal : Boolean
        - saldoDisponible : Decimal
        - saldoPendiente : Decimal
    }

    class SolicitudMonetizacion {
        - idSolicitud : UUID
        - idCanal : UUID
        - fechaPostulacion : DateTime
        - instantSuscriptores : Int
        - instantHorasVisualizacion : Double
        - estadoResultado : EstadoElegibilidad
        - motivoRechazo : String
    }

    class ControlMonetizacionVideo {
        - idItemCatalogo : UUID
        - idCanal : UUID
        - monetizacionAnunciosHabilitada : Boolean
        - propinasHabilitadas : Boolean
    }

    %% --- DOMINIO: PRODUCTOS DEL CREADOR (RF-M2) ---
    class NivelMembresia {
        - idNivel : UUID
        - idCanal : UUID
        - nombreNivel : String
        - precioMensual : Decimal
        - beneficios : String
        - activo : Boolean
    }

    class SuscripcionMembresia {
        - idSuscripcion : UUID
        - idUsuario : UUID
        - idNivel : UUID
        - fechaInicio : DateTime
        - fechaProximoCobro : DateTime
        - estadoActiva : Boolean
    }

    class AportePropina {
        - idAporte : UUID
        - idUsuario : UUID
        - idItemCatalogo : UUID
        - montoPago : Decimal
        - moneda : String
        - fechaTransaccion : DateTime
        - mensajeAdjunto : String
    }

    %% --- DOMINIO: LIBRO MAYOR CONTABLE Y REVENUE SHARE (RF-M3) ---
    class RegistroIngreso {
        - idRegistro : UUID
        - idCanal : UUID
        - idItemCatalogo : UUID
        - origenIngreso : TipoOrigenIngreso
        - montoBruto : Decimal
        - reglaRepartoAplicada : String
        - montoCreador : Decimal
        - montoPlataforma : Decimal
        - fechaRegistro : DateTime
        - estado : EstadoIngreso
    }

    %% --- RELACIONES ESTRUCTURALES ---
    CuentaCreador "1" *-- "0..*" SolicitudMonetizacion : registra
    CuentaCreador "1" *-- "0..*" ControlMonetizacionVideo : gestiona
    CuentaCreador "1" *-- "0..*" NivelMembresia : ofrece
    CuentaCreador "1" *-- "0..*" RegistroIngreso : acumula

    NivelMembresia "1" <-- "0..*" SuscripcionMembresia : contrata
    ControlMonetizacionVideo "1" <-- "0..*" AportePropina : recibe
