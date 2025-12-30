# CardNest — Offline-First Secure Trading Card Manager (Architecture Case Study)

- Role: Sole developer (with AI coding agent)
- Timeline: 2025-07 → 2025-10
- Status: Private repo, unpublished app (internal TestFlight used)

## Summary
Cross‑platform Flutter app for managing trading card collections with batch OCR, secure offline storage, and a Fastify/Node backend. The system emphasizes privacy and security (SQLCipher at rest, App Check + JWT in transit, SPKI/TLS pinning), with pricing aggregation, quota/rate controls, and encrypted backups.

## Architecture (At a Glance)

```mermaid
flowchart TD
    %% --- DARK MODE THEME DEFINITIONS ---
    %% 1. Force text to white (color:#ffffff)
    %% 2. Use deep, dark fills for nodes
    %% 3. Use bright, neon strokes for high contrast borders
    classDef client fill:#1a237e,stroke:#4fc3f7,stroke-width:2px,rx:10,ry:10,color:#ffffff;
    classDef edge fill:#b71c1c,stroke:#ffeb3b,stroke-width:2px,rx:5,ry:5,color:#ffffff;
    classDef backend fill:#004d40,stroke:#69f0ae,stroke-width:2px,rx:5,ry:5,color:#ffffff;
    classDef db fill:#4a148c,stroke:#ea80fc,stroke-width:2px,shape:cylinder,color:#ffffff;
    classDef external fill:#37474f,stroke:#ffab91,stroke-width:1px,stroke-dasharray: 5 5,color:#ffffff;

    %% Force connecting lines to be white for visibility
    linkStyle default stroke:#ffffff,stroke-width:2px;

    %% --- 3rd Party Services Container ---
    subgraph ExternalServices ["☁️ 3rd Party Services"]
        direction LR
        Firebase["🔥 Firebase Auth & App Check"]
        RC["💰 RevenueCat (IAP)"]
    end
    %% Style container: Very dark grey fill, bright neon orange border
    style ExternalServices fill:#121212,stroke:#ffab91,stroke-width:2px,color:#ffffff

    %% --- Client Side Container ---
    subgraph ClientDevice ["📱 Mobile Device"]
        FlutterClient["Flutter Client App
        -------------------
        • Batch OCR Pipeline (Resize/Decode/Chunking)
        • SQLCipher DB (AES-256)
        • TLS Pinning (SPKI)
        • IAP Entitlement Gating"]
    end
    %% Style container: Very dark grey fill, bright neon blue border
    style ClientDevice fill:#121212,stroke:#4fc3f7,stroke-width:2px,color:#ffffff

    %% --- Edge Layer Container ---
    subgraph EdgeLayer ["🌐 Cloudflare Edge"]
        CFProxy["Reverse Proxy
        (TLS Termination for Public Hostname)"]
    end
    %% Style container: Very dark grey fill, bright neon yellow border
    style EdgeLayer fill:#121212,stroke:#ffeb3b,stroke-width:2px,color:#ffffff

    %% --- Backend Infrastructure Container ---
    subgraph ServerInfra ["🔒 Private Server - Railway Infrastructure"]
        direction TB
        
        %% Internal subgraphs set to pure black for depth
        subgraph Tunneling ["Ingress Layer"]
            Cloudflared["cloudflared Daemon
            (Server-side Tunnel)"]
        end
        style Tunneling fill:#000000,stroke:#69f0ae,stroke-width:1px,color:#ffffff

        subgraph AppLayer ["Application Layer"]
            Fastify["🟢 Fastify/Node Backend
            -------------------
            • Binds 127.0.0.1:PORT (No Public IP)
            • OCR Endpoint (Quotas/Concurrency)
            • Pricing & Search Logic
            • Abuse Detection & Server-Timing"]
        end
        style AppLayer fill:#000000,stroke:#69f0ae,stroke-width:1px,color:#ffffff
        
        DB[("🐘 Postgres
        Primary App Database")]
    end
    %% Style container: Very dark grey fill, bright neon green border
    style ServerInfra fill:#121212,stroke:#69f0ae,stroke-width:2px,color:#ffffff

    %% --- Relationships & Data Flow ---
    %% Note: Line labels might still vary depending on the renderer used
    
    %% Client -> Edge
    FlutterClient -- "HTTPS (JWT + App Check)
    Target: API_BASE_URL" --> CFProxy
    
    %% Edge -> Tunnel
    CFProxy -- "Secure Tunnel" --> Cloudflared
    
    %% Tunnel -> Backend
    Cloudflared -- "Localhost Traffic
    http://127.0.0.1:PORT" --> Fastify
    
    %% Backend -> DB
    Fastify -- "Private Networking" --> DB
    
    %% External Integrations (Client SDKs)
    FlutterClient -. "SDK Auth / Tokens" .-> Firebase
    FlutterClient -. "Purchase Sync" .-> RC
    
    %% External Integrations (Backend Verification)
    Fastify -. "Verify Token / App Check" .-> Firebase
    Fastify -. "Sync Entitlements / Webhooks" .-> RC

    %% Apply Styles
    class FlutterClient client;
    class CFProxy edge;
    class Fastify,Cloudflared backend;
    class DB db;
    class Firebase,RC external;
```

