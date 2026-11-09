# Spring Boot testing skills — Elysium / Opereta selection

Fork of [Kehrlann/spring-boot-testing-skills](https://github.com/Kehrlann/spring-boot-testing-skills)
(agent skills for testing Spring Boot 4 / Spring Framework 7 apps, distilled from the book
*Spring Boot Testing In Action*), used as a **storage of agent skills** (Claude format)
for the Elysium Java base and for the Opereta application.

## Branches

| Branch | Content |
|---|---|
| `main` | exact mirror of the upstream `main`; never committed to |
| `elysium-stack` | skills selected and adapted for the Elysium Java base |
| `opereta-stack` | `elysium-stack` plus what is specific to Opereta |

Changes flow one way: upstream → `main` → `elysium-stack` → `opereta-stack` (by merge).
Fixes that are useful for everyone go to a topic branch from `main` and upstream as a pull request.

## Layout

- `.claude/skills/<skill>/` — the skills in use.
- `_unused/` — everything else from upstream, moved (not deleted) so that upstream changes keep merging:
  `_unused/skills/` (skills not selected), `_unused/root/` (original README and repository files).

Moves are always separate commits without content changes; content changes follow in their own commits.

## License

Apache License 2.0, see [LICENSE](LICENSE). Original work by Daniel Garnier-Moiroux;
the original README is kept in `_unused/root/README.md`.
