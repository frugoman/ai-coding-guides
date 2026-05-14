# iOS Architecture Guide For Claude Code

Use this file as the default architecture playbook for iOS projects. It is intentionally opinionated. Follow it unless the existing codebase has a stronger local convention or the user explicitly asks for a different design.

The goal is not to maximize layers, protocols, or patterns. The goal is to make change easier, safer, and more predictable.

## Instruction Priority

When Claude Code works in an app that contains this file, apply instructions in this order:

1. Direct user request.
2. Project-specific `AGENTS.md`, `CLAUDE.md`, package docs, or existing architecture docs.
3. Existing code patterns in the touched feature.
4. This file.
5. General iOS conventions.

If the existing project conflicts with this guide, do not rewrite the project casually. Make the requested change in the local style, then call out the conflict and suggest a small migration step when it matters.

## Core Rule

Every important decision needs a clear home.

- Views render and report user intent.
- ViewModels expose renderable state, map meaningful results into state, and emit small UI events.
- Use cases perform application operations.
- Repositories answer data questions and hide data policy.
- Factories build feature graphs.
- Composition roots choose concrete implementations.
- Flows or parent features own navigation decisions.
- Adapters translate between boundaries.
- Decorators add behavior without editing the wrapped object.

If a type is doing work from several bullets, split the responsibility or name the type more honestly.

## Default Dependency Direction

Prefer this direction for non-trivial features:

```text
View intent
    -> composed action / route closure
    -> use case or feature operation
    -> output protocol
    -> ViewModel
    -> renderable state
    -> View
```

Navigation follows a separate path:

```text
ViewModel screen event or row selection
    -> parent flow / root view / coordinator
    -> route / NavigationStack path / UIKit navigation
```

Do not let a leaf View or leaf ViewModel know the whole app journey.

## Canonical SwiftUI Feature Shape

For a new non-trivial SwiftUI feature, prefer this shape unless the project already has a clear alternative:

```text
ProductsRootView / Flow
    owns NavigationStack path and navigation decisions

ProductsFeatureFactory
    builds the feature graph and wires callbacks

ProductsRoute
    owns or observes the ViewModel lifetime
    triggers the operation through injected closures

ProductsView
    renders ProductsViewState
    reports user intent

ProductsViewModel
    maps operation outputs/results into renderable state
    emits small screen events or row callbacks

LoadProductsUseCase
    coordinates the application operation

ProductsRepository
    hides data policy and data-source details
```

Default file shape:

```text
Features/
  Products/
    ProductsRootView.swift
    ProductsFeatureFactory.swift
    ProductsRoute.swift
    ProductsView.swift
    ProductsViewModel.swift
    ProductsViewState.swift
    LoadProductsUseCase.swift
    ProductsRepository.swift
```

This is a starting point, not a quota. Remove pieces that do not carry responsibility.

## Claude Code Workflow

Before making architecture changes:

1. Read the existing feature files and local tests.
2. Identify the current ownership of state, navigation, construction, data loading, and analytics.
3. Make the smallest change that moves the decision to the right owner.
4. Do not introduce a protocol, use case, factory, or decorator unless it buys a specific change.
5. Update or add focused tests for behavior and boundaries.
6. Build and run the relevant tests.

Do not perform broad refactors while implementing a feature unless the user asked for it. If a better architecture requires a large migration, create a small transitional boundary first.

When changing existing code:

- Preserve public behavior first.
- Move one responsibility at a time.
- Prefer adding a focused seam near the painful dependency over introducing an app-wide pattern.
- Keep old and new styles temporarily if that is the safest migration.
- Leave a short note in the final response when architecture debt remains.

## SwiftUI View Rules

SwiftUI views should be boring.

Views may:

- Render state.
- Own local ephemeral UI state with `@State`.
- Own an observable object with `@StateObject` only when the view creates that object and owns its lifetime.
- Observe an injected observable object with `@ObservedObject`.
- Read truly environmental values with `@Environment`.
- Trigger injected intent closures, such as `onAppear`, `onSubmit`, `onRetry`, or `onProductSelected`.

Views must not:

- Create repositories, API clients, databases, analytics trackers, or use cases.
- Decide which concrete implementation a feature uses.
- Navigate to app-level destinations from leaf screens.
- Contain business rules.
- Pull core feature dependencies from `Environment` just to avoid initializer parameters.

### StateObject vs ObservedObject

