# Property Wrappers en SwiftUI: `@State`, `@Binding`, `@Published`, `@StateObject`, `@ObservedObject`, `@EnvironmentObject`

Guía de referencia rápida con ejemplos tomados de un proyecto real (Projects / Analytics / Login).

---

## `@State` — estado local de la vista

Estado propio y privado de **una sola vista**. SwiftUI lo persiste entre redibujados y lo recrea si la vista desaparece de la jerarquía.

**Cuándo usarlo:** valores simples (`Bool`, `String`, `Int`, `struct`s pequeños) que solo le importan a esa vista puntual — no a sus hijos ni a sus padres.

```swift
struct ProjectsScreen: View {
    @State private var searchText = ""
    @State private var loaderProgress: Float = 0.0

    var body: some View {
        // searchText solo le importa a ProjectsScreen y a su .searchable(...)
        List { ... }
            .searchable(text: $searchText, prompt: "Search Projects...")
    }
}
```

---

## `@Binding` — el hijo actualiza el estado del padre

Una **referencia** a un `@State` (u otra fuente de verdad) que vive en otra vista. Permite que un componente hijo lea *y escriba* un valor cuyo dueño real es el padre, sin duplicar el estado.

**Cuándo usarlo:** cuando construyes un componente reutilizable (loader, formulario, control) que necesita modificar un valor que no le pertenece.

```swift
struct Loader: View {
    @Binding var progress: Float   // no es dueño del valor, solo lo referencia
    var animated: Bool

    var body: some View {
        ProgressView(value: progress)
    }
}

// El padre es el dueño real del valor:
struct ProjectsScreen: View {
    @State private var loaderProgress: Float = 0.0

    var body: some View {
        Loader(progress: $loaderProgress, animated: true) // $ crea el Binding
    }
}
```

---

## `@Published` — notifica cambios desde dentro del modelo

Vive **dentro de una clase que conforma a `ObservableObject`**. Cuando el valor cambia, dispara automáticamente `objectWillChange.send()`, avisando a cualquier vista suscrita que debe redibujarse.

**Cuándo usarlo:** en cada propiedad de tu ViewModel o Service que, al cambiar, debería reflejarse en la UI.

```swift
class ProjectsViewModel: ObservableObject {
    @Published var projects: [Project] = []   // cambia → notifica a los observadores
    @Published var isLoading: Bool = false
    let projectService: ProjectService          // no es @Published: cambiarlo no notificaría nada

    func loadProjects() {
        Task {
            isLoading = true                     // dispara notificación
            projects = await projectService.fetchProjects()  // dispara notificación
            isLoading = false                     // dispara notificación
        }
    }
}
```

`@Published` es **el emisor**. Por sí solo no hace nada en la UI — necesita que alguna vista lo esté observando (ver siguiente sección).

---

## `@StateObject` — la vista es dueña del objeto observable

Se usa al *crear* una instancia de un `ObservableObject` **dentro** de una vista. SwiftUI garantiza que esa instancia sobreviva a los redibujados de la vista (no se recrea cada vez que el `body` se evalúa).

**Regla:** si ves `= NombreDeClase(...)` al declarar la propiedad, es `@StateObject`.

```swift
struct MainScreen: View {
    // MainScreen CREA y POSEE el coordinator.
    // Si fuera @ObservedObject, cada redibujo de MainScreen
    // generaría un MainCoordinator nuevo, perdiendo el path de navegación.
    @StateObject private var coordinator = MainCoordinator()

    var body: some View {
        TabView {
            ProjectsScreen(viewModel: ProjectsViewModel(projectService: MockProjectService()))
                .tabItem { Label("Projects", systemImage: "folder") }
        }
        .environmentObject(coordinator)
    }
}
```

Otro ejemplo, el `LoginScreen`, dueño de su `AuthenticationManager`:

```swift
struct LoginScreen: View {
    @StateObject var auth = AuthenticationManager()   // se crea aquí → StateObject
    let onSuccess: () -> Void
    ...
}
```

---

## `@ObservedObject` — la vista observa un objeto inyectado desde afuera

Se usa cuando el `ObservableObject` **no se crea en esta vista**, sino que llega como parámetro desde un padre. La vista se suscribe a sus cambios, pero no controla su ciclo de vida — si la vista se recrea, el objeto no se pierde porque vive afuera.

