# 6eec91a0-1ed8-4e6d-a9cf-213aa1b14787 — Points #3 & #4: Messaging & ConfigAdmin Integration for FractalTFX on Pandino

## Messaging Integration
- Replace JacpFX message bus with Pandino's `EventAdmin` service.
- Use Whiteboard pattern for decoupled event consumers.
- Components declare event topics via metadata and auto-register with EventAdmin.
- Map JacpFX message routing rules to Pandino `Event` object properties.

### Example Event Consumer
```ts
@Component()
@Service({ interfaces: ['EventHandler'] })
class MyHandler implements EventHandler {
  handleEvent(event: Event): void {
    console.log('Received event', event.getTopic());
  }
}
```

## ConfigAdmin Integration
- Map JacpFX configuration files/props to Pandino `ConfigurationAdmin` PIDs.
- Inject configuration into services using `@Reference`.
- Watch for config changes to dynamically update UI/services without reload.

### Example Config Injection
```ts
@Reference({ interface: 'ConfigurationAdmin' })
private configAdmin?: ConfigurationAdmin;
```