Do not decide this based on the type name. A ViewModel is not automatically a `@StateObject`.

Choose the wrapper based on ownership:

- Use `@StateObject` when the view creates the observable object and owns its lifetime.
- Use `@ObservedObject` when the observable object is created and owned elsewhere, such as by a factory, route, parent view, or flow.
- If the object is injected through an initializer, default to `@ObservedObject` unless the initializer is explicitly transferring lifetime ownership to this view.

`@StateObject` example:

```swift
struct ProductsRoute: View {
    @StateObject private var viewModel: ProductsViewModel

    init(factory: ProductsFeatureFactory) {
        _viewModel = StateObject(
            wrappedValue: factory.makeViewModel()
        )
    }
}
```

`@ObservedObject` example:

```swift
struct ProductsRoute: View {
    @ObservedObject var viewModel: ProductsViewModel
    let onAppear: () async -> Void
}
```

Never recreate a ViewModel from a frequently recomputed `body` if it should keep state. Do not wrap an injected ViewModel in `@StateObject` just because it is a ViewModel.

If the project uses Swift's Observation framework instead of `ObservableObject`, keep the same ownership rule:

- Use `@State` when the view creates and owns an `@Observable` model.
- Use `@Bindable` only when the view needs bindings into an injected observable model.
- Pass injected observable models normally when the view does not own them.
- Do not hide dependency ownership by creating observable models inside `body`.

## ViewModel Rules

ViewModels prepare UI state. They are not the whole feature.

ViewModels may:

- Expose `@Published private(set)` state.
- Map use case outputs or feature results into renderable state.
- Own editable draft state for a screen.
- Receive field-change intents that update editable draft state.
- Emit small screen events to a parent.
- Use presentation collaborators, such as formatters.

ViewModels must not:

- Create concrete repositories, API clients, databases, or SDKs.
- Call use cases or repositories to perform application operations.
- Submit forms.
- Validate operation rules.
- Map editable drafts into application commands.
- Decide navigation routes.
- Know `NavigationPath`, `NavController`, UIKit navigation controllers, or app-level route strings.
- Own analytics policy by default.
- Become the place where every operation is triggered, interpreted, tracked, and navigated.

Default ViewModel shape:

```swift
@MainActor
final class ProductsViewModel: ObservableObject, LoadProductsOutput {
    @Published private(set) var state: ProductsState = .loading

    func productsLoadingStarted() {
        state = .loading
    }

    func productsLoaded(_ products: [Product]) {
        state = .loaded(
            products.map(ProductRowViewState.init)
        )
    }

    func productsLoadingFailed() {
        state = .failed("Could not load products.")
    }
}
```

Views may call ViewModel methods for presentation intents, such as field changes or local UI actions that update renderable state. Views must not call ViewModel methods that perform application operations, such as `submit()`, `loadProducts()`, `placeOrder()`, or `login()`.

## Use Case Rules

A use case is an application operation. Add one when the operation deserves a name.

Use a use case when the operation:

- Coordinates multiple dependencies.
- Has meaningful business/application outcomes.
- Owns retry, cache, validation, ordering, or persistence policy.
- Should be tested independently from presentation.
- Would make the ViewModel know too much.

Do not add a use case that only forwards to one repository unless the operation name creates a real boundary.

Preferred shape:

```swift
protocol LoginUseCaseOutput: AnyObject {
    func loginDidStart()
    func loginDidSucceed()
    func loginDidFail(_ reason: LoginFailure)
}

final class LoginUseCase {
    let authenticator: Authenticating
    let sessionStore: SessionStoring
    weak var output: LoginUseCaseOutput?

    init(
        authenticator: Authenticating,
        sessionStore: SessionStoring,
        output: LoginUseCaseOutput?
    ) {
        self.authenticator = authenticator
        self.sessionStore = sessionStore
        self.output = output
    }

    func execute(email: String, password: String) async {
        await MainActor.run {
            output?.loginDidStart()
        }

        do {
            let session = try await authenticator.login(
                email: email,
                password: password
            )
            try await sessionStore.save(session)

            await MainActor.run {
                output?.loginDidSucceed()
            }
        } catch {
            await MainActor.run {
                output?.loginDidFail(.invalidCredentials)
            }
        }
    }
}
```

If the use case is `@MainActor`, you do not need `MainActor.run`, but do not perform heavy work on the main actor.

## Outputs vs Screen Events

Use case outputs describe operation outcomes:

