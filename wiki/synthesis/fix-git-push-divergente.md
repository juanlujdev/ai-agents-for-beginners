---
type: synthesis
date_updated: 2026-08-01
source_count: 1
---

# Fix: push rechazado tras hacer pull del upstream

## Síntoma

Commit local hecho, `pull` del repositorio remoto, y el `push` falla. La reacción intuitiva —"¿me cambio de rama para no tener el error?"— **no arregla nada**.

## Causa

No es la rama: `main` local y `origin/main` han **divergido**. En este caso 1 commit local (cambios en notebooks) frente a 16 commits nuevos del upstream. Git no puede hacer fast-forward entre dos historias que se han separado.

## El fix

```bash
git pull --rebase origin main
git push
```

`--rebase` reaplica el commit local encima de los del remoto, evitando un merge commit innecesario cuando el cambio propio es solo output de notebooks. Si hay conflicto (poco probable si el upstream toca otros ficheros), resolver y `git rebase --continue`.

## Contexto de este repo

Es un fork de un curso de Microsoft que sigue recibiendo commits. La divergencia va a repetirse cada vez que se sincronice con upstream tras trabajar en local. Trabajar en una rama propia (como `test_curso`) mantiene `main` limpio para sincronizar, pero no evita el rebase: solo lo mueve de sitio.

Fuentes: [[02-explore-agentic-frameworks]]
