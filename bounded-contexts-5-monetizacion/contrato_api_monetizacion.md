# Contrato de API — Contexto 5: Monetización del Ecosistema Creador

> Documento de referencia para el equipo. Describe qué expone este contexto financiero, cómo consume los eventos de otros dominios y qué identificadores compartidos requiere bajo el estándar RNF-2.

---

## Responsabilidad del contexto

Este servicio opera como el **Libro Mayor Contable (Ledger) y motor de cumplimiento (Compliance)** de la plataforma. Es responsable de evaluar si un canal cumple los requisitos para ganar dinero, procesar los pagos directos de los espectadores (propinas/membresías), calcular el *Revenue Share* de los ingresos publicitarios y gestionar la salida física del dinero hacia los bancos de los creadores garantizando que cumplan con sus obligaciones fiscales.

**Frontera estricta:** Conoce al dinero y a las cuentas, pero **no conoce el contenido en sí**. No sabe de qué trata un video, no almacena resoluciones de video, no gestiona campañas de anunciantes y no decide en qué orden aparecen los videos en el inicio.

---

## Identificadores Compartidos (Puntos de Integración)

Para que el dinero fluya sin errores de consistencia, este contexto es sumamente estricto con los IDs que recibe. Si otro contexto envía un ID inventado, la transacción será rechazada automáticamente.

| Identificador | Tipo | Origen (Dueño) | Uso en este contexto (Monetización) |
|---|---|---|---|
| `idCanal` | UUID | Catálogo (C2) | Puntero a la cuenta de negocio del creador. Es la "cuenta bancaria" interna donde se acumula el `saldoDisponible`. |
| `idItemCatalogo` | UUID | Catálogo (C2) | Puntero al video publicado. Se requiere para calcular métricas de rendimiento por video (RPM) y para saber a qué video se le dio una propina. |
| `idUsuario` | UUID | Plataforma (Identidad)| Puntero al espectador que está ingresando su tarjeta de crédito para realizar un pago directo (B2C). |

---

## Endpoints expuestos (Consultas y Transacciones Síncronas)

Estos son los únicos canales de entrada al sistema financiero.

### 1. Ingesta B2B Interna (RF-M3)
*Endpoints consumidos exclusivamente por otros microservicios de la plataforma.*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/webhooks/ingresos-publicidad` | Recibe el aviso de que un anuncio generó dinero. Aplica el Revenue Share internamente y capitaliza al creador. | **Publicidad (C6)** |

### 2. Transacciones Directas B2C (RF-M2)
*Endpoints consumidos por la aplicación del espectador cuando decide apoyar económicamente.*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `POST` | `/propinas` | Inicia un cobro en la pasarela externa (Stripe/Transbank) usando la tarjeta del espectador y genera un recibo a favor del creador. | Player (Frontend) |

### 3. Analítica y Retiros (RF-M4, RF-M5)
*Endpoints consumidos por el creador en su panel de administración (Dashboard).*

| Método | Path | Qué hace | Quién lo llama |
|---|---|---|---|
| `GET`  | `/canales/{idCanal}/resumen-financiero` | Consulta analítica de alta velocidad (OLAP). Retorna ganancias agregadas y el RPM de los videos. | Dashboard Creador |
| `POST` | `/canales/{idCanal}/retiros` | Inicia una orden SWIFT al banco real del creador. Falla con `403` si falta el RUT o ID fiscal. | Dashboard Creador |

---

## Integración por Eventos (Asíncrona)

### Qué consumimos de otros contextos:
Monetización se suscribe a las salidas de los demás para accionar lógica financiera.

1. **De Publicidad (C6):**
   * Escuchamos activamente las notificaciones de ingresos (mediante el webhook documentado arriba) para calcular el corte del creador vs. el de la plataforma.

### Qué emitimos hacia otros contextos:
Nuestros eventos informan sobre cambios en el estado comercial.

#### `estadoElegibilidadActualizado`
**Cuándo se emite:** Cuando un creador es aceptado o expulsado del programa de socios (RF-M1) tras una revisión de Compliance.
**Consumidor principal:** **Catálogo (C2)** y **App Frontend**. Catálogo necesita saber esto para habilitar visualmente los botones de "Unirse" (Membresías) o "Dar Propina" en la interfaz del video.

```json
{
  "idEvento": "550e8400-e29b-41d4-a716-446655440000",
  "idCanal": "123e4567-e89b-12d3-a456-426614174001",
  "estadoElegibilidad": "Apto",
  "motivoRechazo": null,
  "timestamp": "2026-06-21T09:00:00Z"
}
```

---

## Lo que este contexto NO HACE (Prevención de Acoplamiento)

Si tu equipo necesita algo de esta lista, **no llamen a Monetización**.

| Responsabilidad prohibida en este dominio | Bounded Context al que debes llamar |
|---|---|
| Contar las "visualizaciones" de un video. | Descubrimiento (C3) / Publicación (C1) |
| Saber si un video tiene reclamos de Copyright. | Catálogo editorial y derechos (C2) |
| Cobrarle dinero a una marca o anunciante. | Publicidad y Marketplace (C6) |
| Guardar el título, descripción o miniatura. | Catálogo editorial y derechos (C2) |
| Suspender la cuenta de un usuario por insultar en chat. | Audiencia, comunidad y engagement (C4) |