## Key Problems → Solutions

- OCR throughput under variable mobile networks (distinct from accuracy)
  - Solution: Client pipeline resizes images to a 1536px long‑side cap (native HEIC/AVIF decode; JPEG quality tuning), then sends chunked batches (5 images/chunk) with up to 4 parallel requests. Backend enforces per‑user inflight caps (max 20 images) and processes images concurrently per batch (tier‑based concurrency, env‑tunable). Progressive timeouts and limited retries keep tail latency bounded; request body capped to prevent oversized uploads.
  - Impact: Single card ~4–8s; 20‑card batch ~20–40s at ~20–30 Mbps; fewer timeouts; predictable end‑to‑end latency under load.

- Recognition accuracy and duplicate prevention (separate from throughput)
  - Solution: Structured extraction (JSON mode) feeding candidate retrieval with lightweight heuristic scoring; optional vision re‑ranking among top‑K image candidates with confidence gating; exact‑match DB lookup by `langId`; and “withhold on ambiguity” to avoid incorrect saves. Local DB dedupe prevents duplicates; user‑facing messages localize OCR errors.
  - Impact: Precision ≈ 99%, Recall ≈ 97% on internal test set; reduced dupes and clearer UX on ambiguous results.

- Multilingual data normalization and search quality
  - Solution: Offline ingestion and normalization of card data across languages with deterministic canonical IDs, conflict detection/resolution rules, and idempotent, repeatable scripts/tests to keep the dataset consistent.
  - Impact: Higher‑quality OCR candidate sets and better search recall/precision across languages; fewer mismatches and cleaner UX.

- Security across client, network, and server
  - Solution: Client — SQLCipher‑encrypted DB (AES‑256), secure PIN (PBKDF2), TLS pinning, env‑based config (no secrets in repo). Network — Cloudflare Edge + server‑side Tunnel (no public ingress), strict CORS, hardened security headers. Server — Firebase Auth + App Check verification, per‑user quotas and concurrency limits, request size caps, suspicious‑usage flags, and structured logging. Backups — encrypted every 4 hours with 7/4/12 retention and scripted restore.
  - Impact: Reduced attack surface, protected user data at rest/in transit, predictable resource usage under load, and fast recovery (RPO ≈ 4h).

## Results & Metrics (internal testing)
- OCR accuracy: Precision ≈ 99%, Recall ≈ 97% (test set) 
- Latency: Single card ~4–8s; 20‑card batch ~20–40s with ~20–30 Mbps uplink
- Backups: Encrypted every 4 hours; 7/4/12 retention strategy (daily/weekly/monthly)
- Distribution: Internal TestFlight builds used for testing

## Security Posture
- Data at rest: SQLCipher (AES‑256), generated key material, schema migrations
- Data in transit: TLS pinning (SPKI), Firebase Auth + App Check verification on backend
- Edge/tunnel trust: Backend binds to 127.0.0.1 and is reachable only via a server‑side Cloudflare Tunnel; client connects to a Cloudflare‑proxied hostname. Strict CORS allowlist and optional edge/tunnel header checks further constrain origin access.
- Secrets/config: Build‑time environment values (no secrets in repo)
- Backups: Client‑side GPG AES‑256 encryption; tiered retention; restore scripts
- Headers/hardening: Strict response security headers and CORS whitelist

## UI/UX Highlights
- Batch OCR flow: camera/upload → client‑side preprocessing → results with exact match/high‑confidence picks, clear “withheld/ambiguous” states, and localized error messages. Progress and quota state surfaced to set expectations.
- Condition/grade workflows: interactive drawer for condition selection (incl. graded services), multi‑quantity, and live market value summary; consistent dialogs for grading details and edits.
- Collections management: reusable Advanced Filter dialog, filter chips, bulk actions, and at‑a‑glance collection stats; polished list/detail screens.
- Theming and consistency: custom theme extensions (`lib/core/theme/*`) and design tokens drive colors/typography; cohesive “card” visual style across screens.
- Premium gating: entitlement‑aware UI (paywall and feature guards) that appears only when enabled via feature flags; seamless transitions between free/Plus/Pro experiences.

## Deployment & Ops
- CI/CD: Documented iOS builds (unsigned/signed IPA) with optional TestFlight upload
- Runtime: Backend deployed with private networking and exposed only via a secure tunnel (no public ingress)
- Observability: Server‑Timing headers, structured logs, and optional DB timing logs
- Backups/Restore: Automated encrypted backups and one‑command restore

## Tech Stack
- Flutter/Dart, Firebase Auth/App Check, SQLCipher (sqflite_sqlcipher)
- Fastify/Node.js, Postgres, RevenueCat
- Cloudflare Tunnel, Railway, Backblaze B2 (encrypted backups)

## Screenshots (to be inserted before export)
Include 3–5 images with short captions (blur any sensitive data):
- OCR capture → result match
- Condition/grade drawer and market summary
- Collections list with filters/analytics
- Pricing/entitlement UI (if relevant)

## Future Work / Status
- App completeness ≈ 90%
- Remaining: Automated card info/price update script, app store review/publishing steps
- Project paused for now

## Access
- Private repository and unpublished app
- Demo/video and selective code excerpts available on request
- Internal TestFlight build available for reviewers (by invite)