**Regla:** si el objeto llega por parámetro/`init`, es `@ObservedObject`.

```swift
struct ProjectsScreen: View {
    // El ViewModel viene inyectado desde afuera (por ejemplo, desde MainScreen)
    @ObservedObject private var viewModel: ProjectsViewModel

    init(viewModel: ProjectsViewModel) {
        _viewModel = ObservedObject(wrappedValue: viewModel)
    }

    var body: some View {
        if viewModel.isLoading {
            Loader(progress: .constant(0), animated: true)
        } else {
            List(viewModel.projects) { project in
                ProjectRowView(project: project)
            }
        }
    }
}
```

⚠️ **Cuidado:** si en vez de recibirlo por parámetro lo instancias tú mismo (`@ObservedObject private var vm = VM()`), tienes el bug clásico: la vista no es dueña real del ciclo de vida y puede perder el estado en cada redibujo. En ese caso, corresponde `@StateObject`.

---

## `@EnvironmentObject` — el objeto viene de un ancestro, inyectado implícitamente

Igual que `@ObservedObject`, pero **sin pasarlo explícitamente por cada `init`**. Un ancestro en la jerarquía lo inyecta una vez con `.environmentObject(...)`, y cualquier descendiente puede leerlo directamente, sin que las vistas intermedias necesiten conocerlo ni reenviarlo manualmente.

**Cuándo usarlo:** objetos compartidos por muchas vistas en distintos niveles de profundidad (coordinators de navegación, sesión de usuario, tema de la app), donde pasar el objeto por cada `init` sería repetitivo.

```swift
// El ancestro lo inyecta una sola vez:
struct MainScreen: View {
    @StateObject private var coordinator = MainCoordinator()

    var body: some View {
        TabView { ... }
            .environmentObject(coordinator)   // disponible para TODOS los descendientes
    }
}

// Cualquier descendiente, sin importar cuán profundo, lo recibe sin que se lo pasen por init:
struct ProjectDetailScreen: View {
    let project: Project
    @EnvironmentObject var coordinator: MainCoordinator   // llega "del aire"

    var body: some View {
        List {
            ForEach(project.items) { item in
                NavigationLink(value: item) {
                    WorkItemRowView(item: item)
                }
            }
        }
    }
}
```

---

## Tabla resumen

| Wrapper | Dónde vive | Qué resuelve | Ejemplo del proyecto |
|---|---|---|---|
| `@State` | En la vista | Estado local, privado, simple | `searchText` en `ProjectsScreen` |
| `@Binding` | En un componente hijo | El hijo lee/escribe estado del padre | `Loader(progress: $loaderProgress)` |
| `@Published` | Dentro del `ObservableObject` | Marca qué propiedades notifican cambios | `@Published var isLoading` en `ProjectsViewModel` |
| `@StateObject` | En la vista que **crea** el objeto | Posee el ciclo de vida del objeto | `MainCoordinator()` en `MainScreen` |
| `@ObservedObject` | En la vista que **recibe** el objeto | Observa sin poseer el ciclo de vida | `ProjectsViewModel` inyectado en `ProjectsScreen` |
| `@EnvironmentObject` | En cualquier descendiente | Observa un objeto inyectado por un ancestro, sin pasarlo por cada `init` | `MainCoordinator` en `ProjectDetailScreen` |

## Cadena completa, de punta a punta

```
1. Clase (ObservableObject)
   └── @Published var isLoading      → cambia el valor

2. Objeto observable
   └── posee/observa vía @StateObject o @ObservedObject
                                       ↓
3. Vista
   └── se suscribe a objectWillChange → se redibuja automáticamente

4. (Opcional) .environmentObject(objeto)
   └── cualquier descendiente lo recibe con @EnvironmentObject,
       sin que las vistas intermedias lo tengan que pasar manualmente
```

**Regla mental rápida para decidir cuál usar:**

- ¿Es un valor simple, privado de esta vista? → `@State`
- ¿Un hijo necesita modificar el estado de su padre? → `@Binding`
- ¿Estoy dentro de una clase `ObservableObject` marcando qué dispara UI updates? → `@Published`
- ¿La vista **crea** el objeto (`= Algo()`)? → `@StateObject`
- ¿El objeto **llega por parámetro** desde afuera? → `@ObservedObject`
- ¿El objeto viene del **environment**, inyectado por un ancestro lejano? → `@EnvironmentObject`