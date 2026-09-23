# Error #2 — ethers devuelve `bytes` como hex mientras el test usa `Uint8Array`

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 1 (tests del `IdentityRegistry`)
- **Contexto:** test "debería registrar un DID y emitir el evento DIDRegistered"

## Síntoma

```
AssertionError: expected '0x7075626b65792d616c6963652d303030303…' to equal Uint8Array[ 112, 117, 98, 107, …(-62) ]
  at Context.<anonymous> (test/IdentityRegistry.ts:24:52)
```

El test fallaba al comparar la clave pública devuelta por el contrato (`getPubKey`) y la
emistada en el evento `DIDRegistered` contra el valor que el test enviaba como entrada.

## Causa raíz

En ethers v6 los `bytes` de Solidity se representan de forma distinta según la dirección del viaje:

| Lugar | Representación |
|---|---|
| **Entrada** a una función del contrato | `Uint8Array` o `BytesLike` (hex) — el test pasaba `ethers.toUtf8Bytes(...)` |
| **Salida / evento** devuelto por el contrato | **`string` en formato hex** (`0x656432...`) |

El test comparaba un `Uint8Array` (entrada) con la salida hex del contrato → nunca coincidían.

## Solución aplicada

Normalizar la salida a hex antes de comparar:

```ts
const pubKeyHex = ethers.hexlify(PUBKEY_ALICE);

expect((await registry.getPubKey(DID_ALICE))).to.equal(pubKeyHex);
await expect(registry.registerDID(DID_ALICE, PUBKEY_ALICE))
  .to.emit(registry, "DIDRegistered")
  .withArgs(DID_ALICE, pubKeyHex);
```

## Lección para la memoria

> Al interactuar con contratos desde ethers.js hay que ser consciente de los **tipos de
> retorno**: los `bytes` y `string` se devuelven como cadenas hex, no como `Uint8Array`. Se
> recomienda fijar una convención en el proyecto (comparar siempre contra `ethers.hexlify()`
> de la entrada) y reutilizarla en todos los tests para evitar fallos intermitentes.