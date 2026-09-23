# Error #8 — web3.py: `keccak(...).hex()` no lleva prefijo `0x` y `bytes32` lo exige

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 4 (registro de identidad desde el Proxy: `_garantizar_politica_archivos`)

## Síntoma

`POST /auth/register` respondía 500 al consultar `policies(bytes32,string)`:

```
ABI Not Found! ... Argument 1 value `8873feaf…70417` is not compatible with type `bytes32`
```

La misma llamada desde `seed.ts` (ethers) funcionaba: el hash del rol era correcto.

## Causa raíz

`Web3.keccak(text="ROLE_VIEWER").hex()` devuelve el hex SIN el prefijo `0x`
(`HexBytes.hex()` es `bytes.hex()`, no como el `str()` que sí lo incluye). El encoder ABI de
web3.py rechaza un `str` hex sin `0x` para argumentos `bytes32`. En consecuencia el hash de rol
usado en llamadas/programaciones de contratos (`ROLE_BYTES`) era un valor que web3 no podía
codificar como `bytes32` → `MismatchedABI` en toda llamada que pasara ese hash.

## Solución aplicada

Normalizar los hashes heredados con el prefijo:

```python
ROLE_BYTES = {r: w3.to_hex(w3.keccak(text=r)) for r in ROLES}   # incluye "0x"
# y al leer el resultado de un getter bytes32:
ROLE_BY_BYTES.get(w3.to_hex(rb))
```

Precaución: `roleOf(dh)` compara el getter contra `ROLE_BY_BYTES`; al cambiar el formato de la
clave hay que cambiar también la lectura, o el rol dejará de reconocerse.

## Lección para la memoria

> En web3.py los `HexBytes` tienen dos representaciones: `str()` añade `0x` y `.hex()` no. Para
> argumentos `bytes32` de contratos y para claves de mapas hash hay que usar SIEMPRE una forma
> canónica (con `0x`), y al leer getters normalizar igual: `w3.to_hex(valor)`. OJO añadido: la
> función `can(dh, tabla, op)` del Proxy espera `op` como NOMBRE («read»/«write»/«delete») y
> mapea la máscara internamente; pasar el entero produce `KeyError: <n>` (el número del op
> como mensaje), síntoma que parece un error de BBDD cuando en realidad es de convención de tipos.