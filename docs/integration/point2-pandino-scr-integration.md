# Point #2 — Integration of FractalTFX → Script4J on top of Pandino & Pandino SCR

This document restates **Point #2 (Bundle modularity)** but reframes the thinking **as an integration of FractalTFX (the TypeScript port of JacpFX) built on top of the Pandino framework and its Service Component Runtime (SCR)**. It focuses on concrete patterns, lifecycle mappings, and practical steps to make FractalTFX a first-class Pandino modular app.

---

## Goal
Run FractalTFX inside a Pandino-managed runtime so that the UI, services, and lifecycle are delivered as modular bundles, with SCR handling the declarative component registration, activation, and dependency injection. This yields hot-swapping, lazy-loading, fragments for theming/locales, and an OSGi-like development model in browser/Electron.

---

## High-level Mapping (FractalTFX → Pandino/SCR)

- **Workbench** → Pandino **root bundle** (bootstrapping bundle + top-level activator).
- **Perspective** → Pandino **bundle** or **component** (exporting perspective service(s) or view factories).
- **Component / ManagedFragment** → Pandino **components/services** registered via decorators and managed by SCR.
- **Messaging layer** (actor-like mailbox) → Pandino **EventAdmin** or a dedicated `MessageBus` service tracked by SCR.
- **FXML / HTMLLoader** → `script4jfx.loader.HTMLLoader` invoked by SCR-activated UI component factories or by bundle activators.
- **Annotations** (`@View`, `@Perspective`, `@PostConstruct`, `@Resource`, `@Stateless`) → mapped to Pandino decorators (`@Component`, `@Service`, `@Reference`, `@Activate`, `@Deactivate`, etc.).

---

## Bundle Boundaries & Suggested Layout

1. **core.workbench** (root bundle)
   - Activator: starts Pandino runtime hooks for FractalTFX, registers core services (MessageBus, UIManager).
   - Registers Workbench service and initial Perspective wiring.
2. **perspective.<name>** (one per major view)
   - Contains perspective view factory, registers a Perspective service.
   - Exposes render target IDs via metadata.
3. **component.<name>** (per FractalTFX component)
   - Declares component class decorated with `@Component` / `@Service` and registered with SCR.
   - `handle()`/`postHandle()` executors wired to the MessageBus.
4. **services.* (supporting bundles)**
   - Messaging bundle (EventAdmin wrapper), background executor bundle, config bundle (ConfigAdmin), logging bundle.
5. **fragments** (themes, locale packs, platform-specific impls)
   - Attach to host bundles via `fragmentHost` to augment resources (CSS, JSON, components).

---

## SCR Usage & Lifecycle Mapping

- **Component Declaration**: FractalTFX component classes will carry decorator metadata (via `@pandino/decorators`) that SCR can consume. Use `Service` metadata to declare interfaces (e.g., `PerspectiveService`, `ComponentService`).
- **Registration**: On bundle start, either rely on SCR auto-registration (if components are listed in bundle manifest) or explicitly call:
  ```ts
  const scrRef = context.getServiceReference<ServiceComponentRuntime>('ServiceComponentRuntime')!;
  const scr = context.getService(scrRef)!;
  await scr.registerComponent(MyFractalComponent, bundleId);
  ```
- **Activation Hooks**: Map JacpFX lifecycle annotations:
  - `@PostConstruct` → Pandino `@Activate` (or `Activate` method invoked when SCR activates component).
  - `@PreDestroy` → Pandino `@Deactivate`.
- **handle/postHandle**:
  - `handle()` (worker-thread logic) → run inside an async worker pool or Web Worker, provided by a `BackgroundExecutor` service in Pandino. SCR can inject this executor via `@Reference`.
  - `postHandle()` (UI-thread) → schedule via `script4jfx.application.Platform.runLater()` or the Script4J Platform shim, ensuring DOM-safe updates.

---

## HTMLLoader & Controller Injection with SCR

