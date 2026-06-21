# Contrato de API — Contexto 6: Publicidad y Marketplace de Anunciantes

> Documento de referencia para el equipo. Describe qué expone este contexto transaccional, qué eventos emite y cómo consume los identificadores compartidos bajo el estándar RNF-2.

---

## Responsabilidad del contexto

Este servicio gestiona el **negocio publicitario B2B**. Es responsable de administrar las cuentas de anunciantes, el ciclo de vida de sus campañas (presupuestos y targeting), resolver subastas en tiempo real para decidir qué anuncio mostrar, y cobrar por las impresiones efectivas.

**Frontera estricta:** Conoce al video únicamente como un "espacio publicitario" (slot). No maneja archivos de video (eso es Publicación), no define la metadata pública del contenido (eso es Catálogo) y **no le paga directamente a los creadores de contenido** (esa responsabilidad de reparto de ganancias es exclusiva de Monetización).

---

## Identificadores Compartidos (Puntos de Integración)

Para garantizar la consistencia eventual y el cumplimiento del diseño guiado por dominios (DDD), este contexto respeta la nomenclatura global de la plataforma:

| Identificador | Tipo | Origen (Dueño) | Uso en este contexto (Publicidad) |
|---|---|---|---|
| `idItemCatalogo` | UUID | Catálogo (C2) | Puntero para vincular un espacio publicitario con las reglas de Brand Safety y segmentación. |
| `idCanal` | UUID | Catálogo (C2) / Monetización (C5) | Puntero para emitir el evento de ganancias generadas, indicando a qué cuenta de negocio atribuir el dinero. |
| `idUsuario` | UUID | Plataforma (Identidad) | Puntero para cruzar datos demográficos en la subasta (Targeting) sin almacenar la PII del usuario. |
| `idAnunciante` | UUID | **Publicidad (C6)** | Identificador interno generado aquí para la cuenta empresarial que paga por las campañas. |
| `idCampana` | UUID | **Publicidad (C6)** | Identificador interno generado aquí para agrupar presupuesto, creativos y targeting. |

---

## Endpoints expuestos (Consultas y Transacciones Síncronas)

Estos endpoints definen la frontera HTTP del motor de publicidad.

### 1. Integración de Inventario (RF-F5)
*Endpoints consumidos por sistemas internos (Catálogo) para mantener actualizada la elegibilidad de los espacios.*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `PUT` | `/inventario/{idItemCatalogo}/elegibilidad` | Sincroniza si un video es monetizable y su nivel de seguridad (Brand Safety). | Catálogo (C2) |

### 2. Motor Transaccional de Entrega y Medición (RF-F6, RF-F7)
*Endpoints core consumidos por los reproductores (Frontend).*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/subastas/decidir` | Evalúa targeting, presupuesto y pujas. Retorna el comercial ganador y un `AdTransactionToken`. | Player (Frontend) |
| `POST` | `/telemetria/interacciones` | Registra una impresión o click consumiendo el Token. Descuenta el saldo del anunciante. | Player (Frontend) |

### 3. Gestión Comercial B2B (RF-F1, RF-F2, RF-F3, RF-F4)
*Endpoints consumidos por las marcas y agencias a través del panel de anunciantes.*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/anunciantes` | Registra una nueva cuenta empresarial y sus datos de facturación. | Anunciante |
| `POST` | `/campanas` | Crea una campaña con presupuesto máximo y vigencia. | Anunciante |
| `PATCH`| `/campanas/{idCampana}/estado` | Pausa, reanuda o finaliza una campaña. | Anunciante / Sistema |
| `POST` | `/campanas/{idCampana}/creativos` | Sube un recurso publicitario y lo envía a revisión. | Anunciante |
| `PUT`  | `/campanas/{idCampana}/targeting` | Define filtros de audiencia (país, edades, categorías). | Anunciante |

### 4. Facturación y Reportes (RF-F8)
| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `GET`  | `/anunciantes/{idAnunciante}/reportes` | Retorna métricas agregadas (CTR, CPM, Gasto). | Anunciante |

---

## Eventos que este contexto emite (Integración asíncrona)

Las salidas desacopladas de nuestro dominio hacia el ecosistema.

### `ingresoPublicitarioGenerado` ⭐
**Cuándo se emite:** Inmediatamente después de procesar exitosamente un `POST /telemetria/interacciones` de tipo "Impresión" que descontó saldo del anunciante.
**Consumidor principal:** **Monetización (C5)**, quien toma el monto bruto, aplica el Revenue Share y actualiza el saldo del creador.

```json
{
  "idEvento": "550e8400-e29b-41d4-a716-446655440000",
  "idCanal": "123e4567-e89b-12d3-a456-426614174001",
  "idItemCatalogo": "987e6543-e21b-34d2-b876-556677889900",
  "montoTotalPlataforma": 0.015,
  "moneda": "USD",
  "timestamp": "2026-06-20T14:30:00Z"
}
```

### `presupuestoCampanaAgotado`
**Cuándo se emite:** Cuando el saldo restante de la `CampanaPublicitaria` llega a 0.
**Consumidor:** Sistemas internos de notificación por email al anunciante y el propio motor de subastas para invalidar la campaña.

```json
{
  "idEvento": "uuid",
  "idCampana": "uuid",
  "idAnunciante": "uuid",
  "timestamp": "2026-06-20T18:45:00Z"
}
```

---

## Flujo principal de entrega (Secuencia Core)

```text
Frontend (Player)
  │
  ├─ 1. POST /subastas/decidir (idItemCatalogo, datosEspectador)
  │      └─ [Motor cruza campañas activas vs. Brand Safety local]
  │      <─ 200 OK (Retorna Creativo + AdTransactionToken)
  │
  ├─ 2. [Usuario mira el anuncio completo en su dispositivo]
  │
  ├─ 3. POST /telemetria/interacciones (Token, Impresión)
  │      ├─ [Sistema descuenta dinero del Presupuesto de Campaña]
  │      └─ 201 Created
  │
  └─ [Proceso asíncrono en background]
         │
         └─ evento: ingresoPublicitarioGenerado ──────→ MONETIZACIÓN (C5) 
                                                        Recibe dinero para el creador
```

---

## Lo que este contexto NO HACE (Prevención de Acoplamiento)

| Responsabilidad prohibida en este dominio | Bounded Context responsable |
|---|---|
| Pagarle dinero al creador o guardar su saldo. | Monetización del ecosistema creador (C5) |
| Servir el streaming técnico del video principal. | Publicación y distribución (C1) |
| Decidir en qué posición del ranking aparece el video. | Descubrimiento y personalización (C3) |
| Permitir que el espectador deje comentarios o likes. | Audiencia y comunidad (C4) |
| Definir si el video tiene reclamos de Copyright. | Catálogo editorial (C2) |