```swift
protocol PlaceOrderOutput {
    func placeOrderDidStart()
    func placeOrderDidSucceed(_ order: Order)
    func placeOrderDidFail(_ reason: PlaceOrderFailure)
}
```

Screen events describe UI-level events that a parent may interpret:

```swift
enum LoginScreenEvent {
    case loginCompleted
}
```

Do not use a domain/use-case output as a navigation command. Do not name navigation callbacks `Output` unless they are truly use case outputs.

## Repository Rules

A repository answers data questions and hides data policy.

Use a repository when it hides at least one of these:

- Remote API shape.
- Persistence details.
- Cache/network fallback.
- Mapping DTOs/entities into domain models.
- Offline or refresh policy.
- Multiple data sources.

Good repository:

```swift
protocol ProductsRepository {
    func products() async throws -> [Product]
    func product(id: Product.ID) async throws -> Product
}
```

Weak repository:

```swift
protocol ProductsRepository {
    func productsResponse() async throws -> ProductsResponse
}
```

The second leaks transport language. Prefer app/domain language at the repository boundary.

## Model Boundary Rules

Keep model languages separate:

- DTOs describe transport.
- Database entities describe persistence.
- Domain models describe application meaning.
- UI state describes rendering.
- Route values describe navigation identity.

Do not pass DTOs, Core Data models, Realm objects, or Firebase snapshots into ViewModels or Views unless the whole app deliberately uses those as domain models.

Preferred mapping direction:

```text
ProductsResponse
    -> Product DTO
    -> Product domain model
    -> ProductUIState
```

Mapping may live in an adapter, repository, mapper, or ViewModel depending on what is being translated:

- API/database shape to domain: repository or data adapter.
- Domain to display state: ViewModel or formatting collaborator.
- Domain to route identity: flow/root.

## Protocol Rules

Protocols are not architecture by themselves.

Create a protocol when:

- The caller should not know the concrete implementation.
- There are multiple implementations or likely substitutions.
- A boundary crosses modules, teams, or layers.
- Tests/previews need meaningful fakes.
- The protocol name describes a capability from the caller's point of view.

Do not create a protocol only because every class gets one.

Prefer capability names:

```swift
protocol ProductsRepository { ... }
protocol AnalyticsTracking { ... }
protocol SessionStoring { ... }
```

Avoid vague names:

```swift
protocol ProductsManagerProtocol { ... }
protocol ProductsServiceProtocol { ... }
```

## Dependency Injection Rules

Default to initializer injection.

```swift
final class ProductDetailsViewModel: ObservableObject {
    private let priceFormatter: ProductPriceFormatting

    init(priceFormatter: ProductPriceFormatting) {
        self.priceFormatter = priceFormatter
    }
}
```

Do not create concrete dependencies inside feature code:

```swift
final class ProductsViewModel: ObservableObject {
    private let repository = URLSessionProductsRepository()
}
```

Concrete choices belong in the composition root or feature factory.

Use dependency containers sparingly. A container is a wiring tool, not the architecture.

## Route And Container Rules

A route or container view connects SwiftUI lifecycle to the feature graph. It is allowed to know about the ViewModel and injected operation closures, but it should still avoid business policy.

Use a route when the plain view should stay easy to preview and test:

```swift
struct ProductsListRoute: View {
    @ObservedObject var viewModel: ProductsViewModel
    let onAppear: () async -> Void

    var body: some View {
        ProductsListView(
            state: viewModel.state,
            onRetry: {
                Task {
                    await onAppear()
                }
            }
        )
        .task {
            await onAppear()
        }
    }
}
```

Rules:

- The route may trigger lifecycle operations.
- The route may adapt SwiftUI APIs into feature intents.
- The route may own `@StateObject` only if it creates the ViewModel in `init`.
- The route should not choose concrete repositories, API clients, or persistence.
- The route should not interpret domain outcomes as app navigation.
- Avoid calling factories from `body` when that recreates stateful objects.

## Factory Rules

Factories build objects and connect dependencies. They do not decide product policy or navigation meaning.

Good factory:

```swift
struct ProductsFeatureFactory {
    let productsRepository: ProductsRepository

    func makeProductsListRoute(
        onProductSelected: @escaping (Product) -> Void
    ) -> ProductsListRoute {
        let viewModel = ProductsViewModel(
            onProductSelected: onProductSelected
        )
        let useCase = LoadProductsUseCase(
            repository: productsRepository,
            output: viewModel
        )

        return ProductsListRoute(
            viewModel: viewModel,
            onAppear: useCase.execute
        )
    }
}
```

