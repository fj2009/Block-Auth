# Manual de uso — Archivador SSI (TFG)

> **Fecha:** 2026-09-24 · **Alcance:** instalación, arranque en dos fases, uso web, API y
> operación del sistema completo (blockchain local + Proxy + PostgreSQL).

## 1. Qué es

Sistema de **identidad autosoberana (SSI)** sobre una blockchain **Ethereum local** (cero gas,
sin internet) que sustituye a un servidor central de identidad:

- La **cadena** es la única fuente de verdad: identidad (`IdentityRegistry`), revocación
  (`RevocationRegistry`) y permiso rol→tabla→operación (`AccessPolicyRegistry`).
- El **Proxy** (`proxy/`, Python/FastAPI) es el validador que comprueba firma, identidad, rol
  y revocación antes de emitir un JWT para acceder a **PostgreSQL**.
- La **web** (`http://127.0.0.1:8001/`) permite **crear identidad**, **entrar** y **administrar
  archivos personales**. Tu clave privada **se cifra en el navegador** con tu contraseña
  (PBKDF2 + AES-256-GCM) y solo se descifra para firmar: la contraseña nunca viaja.

Flujo de una sesión:

```
usuario+contraseña → descifra clave local → firma payload (SSI)
      → Proxy verifica owner + rol + revocación en cadena → JWT
      → acceso a la tabla `archivos` en PostgreSQL (solo tus DIDs de fila)
```

## 2. Requisitos

- **Node.js ≥ 18** (probado con 24.19) y npm.
- **Python 3** (para el venv del Proxy).
- **Docker** (solo si se usa la BD predeterminada; el arranque no lo exige si usas una BD ya
  montada o personalizada).
- Sin internet durante la demo: todas las dependencias se instalan en el paso de instalación.

## 3. Instalación (una vez)

```bash
./instalar.sh
```

Instala, del tirón, las dependencias de las **dos fases**:

| Paso | Fase | Qué hace |
|---|---|---|
| 1/4 | 1 · blockchain | `npm install` (aprueba el postinstall de esbuild) |
| 2/4 | 1 · blockchain | `npx hardhat compile` |
| 3/4 | 1 · blockchain | `npx hardhat test` (tests Mocha+Chai, sin nodo) |
| 4/4 | 2 · aplicación | venv `proxy/.venv` + `requirements.txt` |

## 4. Arranque en dos fases

```bash
./arrancar-sistema.sh start
```

### Fase 1 · Blockchain

Levanta el nodo en `http://127.0.0.1:8545` (chain id 31337), despliega los 3 contratos con
Ignition (`--reset`) y siembra la demo (`doctorA`, `analyst`, `intruso` + políticas de
`archivos`).

> Cada `start` deja una **cadena limpia**: las direcciones de los contratos cambian y las
> identidades registradas antes se pierden. Es el comportamiento previsto para la demo.

### Fase 2 · Aplicación (Proxy + base de datos)

El script pregunta qué base de datos usar:

| Opción | Qué hace |
|---|---|
| **1 · Predeterminada** | Levanta el contenedor `db-tfg` (postgres:16-alpine, `127.0.0.1:5433`, db `tfg`) vía `levantar-db.sh` |
| **2 · Ya montada** | Usa una BD existente sin tocarla (espera esquema y roles de `infra/postgres/init.sql`) |
| **3 · Personalizada** | Te pide host, puerto, nombre de BD y credenciales de `rol_viewer`/`rol_analyst` |

Según la elección se escribe `proxy/.env` y se **(re)arranca el Proxy** en
`http://127.0.0.1:8001`. El reinicio es deliberado: la Fase 1 (`--reset`) cambió las
direcciones de los contratos y un proceso previo quedaría apuntando a la cadena vieja.

**Sin preguntas (automatización):**

```bash
DB_CHOICE=default  ./arrancar-sistema.sh start    # contenedor local
DB_CHOICE=existing ./arrancar-sistema.sh start    # BD existente
DB_CHOICE=custom DB_HOST=1.2.3.4 DB_PORT=5432 DB_NAME=tfg \
  DB_USER_VIEWER=rol_viewer DB_PASS_VIEWER=... \
  DB_USER_ANALYST=rol_analyst DB_PASS_ANALYST=... ./arrancar-sistema.sh start
```

### Usuario demo CARDI

Al terminar, el arranque **aprovisiona el usuario CARDI**:

```text
Usuario : did:web:tfg:cardi
Password: cardofj2009
```

Se genera una wallet nueva, se inscribe `did:web:tfg:cardi` en la cadena con `ROLE_VIEWER` y
su clave privada se guarda **cifrada** (contraseña `cardofj2009`) en
`proxy/web/static/keystore-cardi.json`. La contraseña se puede cambiar por env:
`CARDI_PASSWORD=... ./arrancar-sistema.sh start`.

## 5. Uso de la web

Abre **http://127.0.0.1:8001/**.

### 5.1 Entrar con CARDI

1. En *2 · Entrar*: usuario `cardi`, contraseña `cardofj2009`.
2. El navegador descarga el keystore de CARDI, lo descifra con la contraseña y firma el
   payload; el Proxy verifica la firma frente al owner en cadena y emite un JWT.
3. Verás *Mis archivos* con tu rol (`ROLE_VIEWER`).

### 5.2 Crear una identidad nueva

1. En *1 · Crear identidad*: elige usuario y contraseña (mínimo 8 caracteres).
2. Se genera una wallet en el navegador, se inscribe el DID
   `did:web:tfg:<usuario>` en cadena (firma el nodo con la cuenta admin) y la clave se
   **cifra con tu contraseña** en `localStorage`.
