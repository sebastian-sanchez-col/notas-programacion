# Roadmap de React Native: De Cero a Semi-Senior

## Fase 0: Fundamentos web (4-8 semanas, omite lo que ya sepas)
* **JavaScript moderno:** closures, async/await, promesas, event loop, módulos, destructuring.
* **TypeScript:** hoy es el estándar. Tipos, generics, utility types, narrowing, tipado de props y de respuestas de API.
* **React:** hooks (`useState`, `useEffect`, `useMemo`, `useCallback`, `useRef`, `useReducer`), composición, Context, reglas de renderizado, y las novedades de React 19.
* **Git y GitHub:** ramas, PRs, commits claros.
* **Hito:** Una app web pequeña en React + TypeScript.

---

## Fase 1: React Native base con Expo (6-8 semanas)
*Hoy la documentación oficial recomienda Expo como punto de partida, así que empieza ahí.*

* **Componentes core:** `View`, `Text`, `FlatList`/`FlashList`, `Pressable`, `ScrollView`, `Image`.
* **Estilos y layout:** Flexbox, StyleSheet, diseño responsive, safe areas, teclado, orientación.
* **Navegación con Expo Router** (file-based, sobre React Navigation): stacks, tabs, modales, rutas dinámicas, deep links.
* **Diferencias iOS vs Android:** sombras, permisos, back button, status bar.
* **Herramientas:** Expo Go, development builds, simuladores/emuladores, React Native DevTools.
* **Hito:** App de 3-4 pantallas con navegación y listas.

---

## Fase 2: Datos, estado y backend (6-8 semanas)
* **Networking:** `fetch`/Axios, manejo de errores y loading states.
* **Server state:** TanStack Query (cache, invalidación, paginación infinita, optimistic updates).
* **Client state:** Zustand (o Redux Toolkit si el mercado objetivo lo pide). Entiende cuándo usar cada tipo de estado.
* **Formularios:** React Hook Form + Zod.
* **Autenticación:** JWT/OAuth, refresh tokens, almacenamiento seguro (`expo-secure-store`).
* **Persistencia local:** MMKV, AsyncStorage, SQLite (`expo-sqlite`) o WatermelonDB.
* **Backend:** Aprende uno bien (Supabase, Firebase, o tu propia API en Node/NestJS). Es muy valorado entender el otro lado.
* **Offline-first básico:** cola de acciones, sincronización, manejo de conectividad.
* **Hito:** App con login, CRUD real y funcionamiento parcial offline.

---

## Fase 3: Nivel intermedio-avanzado (8-10 semanas)
*Aquí se separa el junior del semi sr.*

* **Animaciones:** Reanimated + Gesture Handler. Entiende el worklet y el UI thread.
* **Rendimiento:** re-renders innecesarios, memoización, virtualización, imágenes optimizadas, Hermes, profiling (React DevTools Profiler, Flipper alternativas actuales, Xcode Instruments, Android Studio Profiler).
* **Nueva arquitectura:** Fabric, TurboModules, JSI, Bridgeless. Entiende cómo funciona y por qué cambió. Verifica en la documentación oficial el estado actual, porque evoluciona rápido.
* **Código nativo:** config plugins de Expo, Expo Modules API, y leer/modificar código Swift/Kotlin básico. Escribir un módulo nativo simple es un diferenciador fuerte.
* **Funcionalidades de dispositivo:** cámara, ubicación, notificaciones push, biometría, permisos, background tasks, deep/universal links.
* **Accesibilidad:** labels, roles, dynamic type, contraste, VoiceOver/TalkBack.
* **Internacionalización:** i18n, formatos de fecha/moneda, RTL.

---