Bad factory:

```swift
struct LoginFeatureFactory {
    func make() -> LoginScreen {
        let viewModel = LoginViewModel()
        viewModel.onEvent = { event in
            switch event {
            case .loginCompleted:
                appRouter.showHome()
            }
        }
        return LoginScreen(viewModel: viewModel)
    }
}
```

That factory is interpreting a screen event. Move that switch to a flow or parent.

## Composition Root Rules

The composition root is where concrete implementations are chosen.

In SwiftUI, the app entry point or root object is usually the composition root:

```swift
@main
struct ProductsApp: App {
    private let compositionRoot = CompositionRoot()

    var body: some Scene {
        WindowGroup {
            compositionRoot.makeRootView()
        }
    }
}
```

The composition root may delegate to feature factories:

```swift
struct CompositionRoot {
    private let httpClient = URLSessionHTTPClient()

    func makeProductsFeature() -> ProductsFeatureFactory {
        ProductsFeatureFactory(
            productsRepository: RemoteProductsRepository(
                httpClient: httpClient
            )
        )
    }
}
```

Do not let leaf features choose concrete infrastructure.

## Navigation And Flow Rules

Navigation belongs to a parent, flow, coordinator, or root view.

SwiftUI example:

```swift
enum ProductsRoute: Hashable {
    case detail(Product.ID)
}

struct ProductsRootView: View {
    @State private var path: [ProductsRoute] = []
    let factory: ProductsFeatureFactory

    var body: some View {
        NavigationStack(path: $path) {
            factory.makeProductsListRoute(
                onProductSelected: { product in
                    path.append(.detail(product.id))
                }
            )
            .navigationDestination(for: ProductsRoute.self) { route in
                switch route {
                case let .detail(productID):
                    factory.makeProductDetailRoute(productID: productID)
                }
            }
        }
    }
}
```

Rules:

- Leaf views expose intent closures.
- ViewModels may emit screen events.
- Parent flows interpret the intent/event.
- Factories wire callbacks but do not decide navigation meaning.
- Use route values that are stable and appropriate for the feature.

## Selection Rules

Selection is not navigation.

For selectable lists, prefer this split:

```swift
struct ProductUIState: Equatable {
    let title: String
    let subtitle: String
    let price: String
}

struct SelectableProductRow: Identifiable {
    let id: String
    let product: ProductUIState
    let onSelection: () -> Void
}
```

`ProductUIState` is render data. `SelectableProductRow` adds list identity and behavior.

Mapping:

```swift
func didLoadProducts(_ products: [Product]) {
    rows = products.map { product in
        SelectableProductRow(
            id: product.id.rawValue,
            product: ProductUIState(
                title: product.name,
                subtitle: product.shortDescription,
                price: product.price.formatted()
            ),
            onSelection: { [onProductSelected] in
                onProductSelected(product)
            }
        )
    }
}
```

View:

```swift
struct ProductsListView: View {
    let rows: [SelectableProductRow]

    var body: some View {
        List(rows) { row in
            Button(action: row.onSelection) {
                ProductRowView(product: row.product)
            }
        }
    }
}
```

Rules:

- Do not add domain IDs to UI state only for navigation.
- Use a stable `String` as list identity when the ID is only UI/list identity.
- Do not make `SelectableProductRow` `Equatable`; it contains a closure.
- `ProductUIState` may be `Equatable` because it is pure data.
- Pass `Product.ID` when detail should reload, restore, deep link, or fetch missing data.
- Pass `Product` when detail is a continuation of already-loaded data.
- Use a row-ID lookup only when selection must use the latest domain object or row closures are inappropriate.

## Complex Form Rules

Complex forms need explicit boundaries. Do not let the ViewModel become the form operation.

Use these roles:

- `FormDraft`: UI-specific editable state for one form experience.
- `ViewState`: renderable data derived from draft, validation, loading, and output events.
- `Validator`: decides whether a draft can be submitted.
- `CommandMapper`: maps a draft into application operation input.
- `FormOperation`: coordinates validation, mapping, use case execution, and output reporting.
- `UseCase`: performs the application operation.
- `ViewModel`: owns draft editing and maps outputs into render state.
- `Flow`: owns navigation after form completion.

The draft belongs to the UI. The command belongs to the operation.

Different UIs may have different draft models that map to the same command:

