---
name: deploying-payload-to-cloudflare
description: Use when deploying Payload CMS to Cloudflare Workers to resolve bundle size limits (3 MiB) and dependency issues (sharp, @vercel/og) - provides a pruning and patching strategy
---

# Deploying Payload CMS to Cloudflare Workers

## Overview

Deploying Payload CMS to Cloudflare Workers often hits the 3 MiB bundle size limit and encounters issues with native dependencies like `sharp` and `@vercel/og`. This skill provides a proven strategy to overcome these limitations by pruning unused files and patching imports.

## When to Use

- **Deploying Payload CMS** to Cloudflare Workers.
- **Encountering "Bundle size exceeds 3 MiB"** errors.
- **Encountering "Could not resolve"** errors for `sharp`, `resvg.wasm`, or `yoga.wasm`.

## The Strategy

1.  **Build**: Use `@opennextjs/cloudflare` to build the worker.
2.  **Prune**: Remove the massive `node_modules` directory from the output. The worker bundle (`handler.mjs`) already contains the necessary code (mostly).
3.  **Patch**: Replace absolute paths to missing native modules (which were in the pruned `node_modules`) with an empty module stub.

## Risks & Trade-offs

By removing `node_modules`, you are forcing the Worker to rely **100% on the bundled code**.

- **Native Modules will fail**: Libraries like `sharp` (C++ binary) cannot be bundled. They _must_ exist on disk. Removing `node_modules` breaks them.
  - _Mitigation_: Disable features relying on them (e.g., `images: { unoptimized: true }`) or use external services (e.g., Cloudflare Images).
- **Dynamic Imports may fail**: If code uses `await import('some-lib')` and that lib wasn't bundled, it will fail at runtime.
  - _Mitigation_: Ensure `output: 'standalone'` is used, which is good at tracing dependencies. Test critical paths.

## Implementation

### 1. Create `build-worker.sh` Script

Create a script (e.g., `scripts/build-worker.sh`) with the following content. This script handles building, pruning, and patching to ensure the worker fits within Cloudflare's limits.

```bash
#!/bin/bash
set -e

# 1. Build
echo "Building Next.js app..."
npm run build
echo "Building OpenNext worker..."
npx opennextjs-cloudflare build

# 2. Prune node_modules
# The worker bundle is self-contained enough that we don't need the full node_modules
# But we need to keep some externalized packages like graphql
echo "Pruning node_modules..."
rm -rf .open-next/server-functions/default/node_modules
mkdir -p .open-next/server-functions/default/node_modules
cp -r node_modules/graphql .open-next/server-functions/default/node_modules/
cp -r node_modules/prettier .open-next/server-functions/default/node_modules/ || true

rm -rf .open-next/server-functions/default/backend/.next/cache

# 3. Create Empty Stubs
echo "Creating stubs..."
# Stub for WASM modules
# Note: We cannot use new WebAssembly.Module() as it may be disallowed in some environments
echo "export default {};" > .open-next/server-functions/default/empty_wasm.js
# Stub for JS libraries (prettier, sharp, etc.)
echo "export default {};" > .open-next/server-functions/default/empty_lib.js

# 4. Patch handler.mjs
echo "Patching handler.mjs..."

# Detect handler path (CI vs Local structure)
if [ -f ".open-next/server-functions/default/backend/handler.mjs" ]; then
  HANDLER_PATH=".open-next/server-functions/default/backend/handler.mjs"
else
  HANDLER_PATH=".open-next/server-functions/default/handler.mjs"
fi

if [ ! -f "$HANDLER_PATH" ]; then
  echo "Error: $HANDLER_PATH not found!"
  exit 1
fi

# Replace absolute paths to WASM files with empty_wasm.js
# Matches both single and double quotes, and optional ?module suffix
perl -pi -e 's|(["'"'"'])[^"'"'"']*node_modules/[^"'"'"']+\.wasm(\?module)?\1|\1./empty_wasm.js\1|g' "$HANDLER_PATH"

# Stub .ttf.bin files (often used by @vercel/og)
perl -pi -e 's|(["'"'"'])[^"'"'"']*node_modules/[^"'"'"']+\.ttf\.bin\1|\1./empty_lib.js\1|g' "$HANDLER_PATH"

# Patch specific problematic libraries (prettier, sharp)
perl -pi -e 's|require\("prettier"\)|require("./empty_lib.js")|g' "$HANDLER_PATH"
perl -pi -e 's|import\("prettier"\)|import("./empty_lib.js")|g' "$HANDLER_PATH"
perl -pi -e 's|require\("sharp"\)|require("./empty_lib.js")|g' "$HANDLER_PATH"
perl -pi -e 's|import\("sharp"\)|import("./empty_lib.js")|g' "$HANDLER_PATH"

# Fix any double-slash issues if they occur
perl -pi -e 's|"/./empty_wasm.js"|"\./empty_wasm.js"|g' "$HANDLER_PATH"
perl -pi -e 's|"/./empty_lib.js"|"\./empty_lib.js"|g' "$HANDLER_PATH"

echo "Build and optimization complete."
```

### 2. Configure `next.config.ts`

Alias problematic dependencies to an empty file during the Next.js build to prevent them from bloating the initial bundle or causing build errors.

```typescript
// next.config.ts
import { withPayload } from "@payloadcms/next/withPayload";
import type { NextConfig } from "next";

const nextConfig: NextConfig = {
  output: "standalone",
  images: {
    unoptimized: true, // Cloudflare handles images via a separate worker or R2
  },
  webpack: (config) => {
    // Alias sharp and @vercel/og to empty to avoid bundling them
    config.resolve.alias.sharp = "./empty.js";
    config.resolve.alias["@vercel/og"] = "./empty.js";
    return config;
  },
};

export default withPayload(nextConfig);
```

### 3. CI/CD Integration

Use Cloudflare's native CI/CD integration by connecting your repository in the Cloudflare Dashboard.

- **Build Command**: `bash scripts/build-worker.sh` (or `npm run build:worker`)
- **Build Output Directory**: `.open-next/server-functions/default`

## Common Mistakes

- **Not Pruning**: Leaving `node_modules` will almost always exceed the 3 MiB limit.
- **Not Patching**: The generated `handler.mjs` will contain absolute paths to the machine where it was built. These paths will fail at runtime on Cloudflare.
- **Using `pnpm` without `shamefully-hoist`**: Sometimes `pnpm` structure makes pruning harder. `npm` is often simpler for this specific pruning strategy.

## References

- [Payload CMS Cloudflare Template](https://github.com/payloadcms/payload/tree/main/templates/with-cloudflare-d1)
- [OpenNext Cloudflare](https://opennext.js.org/cloudflare)
