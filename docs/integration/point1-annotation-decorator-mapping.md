# 05a9da1d-82eb-4129-9391-ae1cbb89d59a — Point #1: Annotation/Decorator Mapping & Migration Strategy for FractalTFX

## Objective
Map JacpFX annotations and lifecycle semantics into Pandino's TypeScript decorators, ensuring compatibility with Script4JFX and browser/Electron environments.

### Mapping Table
| JacpFX Annotation | Pandino Decorator | Notes |
|------------------|-------------------|-------|
| `@Perspective`   | `@Component` + `@Service` | Perspective service bundle |
| `@View`          | `@Component` + `@Service` | View factory or node provider |
| `@Component`     | `@Component` + `@Service` | Core functional service |
| `@PostConstruct` | `@Activate`              | Initialization after DI |
| `@PreDestroy`    | `@Deactivate`            | Cleanup before removal |
| `@Resource`      | `@Reference`             | Injected Pandino service |
| `@Stateless`     | Scope metadata           | Implement via `scope: 'singleton'` |

### Migration Steps
1. Identify all JacpFX annotations in FractalTFX code.
2. Replace with Pandino decorators while preserving metadata.
3. Add lifecycle methods for SCR activation/deactivation.
4. Inject dependencies via `@Reference` rather than manual lookups.
5. Ensure decorators emit metadata (`emitDecoratorMetadata: true`).
