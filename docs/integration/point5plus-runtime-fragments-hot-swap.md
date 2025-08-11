# 8864089e-8645-4ffe-baa0-394bbff31adc — Points #5+: Runtime Adaptability, Fragments, Theming & Hot-Swapping for FractalTFX

## Runtime Adaptability
- Use SCR dynamic service binding to swap implementations at runtime.
- Define service ranking to prefer higher-priority versions.
- Implement lazy-loading for heavy perspectives/components.

## Fragments & Theming
- Use Pandino fragment bundles to deliver CSS, HTML, i18n files.
- Attach fragments to core bundles without modifying them.
- Merge fragment resources at runtime via `FragmentResourceProcessor`.

## Hot-Swapping
- Monitor bundle lifecycle events to replace or remove components live.
- Ensure `@Deactivate` properly cleans up DOM and event listeners.

## Example: Theme Fragment Structure
```
fragment.theme.dark/
  ├─ css/theme.css
  ├─ i18n/en.json
  └─ fragment.json
```
