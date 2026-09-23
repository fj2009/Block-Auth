# Error #3 — `pkill -f "hardhat node"` mató el propio shell de trabajo

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 1 (limpieza del entorno)
- **Contexto:** al parar el nodo local tras la verificación se ejecutó
  `pkill -f "hardhat node"`, y la sesión de shell murió con un timeout de 120 s.

## Síntoma

El comando `rm … Counter … && pkill -f "hardhat node"; ls …` no terminó: el shell quedó muerto
(timeout). Después, comandos posteriores parecían indicar que las carpetas `contracts/`,
`scripts/`, `test/` "no existían" (pista falsa: se ejecutaron desde un cwd distinto).

## Causa raíz

`pkill -f` usa coincidencia de patrón sobre **toda la línea de comandos** de cada proceso. El
shell que ejecutaba el comando tenía `"hardhat node"` literalmente en su propia línea de
comandos (`zsh -c '… pkill -f "hardhat node" …'`) → **el shell se mató a sí mismo** y mató el
resto de la tubería que venía detrás. El patrón siempre debe excluir el proceso que lo lanza.

## Solución aplicada

1. Verificar estado REAL (no suponer): chequear el puerto con `ss -ltnp | grep 8545`
   (estaba libre → el nodo sí se detuvo).
2. Confirmar que el árbol de ficheros seguía intacto desde el cwd correcto.
3. En lo sucesivo, parar procesos de forma inocua:
   ```bash
   # opción A: guardar el PID al lanzar el nodo
   npx hardhat node --port 8545 & echo $! > /tmp/opencode/hh-node.pid
   kill "$(cat /tmp/opencode/hh-node.pid)"
   # opción B (solo si es imprescindible el patrón): excluir el propio shell
   pkill -f "[h]ardhat node"
   ```

## Lección para la memoria

> Al automatizar la gestión de procesos en scripts hay que evitar `pkill -f` con patrones que
> el propio script contiene en su línea de comandos (peligro de *suicidio del proceso*). La
> práctica robusta es guardar el PID al lanzar el servicio y pararlo con `kill <pid>`.
> Además, tras un error conviene **verificar el estado real** (procesos/ficheros) en lugar de
> asumir pérdidas a partir de mensajes ambiguos.