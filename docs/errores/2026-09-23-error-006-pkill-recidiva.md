# Error #6 — `pkill -f` suicida el shell (REINCIDENCIA; caso real del Error #3)

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 2 (reinicio del nodo local)
- **Relacionado:** Error #3 — esta vez el patrón apareció en la propia línea de comandos dos veces

## Síntoma

`pkill -f hh-node` (o versiones con `"hardhat node"`) dentro de un comando cuyo texto también
contenía el patrón (p. ej. la ruta `/tmp/opencode/hh-node.log`) **terminó el shell que estaba
ejecutando el comando** (timeout de 60 s, sin salida). Es la **reincidencia** del Error #3.

## Causa raíz

`pkill -f` matchea la **línea de comandos completa** de cada proceso; la del propio shell
contenía el patrón buscado → el shell se mató a sí mismo antes de hacer nada más. El hábito
de `pkill -f` con patrones que el propio comando contiene es un anti-patrón.

## Lección / corrección definitiva aplicada

Tras la reincidencia se sustituyó el patrón por **gestión por PID** en dos pasos:

```bash
# lanzar el nodo guardando el PID en un fichero
nohup npx hardhat node --port 8545 >/tmp/opencode/hh-node.log 2>&1 & echo $! > /tmp/opencode/hh-node.pid

# pararlo SOLO por PID
kill "$(cat /tmp/opencode/hh-node.pid)"
```

Y al diagnosticar tras el fallo, **verificar el estado real** (procesos `pgrep -af`, puerto
`ss -ltnp`) en lugar de asumir — el nodo viejo seguía vivo ocupando el 8545.

## Lección para la memoria

> Los comandos de gestión de procesos deben discriminar por **PID** (no por patrón de línea de
> comandos). Además, la resolución de errores correcta empieza por **comprobar el estado real**:
> aquí el puerto seguía ocupado por el proceso no terminado, y un reinicio a ciegas habría
> ocultado la causa.