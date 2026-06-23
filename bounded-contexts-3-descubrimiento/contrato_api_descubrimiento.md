# Contrato de API — Contexto 3: Descubrimiento y Personalización

> Documento de referencia para el equipo. Describe qué expone este contexto de descubrimiento, cómo consume eventos de otros dominios y qué identificadores compartidos requiere bajo el estándar RNF-2.

---

## Responsabilidad del contexto

Este servicio opera como el **Motor de Descubrimiento, Indexación y Personalización** de la plataforma. Es responsable de indexar el contenido publicado, construir perfiles dinámicos de intereses, generar resultados de búsqueda, calcular tendencias regionales y producir recomendaciones personalizadas para cada espectador.

Utiliza señales de comportamiento provenientes de distintos dominios (impresiones, clics, tiempo de visualización y engagement social) para alimentar algoritmos de ranking y determinar qué contenido debe aparecer primero para cada usuario.

**Frontera estricta:** Conoce la relevancia y popularidad del contenido, pero **no conoce el contenido en sí**. No almacena archivos multimedia, no administra comentarios, no controla campañas publicitarias y no decide si un video puede o no ser publicado.

---

## Identificadores Compartidos (Puntos de Integración)

Para garantizar consistencia entre los modelos de recomendación, indexación y búsqueda, este contexto utiliza exclusivamente identificadores emitidos por otros dominios.

| Identificador        | Tipo | Origen (Dueño)         | Uso en este contexto (Descubrimiento)                                       |
| -------------------- | ---- | ---------------------- | --------------------------------------------------------------------------- |
| idItemCatalogo       | UUID | Catálogo (C2)          | Identifica el contenido indexado y recomendado.                             |
| idCanal              | UUID | Catálogo (C2)          | Identifica el canal asociado al contenido para búsquedas y recomendaciones. |
| idUsuario            | UUID | Plataforma (Identidad) | Identifica al espectador para personalizar feeds y perfiles de interés.     |
| idSesionReproduccion | UUID | Publicación (C1)       | Permite asociar tiempo de visualización a recomendaciones futuras.          |

---

## Endpoints expuestos (Consultas y Captura de Señales)

Estos son los únicos canales de entrada al sistema de descubrimiento.

### 1. Búsqueda e Indexación (RF-D1, RF-D6)

*Endpoints utilizados para localizar contenido y mantener actualizado el índice.*

| Método | Path                            | Qué hace                                         | Quién lo llama |
| ------ | ------------------------------- | ------------------------------------------------ | -------------- |
| GET    | /search/videos                  | Busca videos mediante texto libre y filtros.     | Frontend       |
| GET    | /search/channels                | Busca canales mediante texto libre.              | Frontend       |
| GET    | /search/suggestions             | Obtiene sugerencias mientras el usuario escribe. | Frontend       |
| POST   | /index/content                  | Indexa nuevo contenido publicado.                | Catálogo (C2)  |
| DELETE | /index/content/{idItemCatalogo} | Elimina contenido del índice.                    | Catálogo (C2)  |

### 2. Recomendaciones y Feed Personalizado (RF-D2, RF-D3)

*Endpoints utilizados para construir experiencias personalizadas.*

| Método | Path                                      | Qué hace                     | Quién lo llama |
| ------ | ----------------------------------------- | ---------------------------- | -------------- |
| GET    | /feeds/home/{idUsuario}                   | Genera feed personalizado.   | Frontend       |
| GET    | /recommendations/related/{idItemCatalogo} | Obtiene videos relacionados. | Frontend       |

### 3. Tendencias y Exploración (RF-D4)

*Endpoints utilizados para descubrir contenido popular.*

| Método | Path                            | Qué hace                             | Quién lo llama |
| ------ | ------------------------------- | ------------------------------------ | -------------- |
| GET    | /trending                       | Obtiene contenido trending global.   | Frontend       |
| GET    | /trending/{region}              | Obtiene contenido trending regional. | Frontend       |
| GET    | /explore/categories/{categoria} | Explora contenido por categoría.     | Frontend       |

### 4. Captura de Señales (RF-D5)

*Endpoints utilizados para alimentar el motor de ranking.*

| Método | Path                            | Qué hace                                    | Quién lo llama   |
| ------ | ------------------------------- | ------------------------------------------- | ---------------- |
| POST   | /signals/impressions            | Registra una impresión en feed.             | Frontend         |
| POST   | /signals/clicks                 | Registra un click sobre recomendación.      | Frontend         |
| POST   | /signals/watch-time             | Registra tiempo de visualización consumido. | Publicación (C1) |
| GET    | /profiles/interests/{idUsuario} | Consulta perfil de intereses inferido.      | Frontend         |

---

## Integración por Eventos (Asíncrona)

### Qué consumimos de otros contextos

Descubrimiento depende de eventos externos para mantener actualizado su modelo de ranking.

### De Catálogo Editorial (C2)

* Escuchamos activamente cuando un contenido es publicado, actualizado o restringido para mantener sincronizado el índice de búsqueda.

### De Publicación y Distribución (C1)

* Escuchamos eventos de sesiones de reproducción completadas para actualizar señales de interés y watch time.

### De Audiencia y Engagement (C4)

* Escuchamos señales de engagement social (likes, comentarios, suscripciones) para enriquecer los algoritmos de recomendación.

---

## Qué emitimos hacia otros contextos

Nuestros eventos permiten medir la efectividad del descubrimiento y alimentar sistemas analíticos.

### recomendacionServida

**Cuándo se emite:** Cuando el sistema entrega una recomendación o feed personalizado a un usuario.

**Consumidor principal:** Analítica, Observabilidad y sistemas de experimentación A/B.

```json
{
  "idEvento": "550e8400-e29b-41d4-a716-446655440000",
  "idUsuario": "123e4567-e89b-12d3-a456-426614174000",
  "idItemCatalogo": "123e4567-e89b-12d3-a456-426614174001",
  "tipoFeed": "Home",
  "scoreRanking": 0.92,
  "timestamp": "2026-06-21T09:00:00Z"
}
```

### perfilInteresesActualizado

**Cuándo se emite:** Cuando nuevas señales modifican significativamente el perfil de intereses de un usuario.

**Consumidor principal:** Sistemas de analítica y personalización avanzada.

```json
{
  "idEvento": "550e8400-e29b-41d4-a716-446655440111",
  "idUsuario": "123e4567-e89b-12d3-a456-426614174000",
  "interesesActualizados": [
    "Tecnología",
    "Programación",
    "Inteligencia Artificial"
  ],
  "timestamp": "2026-06-21T10:15:00Z"
}
```

---

## Lo que este contexto NO HACE (Prevención de Acoplamiento)

| Responsabilidad prohibida en este dominio         | Bounded Context al que debes llamar    |
| ------------------------------------------------- | -------------------------------------- |
| Reproducir videos o gestionar streaming.          | Publicación y Distribución (C1)        |
| Guardar títulos, miniaturas o metadata editorial. | Catálogo Editorial y Derechos (C2)     |
| Almacenar comentarios o likes.                    | Audiencia, Comunidad y Engagement (C4) |
| Cobrar membresías o procesar pagos.               | Monetización (C5)                      |
| Gestionar campañas publicitarias.                 | Publicidad y Marketplace (C6)          |
| Decidir si un contenido puede ser publicado.      | Catálogo Editorial y Derechos (C2)     |
| Aplicar restricciones legales o territoriales.    | Catálogo Editorial y Derechos (C2)     |
