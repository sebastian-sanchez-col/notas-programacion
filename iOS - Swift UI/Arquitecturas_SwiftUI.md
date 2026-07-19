# Arquitecturas más utilizadas en SwiftUI

Ranking según adopción real en proyectos de producción (2024-2026):

| # | Arquitectura | Uso típico |
|---|---|---|
| 1 | **MVVM** | Apps pequeñas/medianas, la más usada con diferencia |
| 2 | **MVVM-C** (MVVM + Coordinator) | Apps medianas/grandes con navegación compleja |
| 3 | **TCA** (The Composable Architecture) | Apps grandes, equipos que priorizan testabilidad y estado unidireccional |
| 4 | **MV** (patrón "solo SwiftUI", promovido por Apple) | Apps simples, cuando se quiere evitar el "boilerplate" del ViewModel |
| 5 | **VIPER / Clean Swift (VIP)** | Legado de UIKit, poco natural en SwiftUI, uso decreciente |

A continuación, estructura de carpetas + código de cada capa para las 4 más relevantes.

---

## 1. MVVM

### Estructura de carpetas
```
MyApp/
├── App/
│   └── MyAppApp.swift
├── Models/
│   └── User.swift
├── ViewModels/
│   └── UserListViewModel.swift
├── Views/
│   ├── UserListView.swift
│   └── UserRowView.swift
├── Services/
│   └── UserService.swift
└── Resources/
```

### Código

**Models/User.swift**
```swift
struct User: Identifiable, Decodable {
    let id: Int
    let name: String
    let email: String
}
```

**Services/UserService.swift**
```swift
protocol UserServiceProtocol {
    func fetchUsers() async throws -> [User]
}

final class UserService: UserServiceProtocol {
    func fetchUsers() async throws -> [User] {
        let url = URL(string: "https://api.example.com/users")!
        let (data, _) = try await URLSession.shared.data(from: url)
        return try JSONDecoder().decode([User].self, from: data)
    }
}
```

**ViewModels/UserListViewModel.swift**
```swift
@MainActor
final class UserListViewModel: ObservableObject {
    @Published private(set) var users: [User] = []
    @Published private(set) var isLoading = false
    @Published var errorMessage: String?

    private let service: UserServiceProtocol

    init(service: UserServiceProtocol = UserService()) {
        self.service = service
    }

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }
        do {
            users = try await service.fetchUsers()
        } catch {
            errorMessage = error.localizedDescription
        }
    }
}
```

**Views/UserListView.swift**
```swift
struct UserListView: View {
    @StateObject private var viewModel = UserListViewModel()

    var body: some View {
        List(viewModel.users) { user in
            UserRowView(user: user)
        }
        .task { await viewModel.loadUsers() }
        .overlay {
            if viewModel.isLoading { ProgressView() }
        }
    }
}
```

**Views/UserRowView.swift**
```swift
struct UserRowView: View {
    let user: User

    var body: some View {
        VStack(alignment: .leading) {
            Text(user.name).font(.headline)
            Text(user.email).font(.subheadline).foregroundStyle(.secondary)
        }
    }
}
```

---

## 2. MVVM-C (MVVM + Coordinator)

Añade una capa de navegación desacoplada de la vista, útil cuando el flujo entre pantallas crece.

### Estructura de carpetas
```
MyApp/
├── App/
│   └── MyAppApp.swift
├── Coordinators/
│   ├── AppCoordinator.swift
│   └── UserFlowCoordinator.swift
├── Models/
│   └── User.swift
├── ViewModels/
│   ├── UserListViewModel.swift
│   └── UserDetailViewModel.swift
├── Views/
│   ├── UserListView.swift
│   └── UserDetailView.swift
├── Services/
│   └── UserService.swift
```

### Código

**Coordinators/AppCoordinator.swift**
```swift
enum AppRoute: Hashable {
    case userList
    case userDetail(User)
}

@MainActor
final class AppCoordinator: ObservableObject {
    @Published var path = NavigationPath()

    func goToDetail(_ user: User) {
        path.append(AppRoute.userDetail(user))
    }

    func pop() {
        path.removeLast()
    }
}
```

**App/MyAppApp.swift**
```swift
@main
struct MyAppApp: App {
    @StateObject private var coordinator = AppCoordinator()

    var body: some Scene {
        WindowGroup {
            NavigationStack(path: $coordinator.path) {
                UserListView(viewModel: UserListViewModel())
                    .navigationDestination(for: AppRoute.self) { route in
                        switch route {
                        case .userList:
                            UserListView(viewModel: UserListViewModel())
                        case .userDetail(let user):
                            UserDetailView(viewModel: UserDetailViewModel(user: user))
                        }
                    }
            }
            .environmentObject(coordinator)
        }
    }
}
```

**Views/UserListView.swift**
```swift
struct UserListView: View {
    @EnvironmentObject private var coordinator: AppCoordinator
    @StateObject var viewModel: UserListViewModel

    var body: some View {
        List(viewModel.users) { user in
            Button(user.name) {
                coordinator.goToDetail(user)
            }
        }
        .task { await viewModel.loadUsers() }
    }
}
```

**ViewModels/UserDetailViewModel.swift**
```swift
@MainActor
final class UserDetailViewModel: ObservableObject {
    @Published var user: User

    init(user: User) {
        self.user = user
    }
}
```

