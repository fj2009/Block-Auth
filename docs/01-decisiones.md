# Decisiones de diseño del proyecto (material para la memoria)

Documento vivo que recoge las decisiones arquitectónicas y técnicas tomadas en cada fase, con
su justificación. Cada decisión se identifica con prefijo `DEC-`.

## DEC-01 · Modelo de identidad: SSI (W3C)

- **Decisión:** Autosoberanía de identidad (SSI) con la trinidad **DID + Credencial Verificable
  (VC) + Presentación Verificable (VP)**. El usuario es *holder* de su propia identidad.
- **Justificación:** elimina el repositorio central de credenciales (spoofing del servidor de
  identidad); el servidor no guarda secretos de usuarios.

## DEC-02 · Blockchain: Ethereum local (Hardhat) en lugar de mainnet/testnet pública

- **Decisión:** Red Ethereum **local de coste cero** con Hardhat (nodo dev) para el MVP.
- **Justificación:** (a) *cero gas* — el tribunal no debe ver costes; (b) *sin dependencia de
  internet / RPC externos* — coherente con el objetivo de eliminar el punto único de fallo;
  (c) tooling sólido y extensible.
- **Alternativas descartadas:** Redes públicas Ethereum/Polygon (SPOF de RPC + coste),
  Hyperledger Fabric (idóneo en producción corporativa pero toolchain más complejo para el
  alcance de un MVP de TFG).
- **Nota:** en el Módulo 5 se puede sustituir el nodo Hardhat por **Besu IBFT** (red privada
  permisionada real) como mejora de realismo con el mismo código Solidity.

## DEC-03 · Lenguaje de contratos: Solidity 0.8.34; lenguaje del Agente: Python (Módulo 3)

- **Decisión:** Solidity para contratos; Python + FastAPI para el Agente de Autenticación.
- **Justificación:** Solidity es de facto el estándar de smart contracts (empleabilidad y
  documentación). Python acelera la implementación del Agente (verificación criptográfica,
  integración con FreeRADIUS) manteniendo legibilidad para la defensa.

## DEC-04 · Compilador/toolchain: Hardhat 3

- **Decisión:** Hardhat **3.17** (v3 estable en 2026) en lugar de depositarse en la v2.
- **Justificación:** compilación por perfiles, **Ignition** (despliegues declarativos con estado
  persistente), TypeScript 6 de serie, plantillas `npx hardhat --init --template`.
- **Riesgo:** documentación más escasa que la v2 → se ha repetido la lectura de fuentes
  oficiales (hardhat.org docs / llms.txt).

## DEC-05 · Registro de identidades: hash del DID, claves públicas, no secretos on-chain

- **Decisión:** el contrato almacena `keccak256(DID) → (pubKey, active)`. La verificación
  criptográfica (firmas, VC) ocurre **fuera de la cadena** en el Agente.
- **Justificación:** la cadena es un registro inmutable y auditable, no un motor de cómputo;
  minimizar lectura/escritura on-chain reduce gas y latencia (argumento clave de la defensa).

## DEC-06 · Revocación y latencia (diseño a implementar en Módulos 2 y 3)

- **Decisión:** revocación on-chain (`RevocationRegistry` con eventos, Módulo 2) + **caché TTL
  con invalidación por eventos** en el Agente (Módulo 3), de modo que la cadena **no está en el
  camino crítico** de cada intento de autenticación.
- **Justificación:** responde directamente a la pregunta esperada del tribunal
  ("¿en qué se diferencia de RADIUS a 10 ms?"): con caché caliente la validación es
  < 150 ms; la cadena sólo interviene en *miss* de caché y en eventos de revocación.

## DEC-07 · Gestión de errores y trazabilidad

- **Decisión:** cada error encontrado se documenta en `docs/errores/` con su causa raíz y
  solución, y cada fase en `docs/fases/`.
- **Justificación:** material directo para el apartado "Dificultades encontradas" de la memoria,
  demostrando gestión real del proyecto.

## DEC-08 · DID vinculado a dirección Ethereum (Módulo 2)

- **Decisión:** cada DID se asocia a una **dirección Ethereum** (clave pública secp256k1 de la
  wallet del usuario), en lugar de almacenar claves arbitrarias en `bytes`.
- **Justificación:** la verificación de firmas en cadena (rotación de clave) usa `ecrecover`
  (precompilado nativo y barato); Ed25519/otras curvas exigirían verificación en el EVM sin
  precompilado. Además el modelo coincide con wallets estándar (ethers/MetaMask). Se verifica
  con `MessageHashUtils.toEthSignedMessageHash` + `ECDSA.recover` (EIP-191/eth_sign) sobre
  `keccak256(didHash, newOwner)`, firmado por la clave ANTERIOR (prueba de pertenencia).

## DEC-09 · API de roles propia sin colisionar con OpenZeppelin

- **Decisión:** el RBAC de organización usa `assignRole`/`unassignRole` en lugar de
  `grantRole`/`revokeRole`.
- **Justificación:** `AccessPolicyRegistry` hereda de `AccessControl`, que ya expone
  `grantRole(bytes32, address)`; una función propia `grantRole(bytes32, bytes32)` crea una **ABI
  ambigua** que ethers rechaza con `ambiguous function description`. El renombrado elimina la
  colisión sin sacrificar claridad semántica.
- **Nota:** equilibrio entre REVOCACIÓN irreversible de identidad (RevocationRegistry, por
  seguridad, tipo kill-switch) y revocación REVERSIBLE de rol (AccessPolicyRegistry, para
  operaciones y demo del "momento mágico").