- SCR-activated components that are UI-backed should expose a **view factory** service returning a `Node`:
  ```ts
  @Component({ name: 'component.user' })
  @Service({ interfaces: ['ComponentViewFactory'] })
  class UserComponentFactory {
    createView(): Node {
      const ctrl = new UserController();
      htmlLoader.setController(ctrl);
      return htmlLoader.load(document.getElementById('...'));
    }
  }
  ```
- Use Pandino's reflection helpers (`getDecoratorInfo()`) to locate metadata for `@HTML` property injection so HTMLLoader can populate controller fields before activation.
- For Declarative (HTML) views, the fragment/resource map can provide the HTML snippet URL and CSS resources that the factory will load.

---

## Messaging & Event Lifecycle on Pandino

- Implement a `MessageBus` as a Pandino `@Service` (or adapt `EventAdmin`) to deliver messages between components.
- Each registered FractalTFX component may get a `MessageBox` injected:
  ```ts
  @Reference({ interface: 'MessageBus' })
  private messageBus?: MessageBus;
  ```
- Message processing lifecycle:
  1. Incoming message → `MessageBus` routes to the target component's queue.
  2. SCR-managed component picks the message for `handle()` on a `BackgroundExecutor` service.
  3. When `handle()` returns a Node or data, `postHandle()` is invoked via `Platform.runLater()` to update UI.

---

## Fragments & Resource Processors

- Use Pandino fragments for theming, i18n, and platform-specific overrides.
- Implement `FragmentResourceProcessor` to merge fragment resources (CSS, JSON bundles) into the host bundle at attach-time.
- Host bundles can call `bundle.getResource('assets/theme.css')` to resolve fragment-provided assets.

---

## Hot-Swap, Lazy-Load & Versioning

- Use Pandino's dynamic bundle lifecycle & `ServiceTracker` for hot replacement and detection of new implementations.
- Lazy-load heavy perspective or component bundles using dynamic `import()` and then `scr.registerComponent(...)` once module is ready.
- Use `service.ranking` metadata to prefer newer or higher-priority implementations.

---

## Packaging & Build Notes

- Each bundle remains a dist artifact with `export default { headers, activator, components }` manifest, suitable for Pandino loader.
- Build pipeline (vite/rollup) should produce hashed assets and populate `resources` map used by fragments.
- Ensure `tsconfig.json` includes `experimentalDecorators` and `emitDecoratorMetadata` for decorators to work.

---

## Testing & Migration Strategy (practical rollout)

1. **Core services** (MessageBus, BackgroundExecutor, Platform shim) as first bundles.
2. **Workbench bundle** to bootstrap and show a minimal perspective.
3. **One perspective bundle** with **one component bundle** — verify lifecycle, messaging, and UI injection.
4. Incrementally add perspectives and components; convert FXML-backed views to HTMLLoader-based views.
5. Add fragments (theme/locale) and test resource merging.

---

## Caveats & Considerations

- Java's true multi-threading differs from JS; use Web Workers or async pools for background `handle()` processing but accept some limitations (no shared memory).
- Watch for memory leaks: SCR registration/unregistration and DOM nodes must be cleaned on `@Deactivate`.
- Security (CSP), module resolution differences in different bundlers (vite/esbuild), and browser/Electron platform APIs must be handled explicitly in respective bundles.

---

## Example: Activator registering a component with SCR

```ts
const activator: BundleActivator = {
  async start(context: BundleContext) {
    const scrRef = context.getServiceReference<ServiceComponentRuntime>('ServiceComponentRuntime')!;
    const scr = context.getService(scrRef)!;
    const bundleId = context.getBundle().getBundleId();

    await scr.registerComponent(UserComponent, bundleId);
  },

  async stop(context: BundleContext) {
    // SCR will handle deregistration if components were registered via scr
  }
};
```

---

## Next Steps (for implementation doc)
- Update `integration-with-pandino.md` to reflect this SCR-first integration blueprint for FractalTFX.
- Create example bundle templates (workbench, perspective, component, fragment) in a sample repository.
- Implement `MessageBus` + `BackgroundExecutor` services in Pandino as foundational bundles for FractalTFX.
