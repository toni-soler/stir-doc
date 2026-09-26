# Coordinación con Claude Code: extensibilidad comunitaria

Estado de entrega local, 2026-09-26. Esta nota no inicia trabajo remoto ni cambia el orden del roadmap STIR.

## Cambios disponibles por repositorio

| Repositorio | Rama local | Commit | Contenido |
|---|---|---|---|
| `stir-frontend` | `claude/community-catalog-consumer-mvp` | `6f296e5` | Cliente público inicial de catálogo y segunda presentación. |
| `stir-frontend` | `codex/community-catalog-next` | `0b3d9af` | Evolución aditiva: fotos autenticadas y segunda presentación actualizada. Contiene `6f296e5` en su historia. |
| `stir-main` | `codex/community-catalog-e2e` | `b4b28fb` | Composición opcional y pruebas reales de oferta/upgrade. |
| `stir-doc` | `claude/community-extension-guide-mvp` | `837501b` | Guía, preparación FreeFolk y validación del primer incremento. |
| `stir-doc` | `codex/community-extension-next` | rama actual | Resultados del segundo incremento y esta coordinación. |

Los commits son **locales** hasta que se integren/pushen; una VM remota no los verá por su nombre sin transportar las ramas. Las ramas `codex/*` viven en worktrees separados. El checkout normal de `stir-backend` contenía cambios sin confirmar de Ordinary Governance de Claude; Codex no los modificó. Para E2E se usó un worktree detached del backend en `381b156`, con build y migraciones apuntando al mismo commit.

## Instrucción breve para Claude Code

> Continúa tu trabajo actual de Ordinary Governance en `stir-backend`. Codex ha preparado, en ramas locales separadas, un contrato frontend de catálogo para distribuciones comunitarias (`stir-frontend` en `codex/community-catalog-next`, después de `6f296e5`), composición/E2E opcionales (`stir-main` en `codex/community-catalog-e2e`) y la guía (`stir-doc` en `codex/community-extension-next`). No cambies de rama ni limpies checkouts compartidos para inspeccionarlos; usa worktrees o lee los commits por hash. Al incorporar nuevas capacidades STIR, mantenlas en STIR y expón entradas públicas pequeñas para presentaciones distintas cuando haya un consumidor real. Respeta los límites STIR/osTRIS/Ledger y el contrato `CATALOG_CONTRACT_VERSION = 1`; cualquier cambio incompatible requiere otra versión y pruebas con ambas presentaciones. Coordina la integración de las ramas por repositorio cuando tu trabajo actual esté estable.

Antes de integrar, comprobar `git status` en cada repo, revisar diffs desde su propia base, ejecutar las pruebas de frontend y composición, y revalidar con el backend ya evolucionado por Claude. La evidencia actual cubre backend `381b156`, no la rama Ordinary Governance sin confirmar. No se pide a Claude repetir trabajo ni bloquear su incremento activo para esta integración.

FreeFolk sigue sin módulo de dominio propio. El primer recorrido específico y la comunidad destinataria deben decidirse antes de usar el DevKit para generarlo. El cliente de catálogo es un contrato inicial; no hay SDK de todas las pantallas ni garantía de que una UI reescrita herede automáticamente cambios de presentación.
