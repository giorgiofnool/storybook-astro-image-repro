# storybook-astro: local images break with Astro 7.4

Minimal reproduction: an empty Astro project (no adapter, no integrations) with one component rendering `<Image>` from `astro:assets` on an image in `src/assets`, and one story for it.

Versions: `astro` 7.4.0, `@storybook-astro/framework` 1.12.0, `storybook` 10.6.1, Node 22.22.2.

## Steps

```sh
npm install
npm run storybook
```

Open the `LocalImage` story and inspect the `<img>`.

## Result

| | Astro 7.3.5 | Astro 7.4.0 |
| --- | --- | --- |
| `<img src>` in `storybook dev` | `/@fs/…/src/assets/sample.png?origWidth=64&…` | `/_image?href=%2F%40fs%2F…sample.png…&w=64&h=64` |
| `GET` of that URL | 200 `image/png` | 404 |
| `<img src>` in `astro-prerendered-stories.json` (`storybook build`) | `/@fs/…` | `/_image?…` |

To compare, `npm install astro@7.3.5 --save-exact` and run again.

## Cause

storybook-astro sets `globalThis.astroAsset.imageService` to its own inline service before creating the Container (documented under "Image service injection" in [Version Compatibility](https://storybook-astro.org/how-it-works/version-compatibility/)), so `getImage()` skips `import("virtual:image-service")` and returns `/@fs/` URLs.

Astro 7.4.0 no longer reads that global. In `astro/dist/assets/internal.js`:

```js
// 7.3.5
async function getConfiguredImageService() {
  if (!globalThis?.astroAsset?.imageService) {
    const { default: service } = await import('virtual:image-service') /* … */;
    if (!globalThis.astroAsset) globalThis.astroAsset = {};
    globalThis.astroAsset.imageService = service;
    return service;
  }
  return globalThis.astroAsset.imageService;
}

// 7.4.0
let configuredImageService;
export async function getConfiguredImageService() {
  if (!configuredImageService) {
    const { default: service } = await import('virtual:image-service') /* … */;
    configuredImageService = service;
  }
  return configuredImageService;
}
/** Test-only: override the image service `getImage()` uses. */
export function setConfiguredImageService(service) { /* … */ }
```

So the injected service is ignored, the passthrough service from the generated config is used, and its `/_image` URLs are not served by Storybook.

A proof-of-concept fix is on the [`fix`](../../tree/fix) branch.
