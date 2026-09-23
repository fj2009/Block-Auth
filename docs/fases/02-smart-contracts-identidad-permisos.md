# Fase 2 — Smart Contracts de Identidad y Permisos

> **Fecha:** 2026-09-23 · **Módulo:** 2 de 5 · **Estado:** ✅ completado

## 1. Objetivo

Pasar del stub del Módulo 1 a la **fuente de verdad de identidad y permisos** completa:
rotación de clave (soberanía), revocación (crítica en sistemas descentralizados) y autorización
**granular rol → tabla → operación** que sustituirá a la tabla `users`/`roles` de la base de
datos. Todo ello con control de roles de administración (OpenZeppelin AccessControl),
eventos para invalidación reactiva de caché (clave del Módulo 3) y cobertura de tests > 80%.

## 2. Diseño: separación de responsabilidades en 3 contratos

| Contrato | Responsabilidad | "Pregunta" que responde |
|---|---|---|
| `IdentityRegistry` | Registro y rotación de claves (DID → dirección wallet) | *¿Quién eres? ¿Sigue registrado? ¿Tienes la clave que dices tener?* |
| `RevocationRegistry` | Revocación de identidad **irreversible** + evento `Revoked` | *¿Estás expulsado?* |
| `AccessPolicyRegistry` | RBAC descentralizado: roles y políticas rol→tabla→operación; **reversible** (`assignRole`/`unassignRole`) | *¿Puedes leer/escribir/borrar en esta tabla?* |

Esta separación es un argumento de diseño del TFG: *quién eres* (identidad) ≠ *qué puedes
hacer* (autorización). Además, las revocaciones de identidad y de rol tienen **semánticas
distintas**: la primera es permanente (kill-switch, clave comprometida), la segunda es
reversible (cambio operativo de permisos — el "momento mágico" de la demodefensa).

## 3. Decisiones clave (DEC-08 y DEC-09)

**DEC-08 · DID vinculado a dirección Ethereum en lugar de clave en bytes.**
Verificar una firma *en cadena* con curvas tipo Ed25519 exige precompilados inexistentes en el
EVM. Asociando cada DID a una **dirección Ethereum** (clave pública secp256k1 de la wallet) la
rotación se verifica con `ECDSA.recover` (`ecrecover`, precompilado nativo, coste mínimo) y el
modelo coincide con interacción de wallet real (ethers/MetaMask).

**DEC-09 · `assignRole`/`unassignRole`, no `grantRole`.**
El contrato hereda de `AccessControl` de OpenZeppelin, que ya define `grantRole(bytes32, address)`.
Una función propia `grantRole(bytes32, bytes32)` **colisiona en el ABI** (ethers no puede
desambiguar, error `ambiguous function description`). Se renombra la API de organización para
evitar el choque.

**Rotación (soberanía del usuario):** `rotateKey(didHash, newOwner, sig)` verifica EIP-191
(`eth_sign`) sobre `keccak256(didHash, newOwner)`; quien posee la clave ANTIGUA demuestra
pertenencia firmando el traspaso. No requiere admin.

## 4. Funciones y eventos (resumen)

| Contrato | Función | Notas |
|---|---|---|
| IdentityRegistry | `registerDID(didHash, owner)` | solo ADMIN_ROLE; revert si existe |
| | `rotateKey(didHash, newOwner, sig)` | autogestión; `MessageHashUtils.toEthSignedMessageHash` + `ECDSA.recover` |
| | `deactivate(didHash)` | kill-switch, solo admin, irreversible |
| | `isRegistered / isDeactivated / getOwner` | vistas sin gas (proxy) |
| | eventos | `DIDRegistered`, `KeyRotated`, `IdentityDeactivated` |
| RevocationRegistry | `revoke(didHash)` | REVOKER_ROLE (concedible a más operadores) |
| | `isRevoked` | vista sin gas |
| | evento | `Revoked` (para invalidación de caché en Módulo 3) |
| AccessPolicyRegistry | `setPolicy(rol, tabla, ops)` | máscara: READ=1, WRITE=2, DELETE=4; solo admin, valida rol |
| | `assignRole(didHash, rol)` / `unassignRole(didHash)` | reversible; eventos `RoleGrantedFor`/`RoleRevokedFor` |
| | `can(didHash, tabla, op)` | **vista sin gas** — oráculo del Proxy |

## 5. Verificación

```
npx hardhat test              # 16 pruebas (Mocha) — todas en verde
npx hardhat test --coverage   # 100% lineas / 100% statements en los 3 contratos (>80% exigido)
```

**Despliegue + seed (cadena local):**

```bash
npx hardhat node --port 8545                                  # nodo persistente
npx hardhat ignition deploy ignition/modules/AccessStack.ts --network localhost
npx hardhat run scripts/seed.ts --network localhost           # idempotente
```

Resultado del seed (tabla de permisos real consultada al contrato):

```
doctorA  diagnosticos READ    -> PERMITIDO     doctorA  ventas READ -> DENEGADO
doctorA  diagnosticos WRITE   -> DENEGADO      analyst  ventas READ -> PERMITIDO
doctorA  diagnosticos DELETE  -> DENEGADO      analyst  diagnosticos READ -> DENEGADO
intruso  diagnosticos READ    -> DENEGADO      (identidad no registrada)

Momento mágico:
  Doctor A lee diagnosticos              -> true
  TX unassignRole(doctorA)               -> (1 transacción)
  Doctor A lee diagnosticos AHORA        -> false   ← acceso retirado en caliente
  TX assignRole(doctorA)                 -> (1 transacción)
  Doctor A lee diagnosticos DE NUEVO     -> true
```

## 6. Criterio de aceptación (✅)

| Criterio | Resultado |
|---|---|
| Rotación de clave con firma válida; revierte con firma inválida | ✅ tests |
| Revocación con evento; irreversible; solo REVOKER_ROLE | ✅ tests |
| Permisos granulares rol→tabla→operación evaluables (vistas sin gas) | ✅ tests + seed |
| Control de roles no-admin revierte | ✅ tests |
| Cobertura > 80% | ✅ 100% los 3 contratos |
| Seed idempotente ejecutable ante el tribunal | ✅ corrección tras rediseño de nonce |

## 7. Dificultades encontradas (Módulo 2)

| # | Error | Resolución |
|---|---|---|
| #4 | `nonce has already been used` al enviar txs secuenciales desde ethers | Pasar nonce RPC explícito antes de cada envío |
| #5 | Incluso con `provider.getTransactionCount` el nonce iba 1 por detrás | Llamada JSON-RPC **cruda** `eth_getTransactionCount(pending)` |
| (ref) | `toEthSignedMessageHash` no encontrado (OZ 5.6) | La API se movió a `MessageHashUtils`; importar ese módulo |
| (ref) | Colisión `grantRole(bytes32, bytes32)` con OZ | Renombrar a `assignRole`/`unassignRole` (DEC-09) |
| (ref) | Matcher `.reverted` deprecado en HH3 | Usar `.revert()` |
| (ref) | ethers `signMessage(hex)` firmaba la cadena, no los bytes | `ethers.getBytes(...)` antes de firmar (actualización del #2) |

Detalles con causa raíz en `docs/errores/`.

## 8. Siguiente paso

**Módulo 3 — Agente de Autenticación (Proxy Validador FastAPI):** verificación de firmas de
petición (nonce + timestamp), consulta `isRegistered`/`isRevoked`/`can` con **caché TTL + JWT
de sesión** (la cadena fuera del camino crítico), y **listener de eventos** `Revoked`/`Revoked`
para invalidar la caché en < 2 s. Se medirá p95 < 150 ms con caché caliente.