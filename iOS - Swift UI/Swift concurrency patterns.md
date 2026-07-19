# Formas de manejar concurrencia en Swift: Callbacks, Combine y async/await

Guía comparativa de los tres paradigmas más comunes para ejecutar trabajo asíncrono y combinar resultados de múltiples operaciones concurrentes.

Caso de ejemplo usado en todo el documento: un `InventoryService` que necesita consultar el stock de un producto en **8 bodegas distintas** (`WarehouseSource`) y combinar todos los resultados en un solo array.

---

## 1. Callbacks + `DispatchGroup` (el enfoque clásico de GCD)

**Cuándo usarlo:** el código base todavía usa closures (`completion: @escaping`) y no puedes/quieres migrar todo de una vez a async/await.

```swift
protocol WarehouseSource {
    func fetchStockLevels(completion: @escaping ([StockItem]) -> Void)
}

final class InventoryService {
    private let source: WarehouseSource

    init(source: WarehouseSource) {
        self.source = source
    }

    func loadAllStock(completion: @escaping ([StockItem]) -> Void) {
        var combinedStock: [StockItem] = []
        let group = DispatchGroup()
        let lock = NSLock()

        for _ in 0..<8 {
            group.enter()
            source.fetchStockLevels { fetchedItems in
                lock.lock()
                combinedStock.append(contentsOf: fetchedItems)
                lock.unlock()
                group.leave()
            }
        }

        group.notify(queue: .main) {
            completion(combinedStock)
        }
    }
}
```

**Puntos clave:**
- `group.enter()` / `group.leave()` deben estar balanceados — olvidar uno cuelga la app para siempre.
- Necesitas un `NSLock` (o una `DispatchQueue` serial) porque 8 closures pueden mutar `combinedStock` al mismo tiempo desde hilos distintos.
- `group.notify` se ejecuta solo cuando **todas** las tareas hicieron `leave()`.

---

## 2. Combine (`Future`, `.success`, `MergeMany`)

**Cuándo usarlo:** ya estás en una capa reactiva — ViewModels con `@Published`, pipelines con `.sink`, `.combineLatest`, `.debounce`, etc. Te conviene quedarte en el mundo `Publisher` para poder seguir componiendo.

```swift
import Combine

final class InventoryService {
    private let source: WarehouseSource

    init(source: WarehouseSource) {
        self.source = source
    }

    func stockPublisher() -> AnyPublisher<[StockItem], Never> {
        let publishers = (0..<8).map { _ in
            Future<[StockItem], Never> { promise in
                self.source.fetchStockLevels { items in
                    promise(.success(items))   
                }
            }
        }

        return Publishers.MergeMany(publishers)
            .collect()                       // espera a que TODOS los publishers emitan
            .map { $0.flatMap { $0 } }        // aplana [[StockItem]] -> [StockItem]
            .eraseToAnyPublisher()
    }
}
```

**Puntos clave:**
- `Future` envuelve una operación de un solo resultado; `promise(.success(...))` o `promise(.failure(...))` la resuelve.
- `MergeMany` combina N publishers del mismo tipo en uno solo; `.collect()` espera a que **todos** completen antes de emitir el array final.
- Un error típico (visto antes) es llamar `promise(.success(...))` **antes** de que las 8 llamadas asíncronas hayan terminado — hay que envolver cada llamada individual en su propio `Future`, como arriba, y dejar que `MergeMany + collect` haga la espera real.

---

## 3. Swift Concurrency: `async let` (tareas fijas y heterogéneas)

**Cuándo usarlo:** sabes de antemano cuántas tareas hay y cada una devuelve un tipo distinto — quieres correrlas en paralelo pero cada una tiene su propio propósito.

```swift
final class DashboardViewModel: ObservableObject {
    @Published var stockSummary: StockSummary?
    @Published var supplierList: [Supplier] = []

    private let inventoryService: InventoryService
    private let supplierService: SupplierService

    init(inventoryService: InventoryService, supplierService: SupplierService) {
        self.inventoryService = inventoryService
        self.supplierService = supplierService
    }

    func loadDashboard() async {
        async let summary = inventoryService.fetchSummary()
        async let suppliers = supplierService.fetchSuppliers()

        stockSummary = await summary
        supplierList = await suppliers
    }
}
```