**Views/UserDetailView.swift**
```swift
struct UserDetailView: View {
    @ObservedObject var viewModel: UserDetailViewModel

    var body: some View {
        VStack {
            Text(viewModel.user.name).font(.title)
            Text(viewModel.user.email)
        }
    }
}
```

> Nota: la ViewModel **no conoce** al Coordinator, ni al revés directamente conoce las vistas: la vista pide navegación, el coordinator decide la ruta. Así se mantiene el desacople y se facilita el testing de cada capa por separado.

---

## 3. TCA (The Composable Architecture)

Basada en `State`, `Action`, `Reducer` y `Store`, con manejo unidireccional de datos. Requiere el paquete `swift-composable-architecture` (Point-Free).

### Estructura de carpetas
```
MyApp/
├── App/
│   └── MyAppApp.swift
├── Features/
│   └── UserList/
│       ├── UserListFeature.swift   // State + Action + Reducer
│       └── UserListView.swift
├── Models/
│   └── User.swift
├── Clients/
│   └── UserClient.swift            // dependencia inyectable
```

### Código

**Clients/UserClient.swift**
```swift
import Dependencies

struct UserClient {
    var fetchUsers: () async throws -> [User]
}

extension UserClient: DependencyKey {
    static let liveValue = UserClient(
        fetchUsers: {
            let url = URL(string: "https://api.example.com/users")!
            let (data, _) = try await URLSession.shared.data(from: url)
            return try JSONDecoder().decode([User].self, from: data)
        }
    )
}

extension DependencyValues {
    var userClient: UserClient {
        get { self[UserClient.self] }
        set { self[UserClient.self] = newValue }
    }
}
```

**Features/UserList/UserListFeature.swift**
```swift
import ComposableArchitecture

@Reducer
struct UserListFeature {
    @ObservableState
    struct State: Equatable {
        var users: [User] = []
        var isLoading = false
        var errorMessage: String?
    }

    enum Action {
        case onAppear
        case usersResponse(Result<[User], Error>)
    }

    @Dependency(\.userClient) var userClient

    var body: some ReducerOf<Self> {
        Reduce { state, action in
            switch action {
            case .onAppear:
                state.isLoading = true
                return .run { send in
                    let result = await Result { try await userClient.fetchUsers() }
                    await send(.usersResponse(result))
                }

            case .usersResponse(.success(let users)):
                state.isLoading = false
                state.users = users
                return .none

            case .usersResponse(.failure(let error)):
                state.isLoading = false
                state.errorMessage = error.localizedDescription
                return .none
            }
        }
    }
}

extension UserListFeature.Action: Equatable {
    static func == (lhs: Self, rhs: Self) -> Bool {
        switch (lhs, rhs) {
        case (.onAppear, .onAppear): true
        default: false
        }
    }
}
```

**Features/UserList/UserListView.swift**
```swift
import ComposableArchitecture

struct UserListView: View {
    let store: StoreOf<UserListFeature>

    var body: some View {
        List(store.users) { user in
            Text(user.name)
        }
        .overlay {
            if store.isLoading { ProgressView() }
        }
        .onAppear { store.send(.onAppear) }
    }
}
```

**App/MyAppApp.swift**
```swift
@main
struct MyAppApp: App {
    var body: some Scene {
        WindowGroup {
            UserListView(
                store: Store(initialState: UserListFeature.State()) {
                    UserListFeature()
                }
            )
        }
    }
}
```

---

## 4. MV (patrón "solo SwiftUI")

Apple, con `@Observable` (Swift 5.9+), ha impulsado un patrón sin ViewModel formal: la vista observa directamente un `@Observable` "Model" que combina estado y lógica. Reduce boilerplate pero mezcla más responsabilidades en la vista.

### Estructura de carpetas
```
MyApp/
├── App/
│   └── MyAppApp.swift
├── Models/
│   ├── User.swift
│   └── UserStore.swift   // @Observable, hace de "model" con lógica
├── Views/
│   └── UserListView.swift
```

### Código

**Models/UserStore.swift**
```swift
@Observable
final class UserStore {
    var users: [User] = []
    var isLoading = false

    func loadUsers() async {
        isLoading = true
        defer { isLoading = false }
        do {
            let url = URL(string: "https://api.example.com/users")!
            let (data, _) = try await URLSession.shared.data(from: url)
            users = try JSONDecoder().decode([User].self, from: data)
        } catch {
            print(error)
        }
    }
}
```

**Views/UserListView.swift**
```swift
struct UserListView: View {
    @State private var store = UserStore()

    var body: some View {
        List(store.users) { user in
            Text(user.name)
        }
        .task { await store.loadUsers() }
    }
}
```

---

## ¿Cuál elegir?

- **Proyecto pequeño / prototipo** → MV o MVVM simple.
- **App mediana con varias pantallas y flujos de navegación** → MVVM-C.
- **App grande, equipo grande, se necesita testear cada transición de estado** → TCA.
- **Legado de UIKit migrando gradualmente** → VIPER/Clean Swift, aunque en SwiftUI puro tiende a sentirse sobre-diseñado.

En la práctica, la combinación más común en apps SwiftUI de tamaño medio sigue siendo **MVVM-C**: MVVM para la lógica de presentación por pantalla, y un Coordinator (o un `NavigationPath` centralizado como arriba) para el enrutamiento.