# Story 1.2: Multi-Stage Dockerfile and nginx.conf

Status: done

## Story

As a platform operator,
I want a multi-stage Docker build that produces a minimal nginx-served image,
so that the portal can be built and deployed without Node.js in the runtime container.

## Acceptance Criteria

1. `docker build -t terminus-portal:test .` completes successfully
2. Final image stage is based on `nginx:1.27-alpine`
3. `docker run --rm -p 8080:80 terminus-portal:test` serves app at `http://localhost:8080`
4. `curl -s http://localhost:8080/nonexistent` returns 200 (SPA fallback via `index.html`)
5. Response headers do NOT include `Server: nginx` or version info (`server_tokens off`)
6. Static assets (`.js`, `.css`) return `Cache-Control: public, immutable`

## Tasks / Subtasks

- [ ] Task 1: Write test script for Docker build (TDD — write assertions first) (AC: #1–#6)
  - [ ] Create `scripts/test-docker.sh` that runs the build, starts container, and curls assertions
  - [ ] Verify the script reports failures against the NOT-YET-updated Dockerfile
- [ ] Task 2: Rewrite Dockerfile as multi-stage (AC: #1, #2)
  - [ ] Stage 1 (`builder`): `FROM node:20-alpine` — `COPY package*.json .`, `RUN npm ci`, `COPY . .`, `RUN npm run build`
  - [ ] Stage 2 (`runtime`): `FROM nginx:1.27-alpine` — `COPY --from=builder /app/dist /usr/share/nginx/html`
  - [ ] Copy `nginx.conf` into stage 2
- [ ] Task 3: Create `nginx.conf` (AC: #4, #5, #6)
  - [ ] `try_files $uri $uri/ /index.html` for SPA routing
  - [ ] `server_tokens off`
  - [ ] `location ~* \.(js|css|png|svg|ico)$` → `add_header Cache-Control "public, immutable, max-age=31536000";`
- [ ] Task 4: Run test script — verify all assertions pass (AC: #1–#6)

## Dev Notes

**Target repo:** `TargetProjects/terminus/portal/terminus-portal`
**Base images:** `node:20-alpine` (build) + `nginx:1.27-alpine` (runtime)
**Important:** The existing `Dockerfile` in the repo uses nginx but is not multi-stage. Replace it entirely.

nginx.conf key settings:
```nginx
server {
    listen 80;
    server_tokens off;
    root /usr/share/nginx/html;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    location ~* \.(js|css|png|svg|ico|woff2?)$ {
        add_header Cache-Control "public, immutable, max-age=31536000";
    }
}
```

### Project Structure Notes

Files to create/modify:
```
terminus-portal/
├── Dockerfile        (replace existing)
├── nginx.conf        (new)
└── scripts/
    └── test-docker.sh (new — test helper, not committed to prod)
```

### References

- NFR4 (multi-stage Dockerfile): `docs/terminus/portal/portal2/epics.md`
- TR6 (nginx SPA routing, server_tokens): `docs/terminus/portal/portal2/epics.md`
- Architecture: `docs/terminus/portal/portal2/architecture.md` — Section "Deployment"

## Dev Agent Record

### Agent Model Used

### Debug Log References

### Completion Notes List

### File List
