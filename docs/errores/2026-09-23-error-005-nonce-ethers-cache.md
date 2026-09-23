# Error #5 — `getTransactionCount` de ethers sigue 1 por detrás (caché del provider)

- **Fecha:** 2026-09-23
- **Fase/Módulo:** Módulo 2 (script de seed contra el nodo local)
- **Relacionado:** sustituye a la solución parcial del Error #4

## Síntoma

Aplicado el fix #4 (`provider.getTransactionCount(addr, "pending")` explícito), el error se
repetía de forma **sistemática** y reproducible, incluso con la cadena totalmente limpia:
tras minar una tx (p. ej. `registerDID(doctorA)`), el siguiente envío seguía usando el nonce
anterior y el nodo respondía `Expected nonce to be N but got N-1`.

## Causa raíz

- `provider.getTransactionCount(addr, "pending")` de ethers v6 no era **estrictamente en vivo**
  en estas condiciones: su valor podía corresponder a un estado anterior a la tx recién minada
  (caché de bloque/pendiente interna del provider).
- Diagnóstico: una llamada **JSON-RPC cruda** (`eth_getTransactionCount` con tag `"pending"`)
  devolvía el valor correcto (`0x5`) mientras que la API del provider devolvía el desfasado.

## Solución definitiva

Usar la llamada RPC directa al nodo para cada nonce, sin intermediación del high-level provider:

```ts
const nonce = Number(
  await provider.send("eth_getTransactionCount", [wallet.address, "pending"]),
);
const tx = await invoke({ nonce });
await tx.wait();
```

Con esto el script `scripts/seed.ts` quedó **100% determinista** e idempotente (puede re-ejecutarse
en una cadena limpia o sobre datos ya sembrados).

## Lección para la memoria

> Cuando se necesite el valor *exacto y vigente* de un estado de red (nonces, etc.) en entornos
> de pruebas con automining, la capa JSON-RPC directamente (`provider.send`) evita las capas de
> caché del provider de ethers. Documentar el motivo evita revertir a la API aparentemente
> equivalente que reintroduciría el bug (YAGNI -> precisión RPC en el caso de uso único).