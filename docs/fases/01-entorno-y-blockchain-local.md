# Fase 1 — Entorno de Desarrollo y Blockchain Local

> **Fecha:** 2026-09-23 · **Módulo:** 1 de 5 · **Estado:** ✅ completado

## 1. Objetivo técnico

Disponer de una **red Ethereum local de coste cero** sobre la que desplegar y probar los
contratos del sistema, un monorepo ordenado que permita crecer módulo a módulo, y un primer
contrato (`IdentityRegistry`) funcionando y verificable de extremo a extremo.

Este módulo NO es una trivialidad: es la base sobre la que el tribunal comprobará que el
sistema se ejecuta **sin depender de internet ni de inversión económica** (gas = 0).

## 2. Herramientas utilizadas

| Herramienta | Versión | Rol en el proyecto |
|---|---|---|
| Node.js / npm | 24.19.0 / 11.19.0 | Ejecución del toolchain |
| Hardhat | **3.17.0** (v3, última mayor) | Framework de desarrollo Ethereum local |
| Hardhat Toolbox (mocha-ethers) | 7.0.0 | Plugins de testing (Mocha + Chai + ethers) |
| solc | 0.8.34 | Compilador de Solidity |
| Hardhat Ignition | 3.1.8 | Despliegue declarativo y reproducible |

> **Nota para la memoria:** se ha usado **Hardhat 3**, que en el momento de redactar este TFG
> es la versión estable más reciente. A diferencia de la v2 (muy extendida en tutoriales), la v3
> introduce: proyectos ESM por defecto, despliegues vía **Ignition** (estado-guardado del
> despliegue en `ignition/deployments/`), plantillas de proyecto con testeo Solidity y Mocha, y
> configuración por perfiles de compilación.

## 3. Estructura del monorepo creada

```
~/Proyectos/auth-blockchain-tfg/
├── contracts/          # Contratos Solidity (*.sol)
│   ├── IdentityRegistry.sol
│   ├── Counter.sol              # (ejemplo de plantilla, se eliminará)
│   └── Counter.t.sol            # (tests en Solidity del ejemplo)
├── test/               # Tests de integración TypeScript (Mocha+Chai)
│   ├── IdentityRegistry.ts
│   └── Counter.ts
├── ignition/modules/   # Módulos de despliegue con Hardhat Ignition
├── scripts/            # Scripts de interacción con la red
├── infra/              # (Módulo 5) Orquestación Docker del sistema completo
├── docs/
│   ├── 01-decisiones.md       # Justificación de stack
│   ├── fases/                  # Documentación fase a fase (material TFG)
│   └── errores/                # Registro de errores y su resolución
├── hardhat.config.ts   # Configuración de compile, redes y plugins
└── AGENTS.md            # Instrucciones del entorno para agentes de IA
```

## 4. El contrato `IdentityRegistry.sol` (explicación para la memoria)

El registro de identidades es la pieza de datos persistente del sistema: **la fuente de verdad
descentralizada** de qué DIDs existen y qué clave pública representan. Los elementos clave:

| Función / elemento | Tipo | Qué hace y por qué |
|---|---|---|
| `struct Identity { bytes pubKey; bool active; }` | Estructura | Modela una identidad: clave pública + flag de actividad (la revocación de clave se completa en Módulo 2) |
| `mapping(bytes32 => Identity) identities` | Almacenamiento | Mapea el **hash keccak256 del DID** → identidad. Se almacena el hash, no el DID literal (determinista y de tamaño fijo) |
| `address immutable owner` | Estado | Sólo el desplegador puede registrar en el stub. En Módulo 2 se sustituirá por control de roles (`AccessControl`) |
| `registerDID(bytes32 didHash, bytes pubKey)` | Escritura | **Única escritura del módulo.** Valida DID no nulo, clave no vacía e inexistencia previa; emite `DIDRegistered` |
| `isRegistered(bytes32)` | Vista (sin gas) | Comprobación rápida de existencia; coste cero, clave para la baja latencia del Proxy Validador |
| `getPubKey(bytes32)` | Vista | Recupera la clave para verificar firmas **fuera** de la cadena |
| `event DIDRegistered(bytes32 indexed, bytes)` | Evento | Permite a servicios (Agente/Módulo 3) **suscribirse y actualizar su caché** sin consultar la cadena a cada instante |

**Consideración de diseño (argumento del TFG):** los contratos de identidad no almacenan
secretos, sólo *huellas* y claves públicas; la verificación criptográfica de las presentaciones
se hace en el Agente (fuera de la cadena), dejando la cadena como registro inmutable y auditable.

## 5. Comandos y proceso de verificación

```bash
# Dependencias + proyecto (ejecutado una sola vez)
npm install --save-dev hardhat @nomicfoundation/hardhat-toolbox
npx hardhat --init --template mocha-ethers

# Compilar
npx hardhat compile                       # -> 3 archivos solc 0.8.34 (evm: osaka)

# Testear (10 pruebas: 3 Solidity + 7 Mocha)
npx hardhat test                          # -> 10 passing

# Red local persistente (nodo dev con 20 cuentas financiadas)
npx hardhat node --port 8545              # RPC http://127.0.0.1:8545

# Despliegue sobre la red local
npx hardhat ignition deploy ignition/modules/IdentityRegistry.ts --network localhost

# Verificación end-to-end (registrar + leer + rechazo de duplicado)
npx hardhat run scripts/register-and-read.ts --network localhost
```

Resultado de la verificación end-to-end:

```
IdentityRegistry en: 0x5FbDB2315678afecb367f032d93F642f64180aa3
DID      : did:key:z6MkhaXgBZDvotDkL5257faiztiGiC2QtKLGpbnnEGta2doK
didHash  : 0x80e705b90c233dc4f67f5583a1f333cb...
Tx de registro: 0x3efa449fa0cd50cc...
--- Estado en cadena ---
isRegistered: true
pubKey     : 0x656432353531392d...
Registro duplicado REVERTIDO (ok): revert
```

## 6. Criterio de aceptación (cumplido ✅)

| Criterio | Resultado |
|---|---|
| `hardhat node` levanta | ✅ Puerto 8545, 20 cuentas con 10000 ETH de prueba |
| Despliegue con dirección y tx hash | ✅ Ignition imprime dirección del contrato |
| Lectura confirma DID registrado | ✅ `isRegistered = true` y pubKey recuperada |
| `npm test` pasa test de humo | ✅ 10/10 tests verdes |
| Conflicto duplicado rechazado | ✅ `revert` contra la regla de negocio |

## 7. Dificultades encontradas

| # | Error | Solución (detalle en `docs/errores/`) |
|---|---|---|
| 1 | npm bloquea `postinstall` de esbuild (política `allowScripts` de npm 11) | `npm install-scripts approve esbuild` |
| 2 | Comparar `bytes` ethers (string hex) contra la entrada (Uint8Array) falla en el test | Convertir con `ethers.hexlify()` al comparar |

## 8. Siguiente paso

**Módulo 2 — Desarrollo del Smart Contract de Identidad y Permisos:** rotación de clave con
firma, revocación con eventos (`RevocationRegistry`), políticas recurso/grupo y control de
roles (`AccessControl`), con cobertura de tests > 80%.