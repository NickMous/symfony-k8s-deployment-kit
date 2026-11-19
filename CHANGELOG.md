# Changelog

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
