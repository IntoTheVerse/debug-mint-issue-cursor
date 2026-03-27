
# Devnet: Ephemeral VRF callback missing on consecutive weapon mints

**Video (repro / UX):** [YouTube — mint flow & stuck forging UI](https://youtu.be/3tM_PFEAS8c)

**Wallet (tx history / ordering):** [Solana Explorer — `A6hdCTRfvdxKmLzhs52RSz1rrCxJ42riB9NiL1JUNEoV` (devnet)](https://explorer.solana.com/address/A6hdCTRfvdxKmLzhs52RSz1rrCxJ42riB9NiL1JUNEoV?cluster=devnet)

---

## Context

- **Cluster:** Solana **devnet**
- **Flow:** Marketplace weapon mint via CCMC `MainMintReincarnateNft` → **Ephemeral VRF** requests randomness → a **separate** transaction is expected to invoke `CallbackMintReincarnateNft` (MPL Core create, etc.).
- **Client behavior:** Unity waits for that **callback** transaction (and/or player PDA update). If the callback **never lands**, the UI stays on “forging” until timeout.

---

## Case A — Cyber Spin 6 (success — full path)

| Step | What happened |
|------|----------------|
| 1 | **Main tx** — `MainMintReincarnateNft` → CPI into **VRF** (`Vrf1RNU…`, log `Idx: 8`) → success. This is the **request** path. |
| 2 | **Callback tx** — Top-level is the **Ephemeral VRF** / fulfillment path, which invokes CCMC **`CallbackMintReincarnateNft`**, then **MPL Core** (`CoREEN…`) `Create` / collection approve, and program log **`R:0`**. The **weapon NFT is created in tx 2**, not tx 1. |

**Links**

1. Main: [devnet tx `5UBmhGjf…`](https://explorer.solana.com/tx/5UBmhGjfP8mqsTbgtwH8xngovYDUrHReir7omFUrPsaVAsYXjUKFubucP7rxCffqnuxacDdDryHUsYzDurwgNDRv?cluster=devnet)  
2. Callback: [devnet tx `47oEF3Cr…`](https://explorer.solana.com/tx/47oEF3Cr7k8NDRD5gmnHD8Zf7C9gGyTp63uAbo5p3NwZi65DCfheZ8jx9W47zebVFx2xp7wosKECjozmyQYu6ZsB?cluster=devnet)

---

## Case B — Nova Shatterer (stuck — callback never observed)

| Step | What happened |
|------|----------------|
| 1 | **Main tx** — Same high-level pattern: `MainMintReincarnateNft` → VRF CPI, log **`Idx: 8`** → success. Request path appears to run. |
| 2 | **Callback tx** — **Not observed** on devnet (no second tx with `CallbackMintReincarnateNft` / MPL Core create for this mint). Forging UI waits for something that **did not confirm on-chain**. |

**Link**

- Main only: [devnet tx `4fK4gMLB…`](https://explorer.solana.com/tx/4fK4gMLBpUvfmbhEUAa71PjcLuAtodrK77qJ6hPqF7Gn8mcGBTnwb6iDfgZnXPv4g8Ur6oKTZu4MeNKjweWZg5EG?cluster=devnet)

**Note on instruction ordering**  
On Nova’s main tx, **System Program** appears in the logs **before** the VRF CPI; on Cyber Spin your capture showed VRF immediately after the CCMC instruction label. That often reflects **rent / allocate / transfer** differences between two valid txs, not proof of a bug—but a **byte-level diff** (account metas + inner instructions) between the two main txs is still useful to rule out a **program-side** registration bug.

---

## Summary table

| | Cyber Spin 6 | Nova Shatterer |
|--|--------------|----------------|
| Main (`MainMintReincarnateNft` + VRF request) | Yes | Yes |
| Callback (`CallbackMintReincarnateNft` + MPL Core create) | Yes | **Missing** |
| Where the NFT is created | **Transaction 2** | **Never (no tx 2)** |

**Conclusion for this repro:** The failure mode is **absence of the Ephemeral VRF fulfillment / callback transaction** on devnet, not “Unity mint logic” or “Explorer wrong” in isolation.

---

## Likely cause buckets (for your review)

1. **Fulfillment / operator path (devnet)** — Second tx never submitted, dropped, heavily delayed, or failing before CCMC runs (slow or flaky keepers, limits, unhealthy worker, etc.).
2. **Request identity / queue** — Both mains logged **`Idx: 8`** in our captures; if that index is meaningful globally, **back-to-back** mints might warrant checking for **collision, reuse, or dropped second registration** on the VRF side (we’re not assuming—we’re flagging for your team to interpret against the Ephemeral VRF spec).
3. **On-chain registration bug (less common)** — Main tx succeeds but **does not** correctly register what the fulfiller listens for; would show up as **no callback** even with a healthy operator. **Diff of the two main txs** helps separate (3) from (1).

---

## What would help from MagicBlock

1. **Confirm** whether, for main tx [`4fK4gMLB…`](https://explorer.solana.com/tx/4fK4gMLBpUvfmbhEUAa71PjcLuAtodrK77qJ6hPqF7Gn8mcGBTnwb6iDfgZnXPv4g8Ur6oKTZu4MeNKjweWZg5EG?cluster=devnet), a **callback was ever scheduled** and if anything **failed** on your side (logs, queue, simulation).
2. **Guidance** on **devnet** reliability and any **rate limits / fees** affecting the **fulfillment** transaction (not only the user-signed main tx).
3. If applicable, whether **consecutive** requests from the same wallet/program need **spacing**, **idempotency**, or **client-side serialization** on our end—and whether **`Idx: 8` repeating** across two mains is expected.

---

## What we’re doing on the client (FYI)

- Polling anchored on `newAssetPDA` + logs for successful `CallbackMintReincarnateNft`.
- **`mintNew`:** optional **player-PDA** check when chain state updates before sig history.
- **Serial guard:** block starting a **second** mint while the first is still **signing or waiting on VRF** (reduces back-to-back load; does not replace a missing callback tx).

---

## Practical next checks (either side)

1. Explorer: **`newAssetPDA`** for Nova — confirm only one signature vs two (main + callback).
2. Retry Nova after several minutes: if tx 2 **sometimes** appears → points to **devnet / fulfillment reliability**; if **never** → deeper look at **request registration + operator**.
3. Side-by-side **decode** of Cyber vs Nova **main** txs (accounts + inner ix).

---
