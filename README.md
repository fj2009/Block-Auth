
# 🛡️ Block-Auth: Decentralized Access Control System

Trabajo de Fin de Grado (Telecomunicaciones):

  **Block-Auth** es un sistema de autenticación avanzado que
  sustituye los servidores de identidad centralizados (como
  RADIUS o Active Directory) por un registro inmutable basado en
  **Blockchain**.

  El objetivo principal es eliminar el **Punto Único de Fallo
  (SPOF)** y devolver la soberanía de la identidad al usuario
  mediante el uso de **SSI (Self-Sovereign Identity)** y 
  endureciendo el arranque frente a DoS.
  .

<img width="956" height="661" alt="Captura desde 2026-09-24 10-10-13" src="https://github.com/user-attachments/assets/7af20e5e-6579-4306-b502-a594c40fda1a" />
<img width="956" height="661" alt="Captura desde 2026-09-24 10-46-05" src="https://github.com/user-attachments/assets/72816ede-607c-4d84-b724-d98fe797feaf" />

## 🚀 ¿Cómo funciona?

  En lugar de utilizar contraseñas almacenadas en una base de
  datos, Block-Auth utiliza **criptografía de clave pública**.
  El flujo de acceso es el siguiente:

  1. **Firma Digital:** El usuario solicita acceso a un recurso
  firmando la petición con su clave privada.
  2. **Proxy de Validación:** Un middleware (desarrollado en
  FastAPI) intercepta la petición y verifica que la firma sea
  válida.
  3. **Consulta a la Blockchain:** El Proxy consulta un **Smart 
  Contract** en la Blockchain para verificar si la dirección
  pública del usuario posee el rol necesario (RBAC - Role-Based
  Access Control).
  4. **Acceso al Recurso:** Si la validación es exitosa, el
  Proxy concede el acceso a la Base de Datos. Si no, la petición
  es rechazada instantáneamente.

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

# bash
 Instalar dependencias, compilar y ejecutar tests (primera vez)
```
./instalar.sh
```
 Lanzar el sistema: nodo + despliegue + seed (si el nodo no estaba, lo arranca)
```
./iniciar.sh start
```
 Estado, parada, reinicio desde cero
```
./iniciar.sh status
```
 para (PID registrado) solo el nodo arrancado por este script
```
./iniciar.sh stop      
```
  para, borra despliegues y relanza todo
```
./iniciar.sh reset      
```
  seguir el log del nodo
```
./iniciar.sh log        
```
## 🛠️ Stack Tecnológico

  ### Backend & Infraestructura
  - **Blockchain:** Ethereum / EVM (implementado localmente con
  Ganache/Hardhat).
  - **Smart Contracts:** Solidity.
  - **Middleware / API:** Python con FastAPI y Web3.py.
  - **Base de Datos:** SQL / NoSQL (Protegida mediante el
  Proxy).

  ### Seguridad y Criptografía
  - **Identidad:** DIDs (Decentralized Identifiers).
  - **Algoritmo de Firma:** ECDSA (Elliptic Curve Digital
  Signature Algorithm).
  - **Modelo de Acceso:** RBAC (Role-Based Access Control)
  descentralizado.

## 🌟 Características Principales

  - ✅ **Sin Punto Único de Fallo:** Al estar la identidad en la
  Blockchain, el sistema no depende de un único servidor
  central.
  - ✅ **Inmutabilidad:** Los permisos son auditables y no
  pueden ser alterados sin dejar rastro.
  - ✅ **Privacidad y Control:** El usuario es el único dueño de
  sus claves privadas.
  - ✅ **Control Granular:** Permite definir roles específicos
  (Admin, Viewer, Editor) directamente en la cadena de bloques.

## 🎓 Proyecto de TFG

  Este proyecto ha sido desarrollado como parte del Trabajo de
  Fin de Grado en Telecomunicaciones, enfocándose en la
  aplicación de tecnologías descentralizadas para la mejora de
  la ciberseguridad en infraestructuras de datos.

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