**Puntos clave:**
- Ambas llamadas arrancan en paralelo apenas se declaran; el `await` solo bloquea cuando realmente necesitas el valor.
- Ideal para 2-4 operaciones distintas y conocidas de antemano. No escala bien si necesitas un número variable de tareas (tendrías que escribir `async let` uno por uno).

---

## 4. Swift Concurrency: `withTaskGroup` (tareas dinámicas y homogéneas)

**Cuándo usarlo:** el número de tareas es dinámico (viene de un loop, una lista, un contador variable) y todas devuelven el **mismo tipo**. Es el reemplazo moderno y seguro de `DispatchGroup`.

```swift
final class InventoryService {
    private let source: WarehouseSource

    init(source: WarehouseSource) {
        self.source = source
    }

    func loadAllStockAsync() async -> [StockItem] {
        await withTaskGroup(of: [StockItem].self) { group in
            for _ in 0..<8 {
                group.addTask { [source] in
                    await source.fetchStockLevelsAsync()
                }
            }

            var combinedStock: [StockItem] = []
            for await result in group {
                combinedStock.append(contentsOf: result)
            }
            return combinedStock
        }
    }
}
```

**Puntos clave:**
- No necesitas locks: `withTaskGroup` acumula resultados de forma segura porque el `for await` los recibe uno a la vez, en el contexto aislado del método.
- El grupo espera **automáticamente** a que todas las child tasks terminen antes de retornar — no hay forma de "olvidar" un `enter()/leave()` como con `DispatchGroup`.
- Si el número de bodegas fuera dinámico (`warehouseCount` en vez de `8` fijo), el `for` simplemente itera esa cantidad — se adapta sin cambiar la estructura.

---

## Tabla comparativa

| Paradigma | Tipo de dato que maneja | Requiere locks manuales | Encaja con... |
|---|---|---|---|
| `DispatchGroup` (callbacks) | Closures `@escaping` | Sí (`NSLock`, `DispatchQueue`) | Código legacy basado en callbacks |
| Combine (`Future` + `.success`) | `Publisher<Output, Failure>` | No (Combine sincroniza internamente) | ViewModels reactivos, pipelines existentes |
| `async let` | `async` funcs, cantidad fija | No | 2-4 tareas heterogéneas conocidas de antemano |
| `withTaskGroup` | `async` funcs, cantidad dinámica | No | N tareas homogéneas (loops, listas variables) |

## Una sola fuente de verdad + wrappers

Si necesitas exponer las tres formas por compatibilidad (por ejemplo, un módulo viejo que solo consume closures, un ViewModel que usa Combine, y una vista nueva que usa async/await), **no dupliques la lógica de negocio tres veces**. Implementa una sola vez en async/await (la forma más simple y segura) y envuelve:

```swift
final class InventoryService {
    private let source: WarehouseSource

    init(source: WarehouseSource) {
        self.source = source
    }

    // ✅ Única fuente de verdad
    func loadAllStockAsync() async -> [StockItem] {
        await withTaskGroup(of: [StockItem].self) { group in
            for _ in 0..<8 {
                group.addTask { [source] in await source.fetchStockLevelsAsync() }
            }
            var combinedStock: [StockItem] = []
            for await result in group { combinedStock.append(contentsOf: result) }
            return combinedStock
        }
    }

    // Wrapper para callbacks (legacy)
    func loadAllStock(completion: @escaping ([StockItem]) -> Void) {
        Task {
            let result = await loadAllStockAsync()
            completion(result)
        }
    }

    // Wrapper para Combine (legacy)
    func stockPublisher() -> AnyPublisher<[StockItem], Never> {
        Future { promise in
            Task {
                let result = await self.loadAllStockAsync()
                promise(.success(result))
            }
        }
        .eraseToAnyPublisher()
    }
}
```

Así, si aparece un bug o cambia la lógica, se corrige **en un solo lugar** y los otros dos wrappers heredan el fix automáticamente.

## Regla mental para elegir

1. ¿El código alrededor usa callbacks, Combine o async/await? → sigue esa convención.
2. ¿Estás escribiendo algo nuevo, sin restricciones? → usa async/await, es el más simple y seguro hoy.
3. ¿Necesitas exponer las tres APIs por compatibilidad? → implementa una vez en async/await y agrega wrappers finos.
4. Dentro de async/await: ¿número fijo y tipos distintos? → `async let`. ¿Número dinámico o mismo tipo en loop? → `withTaskGroup`.