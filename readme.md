# **Algoritmo de Popularidad con Gravedad por Atenuación Temporal (APGAT)**
![brain](https://github.com/user-attachments/assets/fa134f37-552e-4b9c-8497-83be67958c68)
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

- 1.1 - Agrega su `userId` al array `positiveStance` o `negativeStance`.
- 1.2 - Incrementa `likesPositiveCount` o `likesNegativeCount` en engagement.
- 1.3 - Incrementa `recentInteractionsCount`.
- 1.4 - Actualiza `lastInteractionTimestamp`.

---

#### 🔸 **Comentario**

- 2.1 - Crea documento de comentario en colección aparte.
- 2.2 - Incrementa `commentsCount` en engagement.
- 2.3 - Incrementa `recentInteractionsCount`.
- 2.4 - Actualiza `lastInteractionTimestamp`.

---

#### 🔸 **Vista**

- 3.1 - Incrementa `viewsCount` en engagement.
- 3.2 - Incrementa `recentInteractionsCount`.
- 3.3 - Actualiza `lastInteractionTimestamp`.

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

### 🔸 **Ejemplo de Scheduled Cloud Function v2 (resumido)**

```js
/* eslint-disable */
const { onSchedule } = require("firebase-functions/v2/scheduler"); // Import v2 scheduler
const admin = require("firebase-admin");

admin.initializeApp();
const db = admin.firestore();

// v2 function definition
exports.updatePopularityScoresAlgorithmV2 = onSchedule({
    schedule: "every 15 minutes",
    memory: "512MiB", // Note: The unit is now MiB for v2
    timeoutSeconds: 300,
  }, async (event) => {
    console.log("Starting scheduled popularity calculation task (v2)...");

    // Algorithm parameters (NO CHANGE HERE)
    const LIKE_WEIGHT = 1.0;
    const DISLIKE_WEIGHT = 1.0;
    const COMMENT_WEIGHT = 2.5;
    const VIEW_WEIGHT = 0.1;
    const BASE_SCORE = 1.0;
    const GRAVITY = 1.8;

    const fifteenMinutesAgo = new Date(Date.now() - 15 * 60 * 1000);

    const query = db
      .collection("posts")
      .where("lastInteractionTimestamp", ">", fifteenMinutesAgo);

    const activePostsSnapshot = await query.get();

    if (activePostsSnapshot.empty) {
      console.log("No active posts to update. Task completed.");
      return null;
    }

    console.log(`Found ${activePostsSnapshot.size} active posts to process.`);

    const batch = db.batch();

    // The entire logic of your function remains IDENTICAL
    activePostsSnapshot.forEach((doc) => {
      const post = doc.data();
      const engagement = post.engagement || {};

      const interactionScore =
        (engagement.likesPositiveCount || 0) * LIKE_WEIGHT +
        (engagement.likesNegativeCount || 0) * DISLIKE_WEIGHT +
        (engagement.commentsCount || 0) * COMMENT_WEIGHT +
        (engagement.viewsCount || 0) * VIEW_WEIGHT +
        BASE_SCORE;

      if (!post.timestamp) {
        console.log(`Post missing timestamp: ${doc.id}`);
        return;
      }

      const publicationTimestamp = post.timestamp.toDate();
      const ageInHours =
        (Date.now() - publicationTimestamp.getTime()) / 3600000;

      const gravityDenominator = Math.pow(ageInHours + 2, GRAVITY);
      const popularityScore = interactionScore / gravityDenominator;

      const recentInteractions = engagement.recentInteractionsCount || 0;
      const velocityMultiplier = Math.log10(1 + recentInteractions);

      const finalScore = popularityScore * (1 + velocityMultiplier);

      batch.update(doc.ref, {
        popularityScore: finalScore,
        "engagement.recentInteractionsCount": 0,
      });
    });

    try {
      await batch.commit();
      console.log(
        `Successfully updated popularity scores for ${activePostsSnapshot.size} posts.`
      );
    } catch (error) {
      console.error("Error executing batch update:", error);
    }

    return null;
  }
);
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

## ⚖️ **9. Nivel de Dificultad**

### 🔧 **Nivel estimado: [ Moderado-Avanzado ]**

### ✅ **Razones principales**

1. **Cloud Functions programadas**
   - Requiere conocimiento de funciones `onSchedule` en Firebase, deploy con permisos adecuados y gestión de costos.

2. **Optimización de Firestore**
   - Necesita estructurar la base de datos con índices para consultar posts activos eficientemente.

3. **Diseño correcto de modelos**
   - Creación y manejo de `PostModel` y `EngagementModel` con campos de engagement y popularidad sincronizados.

4. **Cálculos algorítmicos precisos**
   - Uso de exponenciales, logaritmos y divisiones por tiempo, evitando errores de redondeo o división por cero.


---

## 🔬 **Dificultad general (backend tradicional)**

- **Nivel: Medio**
  - Lógica clara, pero requiere cron jobs bien implementados y queries eficientes para bases de datos grandes.

---

## 💡 **Conclusión**

> No es un algoritmo complicado en su lógica, pero su **implementación completa en Flutter + Firebase exige nivel intermedio-avanzado** en arquitectura de datos, funciones serverless y eficiencia de costos.

---

### 👥 **Recomendación de equipo**

| Rol sugerido | Justificación |
| ------------ | ------------- |
| **Mobile Developer (Flutter)** | Para integración de la UI con Firestore y estado. |
| **Backend Developer / Cloud Engineer** | Para la creación y optimización de Cloud Functions programadas y estructura de base de datos. |

---
## 🔄 **10. Diagrama de flujo completo del algoritmo**

![diagrama_de_algoritmo_de_popularidad](https://github.com/user-attachments/assets/c28c1c71-f292-4806-bc66-b59fae7ebf23)


---