```swift
struct FullCheckoutDraft: Equatable {
    var street: String
    var city: String
    var postalCode: String
    var useShippingAsBilling: Bool
    var acceptsTerms: Bool
}

struct ExpressCheckoutDraft: Equatable {
    var selectedAddressID: String?
    var selectedPaymentMethodID: String?
    var acceptsTerms: Bool
}

struct PlaceOrderCommand: Equatable {
    let shippingAddress: Address
    let billingAddress: Address
    let paymentMethodID: PaymentMethod.ID
    let acceptsTerms: Bool
}
```

The ViewModel may own and update the draft:

```swift
@MainActor
final class CheckoutViewModel: ObservableObject, CheckoutFormOutput {
    @Published private(set) var state: CheckoutViewState

    private var draft: FullCheckoutDraft

    var currentDraft: FullCheckoutDraft {
        draft
    }

    init(draft: FullCheckoutDraft) {
        self.draft = draft
        self.state = CheckoutViewState(draft: draft)
    }

    func update(_ change: CheckoutFieldChange) {
        draft = draft.applying(change)
        state = CheckoutViewState(draft: draft)
    }

    func checkoutValidationFailed(_ validation: CheckoutValidation) {
        state = CheckoutViewState(
            draft: draft,
            validation: validation
        )
    }

    func checkoutSubmissionStarted() {
        state = state.submitting()
    }

    func checkoutSubmissionFailed(_ message: String) {
        state = state.failed(message)
    }

    func checkoutSubmissionSucceeded() {
        state = state.succeeded()
    }
}
```

The ViewModel must not have a `submit()` method. Submitting is not a presentation-state update.

Use a form operation:

```swift
final class SubmitCheckoutForm {
    private let validator: CheckoutDraftValidating
    private let mapper: CheckoutCommandMapping
    private let placeOrder: PlaceOrderUseCase
    private weak var output: CheckoutFormOutput?

    init(
        validator: CheckoutDraftValidating,
        mapper: CheckoutCommandMapping,
        placeOrder: PlaceOrderUseCase,
        output: CheckoutFormOutput
    ) {
        self.validator = validator
        self.mapper = mapper
        self.placeOrder = placeOrder
        self.output = output
    }

    func execute(_ draft: FullCheckoutDraft) async {
        let validation = validator.validate(draft)

        guard validation.isValid else {
            await MainActor.run {
                output?.checkoutValidationFailed(validation)
            }
            return
        }

        let command = mapper.command(from: draft)

        await MainActor.run {
            output?.checkoutSubmissionStarted()
        }

        do {
            try await placeOrder.execute(command)

            await MainActor.run {
                output?.checkoutSubmissionSucceeded()
            }
        } catch {
            await MainActor.run {
                output?.checkoutSubmissionFailed(
                    "Could not place your order."
                )
            }
        }
    }
}
```

The route wires the current draft into the submit intent:

```swift
struct CheckoutRoute: View {
    @ObservedObject var viewModel: CheckoutViewModel
    let onSubmit: (FullCheckoutDraft) async -> Void

    var body: some View {
        CheckoutView(
            state: viewModel.state,
            onChange: viewModel.update,
            onSubmit: {
                await onSubmit(viewModel.currentDraft)
            }
        )
    }
}
```

Rules:

- Do not use the domain model as form state unless the UI truly edits that exact shape.
- Do not add submit/load/save methods to ViewModels.
- Do not validate operation rules inside ViewModels.
- Do not map drafts into commands inside ViewModels.
- Do not make factories interpret form completion or navigation events.
- Do not pass half-edited UI strings into use cases.
- Test draft editing in ViewModel tests.
- Test validation in validator tests.
- Test draft-to-command mapping in mapper tests.
- Test submit coordination in form-operation tests.

## Adapters And Decorators

An adapter should adapt something. It should translate language or hide a detail.

Good adapter:

```swift
struct RemoteProductsRepository: ProductsRepository {
    let client: HTTPClient

    func products() async throws -> [Product] {
        let response: ProductsResponse = try await client.get("/products")
        return response.items.map(Product.init)
    }
}
```

Weak adapter:

```swift
final class ProductsServiceAdapter: ProductsService {
    let apiClient: APIClient

    func loadProducts() async throws -> [Product] {
        try await apiClient.loadProducts()
    }
}
```

The weak adapter only renames a method. Keep it only if it is a temporary seam or part of a larger boundary.

