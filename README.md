# Sistema de Autenticación Descentralizada sobre Blockchain — TFG

Trabajo de Fin de Grado (Telecomunicaciones): sustitución de los servidores de identidad
centrales (RADIUS/AD) por un sistema **SSI (Self-Sovereign Identity)** sobre **blockchain
Ethereum local**, eliminando el punto único de fallo y endureciendo el arranque frente a DoS.

## Estado del proyecto (roadmap 5 módulos)

| Módulo | Contenido | Estado |
|---|---|---|
| **1** | Entorno de desarrollo + Blockchain local + `IdentityRegistry` (stub) | ✅ Completado |
| **2** | Smart contracts de identidad y permisos (rotación, revocación, roles) | ✅ Completado |
| **3** | Agente de autenticación (Proxy Validador + caché anti-latencia) | ⏳ Pendiente |
| **4** | Base de datos (PostgreSQL en Docker) + PEP + demo clientes | ⏳ Pendiente |
| **5** | Integración final (Docker) + pruebas de estrés y caída de nodo | ⏳ Pendiente |

Documentación: decisiones en `docs/01-decisiones.md`, fases en `docs/fases/`, errores
registrados en `docs/errores/`.

## Requisitos

- Node.js ≥ 18 (probado con 24.19) y npm.
- Sin internet: **cero gas**, red local Hardhat.

## Comandos rápidos

```bash
# Instalar dependencias, compilar y ejecutar tests (primera vez)
./instalar.sh

# Lanzar el sistema: nodo + despliegue + seed (si el nodo no estaba, lo arranca)
./iniciar.sh start

# Estado, parada, reinicio desde cero
./iniciar.sh status
./iniciar.sh stop        # para (PID registrado) solo el nodo arrancado por este script
./iniciar.sh reset       # para, borra despliegues y relanza todo
./iniciar.sh log         # seguir el log del nodo

# Gestión "manual" equivalente:
npx hardhat node --port 8545                                          # nodo local persistente
npx hardhat ignition deploy ignition/modules/AccessStack.ts --network localhost
npx hardhat run scripts/seed.ts --network localhost                   # idempotente: doctorA/analyst/intruso
```

## Estructura

```
contracts/          # Solidity: IdentityRegistry, RevocationRegistry, AccessPolicyRegistry
test/               # Tests Mocha + Chai (TS) — 100% cobertura
ignition/modules/   # Despliegues declarativos (Hardhat Ignition)
scripts/            # Scripts: seed + utilidades de despliegue
infra/              # Orquestación Docker (Módulo 5)
docs/               # decisions + fases + errores (material para la memoria)
```

## Trabajando sobre este proyecto

Ver `AGENTS.md` en la raíz (generado junto al proyecto Hardhat): activa la skill `hardhat`
para tareas de test/config, y consulta la documentación oficial de Hardhat 3
(https://hardhat.org/llms.txt) y ethers v6 (https://docs.ethers.org/v6/).