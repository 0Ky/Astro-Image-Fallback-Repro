# Astro Image Fallback Reproduction

Minimal reproduction for a failed remote `<Image>` optimization causing an Astro static build to exit unsuccessfully.

## Reproduction

Clone the repository, install the dependencies and run the build:

1. `git clone https://github.com/0Ky/Astro-Image-Fallback-Repro.git`

2. `cd Astro-Image-Fallback-Repro && npm install`

3. `npm run build`

The build reaches the optimized-image generation step and then fails when Astro attempts to fetch the unreachable remote image:

```js
generating optimized images
[WARN] [build] Unable to generate optimized image for https://cdn.example.site/image.png: TypeError: fetch failed
Error generating image for https://cdn.example.site/image.png: TypeError: fetch failed
```

The process exits with a non-zero status:

```js
error: script "build" exited with code 1
```

The important part is that the Astro build **does not complete successfully**. There is no successful build completion message and the post-build command is not executed.

The repository uses the following postbuild command to make this observable:

```json
"postbuild": "node -e \"console.log('--> POST BUILD WORK RAN <--')\"",

```

## Control Test

To verify that the remote image is responsible for the failure, comment out the `<Image>` component in `src/pages/index.astro`:

```astro
<!-- <Image src="https://cdn.example.site/image.png" alt="Example Image" width="80" height="80" /> -->
```

Run the build again:

```sh
npm run build
```

Without the remote `<Image>`, the build completes successfully and the post-build command runs.

You should see the normal successful Astro build output followed by:

```text
--> POST BUILD WORK RAN <--
```

This provides a control case showing that the build failure is triggered by the remote `<Image>` optimization.

## What This Demonstrates

The reproduction demonstrates the following sequence:

```text
<Image> with unreachable remote URL
        │
        ▼
Astro attempts remote image optimization
        │
        ▼
Remote image fetch fails
        │
        ▼
astro build exits with code 1
        │
        ├── No successful build completion
        │
        └── Post-build command is not executed
```

With the `<Image>` removed:

```text
No remote image optimization
        │
        ▼
astro build completes successfully
        │
        ▼
Post-build command runs
```

The remote image is non-critical content, but its failed build-time retrieval prevents the overall static build from completing.

## Project Structure

```text
/
├── public/
├── src/
│   └── pages/
│       └── index.astro
├── astro.config.mjs
└── package.json
```

The reproduction intentionally contains no adapters, integrations, or other application-specific dependencies.

## Expected Behavior

A failure to retrieve an individual non-critical remote image should ideally not require the entire static build to fail.

A useful API could allow a local fallback image to be specified for a remote source, for example:

```astro
<Image
    src="https://cdn.example.site/image.png"
    fallback="/assets/avatar-placeholder.svg"
    alt="Avatar"
    width="80"
    height="80"
/>
```

If the remote image cannot be fetched or optimized during the build, Astro could use the local fallback and continue building the rest of the site.

Alternatively, Astro could provide another mechanism for applications to handle remote image optimization failures without having to reimplement the image-fetching and optimization pipeline.