3. Ya puedes *Entrar* con ese usuario/contraseña.

> Si el usuario ya existe en la cadena (como `cardi`), la web te avisa antes de intentar nada.

### 5.3 Archivos

- **Subir:** selecciona un archivo y pulsa *Subir archivo*. Se guarda en la tabla `archivos`
  de PostgreSQL vinculado a tu `didHash`.
- **Listar / Descargar / Borrar:** cada fila es tuya; el Proxy filtra por titular y rechaza
  (403) cualquier archivo de otro usuario aunque tengas JWT.

### 5.4 Restaurar una identidad borrada del navegador

En *Entrar* → *Importar clave privada existente*: pega el usuario, la clave privada `0x...` y
una contraseña (con la que volverá a cifrarse). Si el DID ya está inscrito, la dirección debe
coincidir con el owner en cadena.

## 6. Uso por API / consola

Endpoint del Proxy: `http://127.0.0.1:8001`.

| Endpoint | Uso |
|---|---|
| `GET /health` | Estado del Proxy, cadena y registries |
| `POST /auth/verify` | Login SSI: `{did, payload, firma}` → `{jwt, rol}` |
| `GET /auth/status?did=...` | Estado on-chain de una identidad |
| `POST /auth/register` | Inscribe un DID (firma el nodo con la cuenta admin) |
| `POST /query` | SQL firmado: `{jws, sql}` (bloquea la tabla `archivos`) |
| `POST /files/upload` · `GET /files/list` · `GET /files/download/{id}` · `DELETE /files/{id}` | Archivos personales con `Authorization: Bearer <jwt>` |

### Demo por consola (clientes sembrados)

```bash
npx hardhat run scripts/demo-client.ts --network localhost            # doctorA → SELECT diagnosticos
DEMO_USER=analyst npx hardhat run scripts/demo-client.ts --network localhost
DEMO_USER=doctorA DEMO_SQL="SELECT * FROM archivos;" npx hardhat run scripts/demo-client.ts --network localhost
DEMO_REPLAY=1 npx hardhat run scripts/demo-client.ts --network localhost  # muestra el anti-replay (403)
```

### Ejemplo curl (login SSI + archivos)

```bash
# El payload se firma con tu clave privada (aquí con ethers desde node):
#   node -e 'const {Wallet,keccak256,toUtf8Bytes,getBytes}=require("ethers"); ...'
```

Para el flujo completo es más cómodo escribir un pequeño script con `ethers`
(`provider` + `Wallet(privada)`) replicando `scripts/demo-client.ts`.

## 7. Operación

| Comando | Efecto |
|---|---|
| `./arrancar-sistema.sh status` | Estado de nodo, contratos, BD y Proxy + usuario CARDI |
| `./arrancar-sistema.sh stop` | Para Proxy y nodo (la BD queda en marcha) |
| `./arrancar-sistema.sh restart` | `stop` + `start` (con menú de BD) |
| `./arrancar-sistema.sh reset` | Para todo, borra despliegues, `proxy/.env` y keystores, y vuelve a arrancar |
| `./arrancar-sistema.sh log proxy\|blockchain\|db` | Sigue el log del Proxy, del nodo o de Postgres |

También están disponibles por separado:

```bash
./iniciar.sh start|stop|status|reset|log               # solo la Fase 1 (blockchain)
./infra/levantar-db.sh start|stop|status|log           # solo la BD
./proxy/arrancar.sh start|stop|status|log              # solo el Proxy
```

## 8. Solución de problemas

| Síntoma | Causa / solución |
|---|---|
| `./arrancar-sistema.sh stop` no para el nodo | El nodo se inició externamente y no tiene PID registrado; páralo con `kill $(cat .hardhat-node.pid)` o detén el proceso del nodo |
| El Proxy responde pero `/health` muestra contratos distintos a los del `status` del nodo | El Proxy quedó arrancado antes de un `--reset`; haz `./arrancar-sistema.sh restart` (el arranque siempre reinicia el Proxy a propósito) |
| "No tienes permisos: rol X sobre tabla" | El rol no tiene política; el arranque la asegura vía `_garantizar_politica_archivos` |
| "Contraseña incorrecta" al entrar | Keystore distinto al del arranque actual; recuerda que cada `start` regenera CARDI |
| La `archivos` tabla no existe en BD personalizada | Aplica el esquema `infra/postgres/init.sql` a la BD antes de arrancar |
| El nodo no responde tras el arranque | Mira `tail .hardhat-node.log`; si el puerto 8545 estaba ocupado por otro proceso, libéralo o usa otro puerto |
| `pkill` hablando de "process killed" | **Nunca** uses `pkill -f` con un patrón que esté en la propia línea de comandos; usa los scripts y sus PID |
| Se pierde la sesión web al reiniciar el proxy | Las claves se guardan en `localStorage` (navegador) y el JWT caduca (120 s): vuelve a entrar |

## 9. Seguridad (contexto de demo)

- Red local con cuentas de desarrollo conocidas; **cero gas** y sin tráfico externo.
- El "registro" lo firma el nodo con la cuenta admin desbloqueada (solo desarrollo).
- La contraseña protege la clave privada local; la autenticación **real** sigue siendo la
  firma criptográfica verificada contra el owner en cadena (SSI).
- Claves privadas en `scripts/seed.ts` y `hardhat.config.ts` son las cuentas estándar de la
  frase semilla de `hardhat node` — no usar fuera de una demo local.