Use decorators for cross-cutting behavior:

```swift
final class AnalyticsLoginOutputDecorator: LoginUseCaseOutput {
    private let decoratee: LoginUseCaseOutput
    private let analytics: AnalyticsTracking

    init(
        decoratee: LoginUseCaseOutput,
        analytics: AnalyticsTracking
    ) {
        self.decoratee = decoratee
        self.analytics = analytics
    }

    func loginDidStart() {
        decoratee.loginDidStart()
    }

    func loginDidSucceed() {
        analytics.track("login_succeeded")
        decoratee.loginDidSucceed()
    }

    func loginDidFail(_ reason: LoginFailure) {
        analytics.track("login_failed")
        decoratee.loginDidFail(reason)
    }
}
```

Prefer decorators when analytics/logging/retry/caching should be added without polluting the ViewModel or use case.

## Formatting Rules

ViewModels may produce renderable strings and values as part of mapping outputs into view state.

If formatting has policy, localization complexity, experiments, business rules, or reuse pressure, give it a collaborator:

```swift
protocol ProductPriceFormatting {
    func string(from price: Money) -> String
}

struct ProductPriceFormatter: ProductPriceFormatting {
    let moneyFormatter: MoneyFormatting

    func string(from price: Money) -> String {
        moneyFormatter.string(from: price)
    }
}
```

Do not name this collaborator `Presenter` unless the app is explicitly using MVP.

## Async And Error Rules

Keep concurrency ownership explicit.

Rules:

- UI state mutations happen on the main actor.
- Use cases may be async, but they should not do heavy work on the main actor.
- Repositories should expose async operations in app/domain language.
- Cancellation should follow Swift structured concurrency when possible.
- Do not hide long-running work inside initializers.
- Do not start network requests from a SwiftUI view's `body`.

Prefer errors that are meaningful at the boundary:

```swift
enum LoadProductsFailure: Equatable {
    case offline
    case unauthorized
    case unavailable
}
```

Translate low-level errors at the boundary that understands them:

- HTTP and decoding errors: data adapter or repository.
- Application operation failures: use case.
- User-facing copy: ViewModel or formatting/localization collaborator.

Do not leak `URLError`, `DecodingError`, status codes, or backend error payloads into views unless the UI truly needs that technical detail.

## Environment Rules

Use SwiftUI `Environment` for values that are truly environmental:

- Color scheme.
- Locale.
- Calendar.
- Feature flags that are app context.
- Theme/style values.

Do not use Environment as a service locator for feature dependencies:

```swift
@Environment(\.productsRepository) private var repository
```

Prefer visible feature dependencies:

```swift
ProductsListRoute(
    viewModel: viewModel,
    onAppear: useCase.execute
)
```

## Feature Organization

Prefer organizing by feature when the app grows:

```text
Features/
  Products/
    ProductsListView.swift
    ProductsViewModel.swift
    LoadProductsUseCase.swift
    ProductsRepository.swift
    ProductsFeatureFactory.swift
```

Use shared modules only for stable cross-feature abstractions. Do not create a shared module just because two files look similar today.

## UIKit Compatibility

In UIKit features, keep the same responsibilities with UIKit names:

- `UIViewController` renders, binds, and reports user intent.
- ViewModel prepares state and screen events.
- Coordinator or parent flow owns navigation.
- Factory builds the view controller graph.
- Composition root chooses concrete dependencies.

UIKit view controllers may perform binding work that SwiftUI views express declaratively. They still should not create infrastructure, own app navigation policy, or become use cases.

## Testing Rules

Test behavior and boundaries, not every internal step.

### ViewModel Tests

Compare render state as data:

```swift
XCTAssertEqual(
    viewModel.rows.map(\.product),
    [
        ProductUIState(
            title: "Keyboard",
            subtitle: "Mechanical",
            price: "$120"
        )
    ]
)
```

Test behavior by invoking behavior:

```swift
viewModel.rows[0].onSelection()
XCTAssertEqual(selectedProduct, .keyboard)
```

### Use Case Tests

Test meaningful outcomes:

```swift
func test_login_savesSessionAndReportsSuccess() async {
    let output = RecordingLoginOutput()
    let useCase = LoginUseCase(
        authenticator: SuccessfulAuthenticator(),
        sessionStore: InMemorySessionStore(),
        output: output
    )

    await useCase.execute(email: "a@b.com", password: "secret")

    XCTAssertEqual(output.events, [.started, .succeeded])
}
```

