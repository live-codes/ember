# Taste

## Architecture & approach
- Prefers client-side-only solutions that run code entirely in the browser with no server ("run code in the browser with no servers"); favors CDN-loaded runtime deps over any server-side compilation or backend. Confidence: 0.7
- Prefers starting new framework/language integrations with a minimal, self-contained proof-of-concept (a single simple HTML page that can run code) before building out the full integration. Confidence: 0.7
- Keeps standalone framework POCs as one directory per framework at the repo root, each containing just `index.html` + `README.md` with no build step, runnable straight from `file://` (matching existing sibling POCs like qwik/marko/remix/angular). Confidence: 0.55
