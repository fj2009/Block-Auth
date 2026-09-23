# Error #4 — `nonce has already been used` al enviar txs secuenciales (ethers)

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 2 (script de seed contra el nodo local)

## Síntoma

Al enviar varias transacciones seguidas del mismo emisor (registro de identidades, políticas y
roles), ethers lanzaba:

```
Nonce too low. Expected nonce to be X but got X-1. Note that transactions
can't be queued when automining. (NONCE_EXPIRED)
```

El nodo local de Hardhat está en modo **automining**: cada tx se mina al instante y el nonce
debe ser **exactamente el siguiente** (no admite colas). Ethers gestiona el nonce de la wallet
internamente, y ante el tráfico previo del mismo emisor (p. ej. los despliegues de Ignition)
su contador quedaba desfasado.

## Causa raíz

- El emisor de las txs del seed (cuenta `admin`) ya había firmado otras txs en el mismo nodo
  (despliegues de Ignition), pero la wallet de ethers **no veía ese histórico completo** y
  partía de un nonce antiguo.
- Automining rechaza *estrictamente* cualquier nonce distinto al esperado → la tx no encolaba
  y fallaba con `NONCE_EXPIRED`.

## Solución aplicada (v1)

Pasar el nonce explícito leído del nodo justo antes de cada envío:

```ts
const nonce = await provider.getTransactionCount(wallet.address, "pending");
const tx = await contract.call(..., { nonce });
```

## Lección para la memoria

> En redes con *automining* el nonce debe ser el siguiente exacto. Depender de la caché de
> nonce de ethers es frágil cuando varios clientes (Ignition + scripts) envían del mismo emisor;
> la práctica robusta es **obtener el nonce vigente en el momento de cada envío**.