## Fase 4: Calidad, arquitectura y producción (6-8 semanas)
* **Testing:** Jest + React Native Testing Library (unit/integration), Maestro o Detox (E2E), mocks de API con MSW.
* **Arquitectura:** estructura por features, separación de capas, inyección de dependencias ligera, principios SOLID aplicados a RN, monorepos (Turborepo/Nx) con paquetes compartidos.
* **CI/CD:** GitHub Actions + EAS Build / EAS Submit / EAS Update (OTA). Entornos dev/staging/prod, variables de entorno, versionado.
* **Observabilidad:** Sentry (crashes), analytics (PostHog/Firebase), feature flags, logging.
* **Seguridad móvil:** no guardar secretos en el bundle, certificate pinning, ofuscación, almacenamiento seguro, OWASP Mobile Top 10.
* **Publicación real:** cuenta de Apple Developer y Google Play, certificados, provisioning, TestFlight, tracks de Play Console, revisión de tiendas, políticas de privacidad.
* **Code review y documentación:** ADRs (decisiones de arquitectura), README claros.

---

## Fase 5: Habilidades de semi sr (continuo)
* Estimar tareas y dividir features en entregas pequeñas.
* Depurar problemas de build (Gradle, CocoaPods, Xcode), que ocupan mucho tiempo real.
* Leer código ajeno y contribuir a un repo open source (aunque sea documentación o un fix pequeño).
* Inglés técnico para trabajar con equipos internacionales (clave si buscas trabajo remoto desde Colombia).
* Conocer lo básico de nativo (Swift/SwiftUI, Kotlin/Compose) para entender las limitaciones de RN.
* Uso de IA: usar asistentes de código bien, pero entender y poder justificar cada línea que entregas. En entrevistas lo preguntan.

---

## Portafolio: Qué incluir

> **Recomendación:** 2 proyectos fuertes + 1 pequeño, no 8 tutoriales. Un proyecto publicado en tiendas con usuarios reales vale más que diez clones. Con un solo proyecto sí puede alcanzar, siempre que sea lo bastante complejo y esté publicado, pero 2 muestran más rango.

### Proyecto 1: App principal "producción real" (el más importante)
Elige un dominio con lógica real: marketplace, delivery, fitness tracker, finanzas personales, reservas, app de comunidad, o una herramienta para un negocio local.
* **Debe incluir:**
  * Auth completa (email + social login), perfiles.
  * Backend propio o BaaS con modelo de datos no trivial.
  * Listas paginadas, búsqueda, filtros, estado complejo.
  * Notificaciones push reales.
  * Funcionalidad de dispositivo (cámara, mapas o ubicación).
  * Modo offline con sincronización.
  * Animaciones cuidadas, modo oscuro, accesibilidad, i18n.
  * Tests unitarios y al menos unos flujos E2E.
  * CI/CD con EAS y OTA updates.
  * Publicada en App Store y Google Play (o al menos TestFlight + Play Internal Testing).
  * Sentry + analytics integrados.

### Proyecto 2: App técnica que demuestre profundidad
Algo que enseñe que sabes más que "pantallas + API":
* Un módulo nativo propio (Expo Module en Swift/Kotlin) publicado en npm, o
* Una app con gráficos/animaciones avanzadas (Skia + Reanimated), o
* Una app con tiempo real (WebSockets, chat, colaboración), o
* Una app offline-first con sincronización y resolución de conflictos.

### Proyecto 3 (opcional): Contribución o librería
Un PR aceptado en una librería conocida, o una librería pequeña tuya bien documentada con tests.

---

### ¿Qué debe tener cada repositorio?

| Elemento | Detalle |
| :--- | :--- |
| **README profesional** | Problema que resuelve, capturas/GIF/video, stack, cómo correrlo, enlaces a las tiendas. |
| **Decisiones técnicas** | Por qué elegiste X sobre Y (estado, navegación, backend). |
| **Diagrama de arquitectura** | Capas, flujo de datos, estructura de carpetas. |
| **Historial de commits limpio** | Commits convencionales, PRs incluso si trabajas solo. |
| **Tests y cobertura** | Badge de CI visible. |
| **Video demo (60-90 s)** | Corriendo en un dispositivo real, iOS y Android. |
| **Retos y aprendizajes** | Qué bug difícil resolviste y cómo (esto es lo que más se valora). |
| **Métricas** | Tiempo de arranque, tamaño del bundle, crash-free rate, usuarios, descargas si las tienes. |

*Fuera del repo:* Una landing o página de portafolio simple, perfil de LinkedIn alineado, y 1-2 artículos técnicos cortos explicando un problema que resolviste.

