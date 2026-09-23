# Error #1 — npm bloquea el `postinstall` de esbuild (política `allowScripts`)

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 1 (instalación del toolchain)
- **Contexto:** `npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox`

## Síntoma

```
npm warn install-scripts 1 package has install scripts not yet covered by allowScripts:
npm warn install-scripts   esbuild@0.28.2 (postinstall: node install.js)
```

npm 11 introdujo una política de seguridad que **bloquea los scripts de instalación
(`install`/`postinstall`) de dependencias no aprobadas**, para evitar ejecución de código
arbitrario en `npm install`. El paquete `esbuild` (dependencia del toolchain) no tenía su
`postinstall` autorizado.

## Causa raíz

- npm ≥ 11 **no ejecuta** `postinstall` de un paquete a menos que se apruebe explícitamente
  (`npm install-scripts approve <paquete>`) o se desactive la política.
- Para un TFG es una conducta **correcta y deseable**: protege frente a dependencias maliciosas.
- Técnicamente Hardhat 3.17 seguía funcionando sin el binario de esbuild, porque ese paquete no
  está en el *critical path* de compilación de Solidity; el problema latente aparecería con
  funciones del toolchain que sí lo usan (formateador, source maps).

## Solución aplicada

```bash
npm install-scripts approve esbuild     # aprueba solo ese paquete (no desactiva la política global)
npm install                              # reinstala y ejecuta el postinstall ya aprobado
```

Quedó registrado en `package.json`:

```json
"allowScripts": { "esbuild@0.28.2": true }
```

## Lección para la memoria (apartado "Dificultades encontradas")

> La instalación de dependencias de desarrollo puede verse bloqueada por nuevas políticas de
> seguridad de gestores de paquetes. La solución correcta no es desactivar la política, sino
> **aprobar de forma granular** el paquete que legítimamente necesita ejecutar su script de
> instalación, manteniendo la protección del resto del árbol de dependencias.