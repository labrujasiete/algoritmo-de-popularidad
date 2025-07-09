# **Algoritmo de Popularidad con Gravedad por Atenuación Temporal**

El “Algoritmo de Popularidad con Gravedad por Atenuación Temporal” calcula la relevancia de cada noticia combinando sus interacciones (likes, dislikes, comentarios y vistas) con su antigüedad, de forma que las noticias más nuevas y activas se mantengan arriba, mientras que las viejas pierden visibilidad si no siguen recibiendo interacciones.

## 📈 **Reporte Técnico Completo**

**Fecha:** 2025-07-08\
**Autor:** Cristian J. Delgadillo C.

---

## ✅ **1. Objetivo del algoritmo**

Diseñar un sistema de ranking para publicaciones de noticias en el cual:

- La **popularidad** refleje tanto el interés actual como histórico.
- El sistema sea **escalable, económico y rápido**.
- Las publicaciones viejas decaigan a menos que sigan recibiendo interacciones nuevas (**gravedad temporal**).

---

## 🧠 **2. Principios clave**

### 🔷 **A. Puntos de Interacción (fuerza ascendente)**

| Tipo de interacción | Peso asignado |
| ------------------- | ------------- |
| **Comentario**      | 2.5           |
| **Like positivo**   | 1.0           |
| **Like negativo**   | 1.0           |
| **Vista**           | 0.1           |

✅ **Notas:**

- Likes positivos y negativos suman igual, ya que ambos representan reacciones relevantes.

---

### 🔷 **B. Gravedad del tiempo (fuerza descendente)**

La puntuación decae con el tiempo según la fórmula:

```math
Puntuación = \frac{PuntosDeInteracción}{(EdadEnHoras + 2)^{Gravedad}}
```

- **Edad en horas** = tiempo transcurrido desde publicación.
- **+2** evita divisiones por cero y suaviza la curva inicial.
- **Gravedad (exponente)** = 1.8 (configurable).

---

### 🔷 **C. Bono de velocidad (virilidad repentina)**

Si una publicación recibe muchas interacciones recientes, su score se multiplica por:

```math
MultiplicadorDeVelocidad = log_{10}(1 + InteraccionesRecientes)
```

- Genera rendimientos decrecientes para evitar inflar demasiado el puntaje.

---

## 🗃️ **3. Estructura de datos final**

### 🔸 **Modelo de Post (Firestore)**

```dart
class PostModel {
  final String id;
  final String publicPostId;
  final String ownerId;
  final String title;
  final List<String> tagsIds;
  final String description;
  final Timestamp timestamp;
  final List<Map<String, dynamic>> files;
  final String country;
  final String? state;
  final String? city;

  // Likes y dislikes (usuarios que reaccionaron)
  final List<String> positiveStance;
  final List<String> negativeStance;

  // Engagement: contadores
  final EngagementModel engagement;

  // Score calculado
  final double popularityScore;

  // Timestamp de última interacción
  final Timestamp? lastInteractionTimestamp;
}
```

---

### 🔸 **Modelo Engagement (subcampo en Post)**

```dart
class EngagementModel {
  final int likesPositiveCount;
  final int likesNegativeCount;
  final int commentsCount;
  final int viewsCount;
  final int recentInteractionsCount;
}
```


---

## ⚙️ **4. Flujo de actualización**

### 🔷 **A. Cuando un usuario interactúa**

#### 🔸 **Like**

- Agrega su `userId` al array `positiveStance` o `negativeStance`.
- Incrementa `likesPositiveCount` o `likesNegativeCount` en engagement.
- Incrementa `recentInteractionsCount`.
- Actualiza `lastInteractionTimestamp`.

---

#### 🔸 **Comentario**

- Crea documento de comentario en colección aparte.
- Incrementa `commentsCount` en engagement.
- Incrementa `recentInteractionsCount`.
- Actualiza `lastInteractionTimestamp`.

---

#### 🔸 **Vista**

- Incrementa `viewsCount` en engagement.
- Incrementa `recentInteractionsCount`.
- Actualiza `lastInteractionTimestamp`.

---

### 🔷 **B. Scheduled Function (**``**)**

Se ejecuta **cada 10-15 minutos** (ajustable) para:

1. Consultar posts con `lastInteractionTimestamp` recientes.
2. Calcular **popularityScore** aplicando:
   - Suma ponderada de interacciones.
   - División por la gravedad temporal.
   - Multiplicación por el bono de velocidad.
3. Actualizar `popularityScore` en Firestore.
4. Reiniciar `recentInteractionsCount` a cero.

---

### 🔸 **Ejemplo de Scheduled Function (resumido)**

```js
exports.updatePopularityScores = functions.pubsub
  .schedule("every 10 minutes")
  .onRun(async (context) => {
    const posts = await db.collection("posts")
      .where("lastInteractionTimestamp", ">", cutoffDate)
      .get();

    const batch = db.batch();
    posts.forEach(doc => {
      const data = doc.data();
      const engagement = data.engagement;

      const interactionScore =
        engagement.likesPositiveCount * 1 +
        engagement.likesNegativeCount * 1 +
        engagement.commentsCount * 2.5 +
        engagement.viewsCount * 0.1 +
        1; // BASE_SCORE

      const ageInHours = (Date.now() - data.timestamp.toDate().getTime()) / 3600000;
      const gravity = Math.pow(ageInHours + 2, 1.8);
      const popularityScore = interactionScore / gravity;

      const velocityMultiplier = Math.log10(1 + engagement.recentInteractionsCount);
      const finalScore = popularityScore * (1 + velocityMultiplier);

      batch.update(doc.ref, {
        popularityScore: finalScore,
        "engagement.recentInteractionsCount": 0
      });
    });

    await batch.commit();
  });
```

---

## 🔒 **5. Consideraciones de seguridad**

- Usa **Firebase Security Rules** para:
  - Evitar múltiples likes/dislikes de un mismo usuario (si lo deseas).
  - Proteger la edición directa de campos calculados por backend.

---

## 💰 **6. Costos y optimización**

| Componente            | Frecuencia             | Costo estimado                       |
| --------------------- | ---------------------- | ------------------------------------ |
| Updates de engagement | Cada like/view/comment | Muy bajo (1 escritura)               |
| Scheduled Function    | Cada 10-15 mins        | Bajo si procesas pocos posts activos |

✅ **Consejo:** inicia con cada **15 mins** y monitorea en producción.

---

## ✨ **7. Posibles mejoras futuras**

1. **Dynamic scheduling:** diferente frecuencia según horario pico o tráfico.
2. **Cold start optimization:** mantener warm con Cloud Scheduler pings si usas Node.
3. **Machine Learning:** para personalizar el ranking según el historial de cada usuario.

---

## 📝 **8. Reproducción en cualquier entorno**

Para migrar este sistema:

1. Configura Firestore con la estructura de `posts` y subcolección de comentarios.
2. Implementa las Cloud Functions en Node.js (o adaptadas a GCP Cloud Scheduler).
3. Usa Flutter, React, etc. para interactuar con Firestore siguiendo la lógica de engagement.
4. Ajusta pesos y gravedad según comportamiento real de tus usuarios.

---
## 🔄 **9. Diagrama de flujo completo del algoritmo**

Aquí tienes un diseño en markdown para documentación con pasos claros y ordenados:

```
Inicio
  │
  ▼
Usuario realiza una interacción (like, dislike, comentario, vista)
  │
  ▼
Flutter actualiza engagement en Firestore:
  - Incrementa contador correspondiente
  - Actualiza recentInteractionsCount
  - Actualiza lastInteractionTimestamp
  │
  ▼
(En paralelo, si es comentario: crea documento en colección de comentarios)
  │
  ▼
Scheduler Cloud Function se activa cada X minutos
  │
  ▼
Consulta posts con lastInteractionTimestamp recientes
  │
  ▼
Para cada post:
  ├─ Calcula interactionScore (suma ponderada de interacciones + base)
  ├─ Calcula ageInHours
  ├─ Calcula gravedad
  ├─ Calcula popularityScore = interactionScore / gravedad
  ├─ Calcula velocityMultiplier = log10(1 + recentInteractionsCount)
  ├─ Calcula finalScore = popularityScore * (1 + velocityMultiplier)
  ├─ Actualiza popularityScore en Firestore
  └─ Reinicia recentInteractionsCount a 0
  │
  ▼
Fin
```

---




