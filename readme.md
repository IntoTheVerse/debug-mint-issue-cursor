## VRF mint callback issue for consecutive mints 

you can check the video here: [https://youtu.be/3tM_PFEAS8c](https://youtu.be/3tM_PFEAS8c)

check tx history for this account here to see the order of transactions and their timestamps: [https://explorer.solana.com/address/A6hdCTRfvdxKmLzhs52RSz1rrCxJ42riB9NiL1JUNEoV?cluster=devnet](https://explorer.solana.com/address/A6hdCTRfvdxKmLzhs52RSz1rrCxJ42riB9NiL1JUNEoV?cluster=devnet)

### Cyber Spin (full path)

1. **Main tx** — `MainMintReincarnateNft` → CPI into **VRF** (`Vrf1RNU…`, `Idx: 8`) → `Program returned success`. That step **requests** randomness / schedules the callback path your program expects. 
[https://explorer.solana.com/tx/5UBmhGjfP8mqsTbgtwH8xngovYDUrHReir7omFUrPsaVAsYXjUKFubucP7rxCffqnuxacDdDryHUsYzDurwgNDRv?cluster=devnet](https://explorer.solana.com/tx/5UBmhGjfP8mqsTbgtwH8xngovYDUrHReir7omFUrPsaVAsYXjUKFubucP7rxCffqnuxacDdDryHUsYzDurwgNDRv?cluster=devnet)

2. **Second tx** — **Not your game program as the outer instruction.** It’s the **Ephemeral VRF** path:

[https://explorer.solana.com/tx/47oEF3Cr7k8NDRD5gmnHD8Zf7C9gGyTp63uAbo5p3NwZi65DCfheZ8jx9W47zebVFx2xp7wosKECjozmyQYu6ZsB?cluster=devnet](https://explorer.solana.com/tx/47oEF3Cr7k8NDRD5gmnHD8Zf7C9gGyTp63uAbo5p3NwZi65DCfheZ8jx9W47zebVFx2xp7wosKECjozmyQYu6ZsB?cluster=devnet)
   - Outer: **EphemeralVrf**-style flow (`6EqDKL…`)
   - Inner: **`CallbackMintReincarnateNft`**
   - Then **MPL Core** (`CoREEN…`) `Create` / `Approve`, etc.
   - Log **`R:0`** (your rarity line)

So the **weapon NFT is actually created in tx 2**, driven by the **VRF callback** program, not in tx 1.

### Nova Shatterer (stuck)



1. **Main tx** — Same top-level: `MainMintReincarnateNft` → VRF `Idx: 8` → success. So **tx 1 did the “request” path** similarly to Cyber Spin.
[https://explorer.solana.com/tx/4fK4gMLBpUvfmbhEUAa71PjcLuAtodrK77qJ6hPqF7Gn8mcGBTnwb6iDfgZnXPv4g8Ur6oKTZu4MeNKjweWZg5EG?cluster=devnet](https://explorer.solana.com/tx/4fK4gMLBpUvfmbhEUAa71PjcLuAtodrK77qJ6hPqF7Gn8mcGBTnwb6iDfgZnXPv4g8Ur6oKTZu4MeNKjweWZg5EG?cluster=devnet)

2. **One clear difference in your logs** — On Nova, **System Program runs before VRF**; on Cyber Spin you only pasted VRF right after the instruction label. That usually just means **an extra rent/allocate/transfer** in that build of the ix (not automatically wrong), but it’s worth diffing **accounts + inner ix order** between the two txs in Explorer if something is off.

3. **Second tx** — **Never landed.** So nothing ever ran `CallbackMintReincarnateNft` → **no MPL Core create** for that mint. The forging screen is waiting for something that **never happened on-chain**.

So: **Nova is not “mint failed in Unity” — it’s “VRF callback tx never appeared.”**

---

## Root cause bucket (almost certainly)

The **Ephemeral VRF / fulfillment pipeline** (the stack that produces the **second** transaction) **did not complete** for Nova:

- Devnet **keepers / fulfillment** can be **slow, flaky, or rate-limited**.
- Sometimes the **second** request is **delayed** past your **180s** poll, or **dropped** if the service is unhealthy.
- Less common but important: **bad or reused VRF / request state** so fulfillment never schedules (would need program + account diff).

Your Unity poller is doing the right *kind* of thing: wait for `CallbackMintReincarnateNft` on the poll account. If that tx **doesn’t exist**, the UI will hang **forever** unless you timeout and show a clear error.

---

## What to do in practice

1. **Confirm on Explorer** for Nova’s **main** tx: note **time**, **slot**, **signers**. Then open the **VRF / request PDA** accounts your program documents and see if a **pending** request is still there or if devnet shows **no** follow-up signature.

2. **Retry** the same mint after several minutes; if **sometimes** tx 2 appears, it’s **devnet / fulfillment reliability**.

3. **Compare byte-for-byte** the two **MainMint** transactions (accounts list + logs). If Nova’s path accidentally **skips** registering the request the Ephemeral worker listens for, you’d still see “success” on tx 1 but **no** callback — that’s an **on-chain program** bug to fix.

4. **Ephemeral VRF / operator** — check their **devnet** status, limits, and whether they require a **minimum fee** or **priority fee** for the *callback* path (sometimes only the user tx is visible in your tests).

---

## Short summary

| Item | Cyber Spin | Nova Shatterer |
|------|------------|----------------|
| Main tx (`MainMintReincarnateNft` + VRF request) | Yes | Yes |
| Callback tx (`CallbackMintReincarnateNft` + MPL Core) | Yes | **Missing** |
| Where the NFT is actually created | **Tx 2** | **Never** |

So the issue is **fulfillment of the VRF / Ephemeral callback**, not MPL Core in isolation and not “Explorer logging wrong” — **tx 2 was never submitted or never confirmed on devnet.**