### Composition Tests

Do not assert every concrete type in the object graph unless that is the contract. Prefer smoke tests for wiring and focused tests for behavior.

## Preview Rules

SwiftUI previews should use sample state or fakes, not real infrastructure.

Good preview shape:

```swift
#Preview {
    ProductsListView(
        state: .loaded([
            SelectableProductRow(
                id: "keyboard",
                product: ProductUIState(
                    title: "Keyboard",
                    subtitle: "Mechanical",
                    price: "$120"
                ),
                onSelection: {}
            )
        ]),
        onRetry: {}
    )
}
```

Do not make previews hit the network, read production databases, or require app-wide composition unless the preview is specifically testing integration.

## Naming Rules

Names should reveal responsibility.

Use:

- `ProductsViewModel`
- `ProductUIState`
- `SelectableProductRow`
- `LoadProductsUseCase`
- `ProductsRepository`
- `ProductsFeatureFactory`
- `ProductsRoute`
- `LoginScreenEvent`
- `AnalyticsLoginOutputDecorator`

Avoid:

- `Manager`
- `Service` unless it really represents an external service boundary
- `Helper`
- `Presenter` outside MVP
- `Output` for UI/navigation events
- `Factory` for objects that own lifecycle or policy

## Overkill Rules

Before adding a boundary, answer:

1. What change does this boundary make cheaper?
2. Is that change likely in this codebase?
3. Who owns each side?
4. Does this hide a real detail or only rename it?
5. Does it make tests simpler or more meaningful?
6. Does it reduce what the caller needs to know?

If the answers are weak, do not add the abstraction.

If the answers are strong, the boundary is probably doing useful work.

## Anti-Patterns

Avoid:

- ViewModels that create repositories or API clients.
- ViewModels that call use cases or repositories to perform operations.
- ViewModels with submit/load/save methods for application operations.
- ViewModels that validate form submission rules.
- ViewModels that map form drafts into operation commands.
- ViewModels that own app navigation.
- Factories that interpret screen events.
- Use cases that only forward without meaning.
- Protocols created only for mocks.
- Repositories that expose DTOs or backend response models.
- Environment-based service locator dependencies.
- UI state carrying domain identity only for navigation.
- Equatable conformance on closure-containing types.
- Tests that assert every collaborator call when behavior would be clearer.
- One app-wide ViewModel that owns session, navigation, tabs, deep links, and feature state.

## Hard Rules For Claude Code

When generating or modifying code, do not:

- Add a protocol only so a mock framework can mock it.
- Put `NavigationPath`, root route enums, or app router calls inside a leaf ViewModel.
- Put `URLSession`, Firebase, Core Data, Realm, analytics SDKs, or payment SDKs directly inside a ViewModel.
- Put a `switch` over screen events inside a factory.
- Put closures inside an `Equatable` state type.
- Store domain objects in SwiftUI `Environment` as a shortcut for dependency passing.
- Create a dependency container and then resolve from it throughout the feature.
- Add `Manager`, `Helper`, or `Service` names when a more specific responsibility is known.
- Convert every repository call into a use case when there is no operation-level behavior.
- Hide app behavior in SwiftUI modifiers that make the feature hard to trace.
- Add ViewModel operation methods because the operation looks short.

When the user asks for speed or a small fix, still keep these boundaries honest. A smaller implementation is fine; a hidden dependency direction is not.

## Decision Defaults

Use these defaults when Claude Code must choose:

- New non-trivial feature: create a feature factory, ViewModel, render state, use case if operation has policy, repository if data policy exists.
- Complex form: use a UI-specific draft, validator, command mapper, form operation, use case, output, and flow-owned navigation.
- SwiftUI list selection: use `ProductUIState` plus `SelectableProductRow`.
- Detail navigation: use ID if detail reloads/restores/deep-links; use model if detail continues from loaded data.
- Analytics: prefer output decorators or operation-level decorators over ViewModel calls.
- Formatting: use a formatter collaborator when formatting has policy.
- Existing codebase: follow local patterns unless they conflict with these rules and the task requires correction.

## Final Check Before Finishing

Before returning work:

- Does each important decision have a clear owner?
- Did any ViewModel become the whole feature?
- Did any factory start doing policy?
- Did any view learn about infrastructure or app navigation?
- Are protocols meaningful?
- Are tests protecting behavior rather than implementation details?
- Does the code compile?
