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
    class EstadoReporte {
        <<enumeration>>
        Generando
        Listo
        Expirado
    }
    class EstadoRetiro {
        <<enumeration>>
        Pendiente
        Procesando
        Completado
        Fallido
    }
    class EstadoFiscal {
        <<enumeration>>
        Incompleto
        EnRevision
        Verificado
        Rechazado
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

    %% --- DOMINIO: LIBRO MAYOR CONTABLE (RF-M3) ---
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

    %% --- DOMINIO: ANALÍTICA Y REPORTES (RF-M4) ---
    class ResumenFinancieroPeriodo {
        - idResumen : UUID
        - idCanal : UUID
        - periodoIdentificador : String
        - totalIngresosCreador : Decimal
        - totalIngresosPlataforma : Decimal
        - fechaActualizacion : DateTime
    }

    class DesgloseIngresoVideo {
        - idDesglose : UUID
        - idResumen : UUID
        - idItemCatalogo : UUID
        - gananciaPorPublicidad : Decimal
        - gananciaPorPropinas : Decimal
        - vistasMonetizadasContabilizadas : Long
        - rpmCalculado : Decimal
    }

    class ReporteExportable {
        - idReporte : UUID
        - idCanal : UUID
        - periodoSolicitado : String
        - fechaGeneracion : DateTime
        - formatoDescarga : String
        - urlDescarga : String
        - estado : EstadoReporte
    }

    %% --- DOMINIO: PAGOS Y FISCALIDAD (RF-M5) ---
    class PerfilFiscal {
        - idPerfilFiscal : UUID
        - idCanal : UUID
        - paisResidencia : String
        - tipoDocumento : String
        - identificadorFiscal : String
        - estadoVerificacion : EstadoFiscal
    }

    class MetodoPagoCreador {
        - idMetodoPago : UUID
        - idCanal : UUID
        - tipoMetodo : String
        - tokenPasarelaExterna : String
        - esPrincipal : Boolean
    }

    class SolicitudRetiro {
        - idRetiro : UUID
        - idCanal : UUID
        - idMetodoPago : UUID
        - montoSolicitado : Decimal
        - moneda : String
        - fechaSolicitud : DateTime
        - comprobanteFiscalEmitido : String
        - estado : EstadoRetiro
    }

    %% --- RELACIONES ESTRUCTURALES ---
    CuentaCreador "1" *-- "0..*" SolicitudMonetizacion : registra
    CuentaCreador "1" *-- "0..*" ControlMonetizacionVideo : gestiona
    CuentaCreador "1" *-- "0..*" NivelMembresia : ofrece
    CuentaCreador "1" *-- "0..*" RegistroIngreso : acumula
    CuentaCreador "1" *-- "0..*" ResumenFinancieroPeriodo : consolida
    CuentaCreador "1" *-- "0..*" ReporteExportable : solicita
    
    %% Relaciones RF-M5
    CuentaCreador "1" *-- "0..1" PerfilFiscal : posee
    CuentaCreador "1" *-- "0..*" MetodoPagoCreador : inscribe
    CuentaCreador "1" *-- "0..*" SolicitudRetiro : ejecuta
    
    SolicitudRetiro "0..*" --> "1" MetodoPagoCreador : depositado_en
    ResumenFinancieroPeriodo "1" *-- "0..*" DesgloseIngresoVideo : detalla_por_video
    NivelMembresia "1" <-- "0..*" SuscripcionMembresia : contrata
    ControlMonetizacionVideo "1" <-- "0..*" AportePropina : recibe
