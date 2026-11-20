# Changelog

## 0.2.0 (2026-11-20)

**Breaking:** Postgres runs on `postgres:18-alpine` (was `postgres:16-alpine`).
Postgres 18 can't open a Postgres 16 data directory, so existing databases
need a one-time dump and restore before or while moving to this version. A
database that hasn't been migrated stops with "database files are
incompatible with server" instead of starting empty. Follow
[docs/upgrading-to-postgres-18.md](docs/upgrading-to-postgres-18.md).

- The `wait-for-postgres` init container uses `postgres:18-alpine` too.
- Redis restarts with the `Recreate` strategy. Its volume is ReadWriteOnce, so
  with a rolling update a new pod on another node waited forever to attach it.

## 0.1.9 (2026-11-19)

- Redis runs on `redis:8-alpine` (was `redis:7-alpine`). Existing data carries
  over: Redis 8 loads the append-only files Redis 7 wrote. Idle memory rises
  from about 10 MB to about 24 MB because the image loads its bundled modules;
  the 64Mi limit still fits.

## 0.1.8 (2026-11-16)

- A published release, or a change to the README, `docs/`, `examples/` or
  `imlostincode.json` on main, asks imlostincode.com to rebuild, so the docs
  there stay current. No changes to the kit itself.

Earlier releases are listed on
[GitHub](https://github.com/NickMous/symfony-k8s-deployment-kit/releases).
