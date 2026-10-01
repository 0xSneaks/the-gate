# Phase 0 — Forensics report

Date: 2026-10-01. Sources: `@meteora-ag/dynamic-bonding-curve-sdk@1.5.13` (embedded IDL v0.2.1,
address `dbcij3LWUppWqq96dh6gJWwBifmcGfLSB5D4DuSMaqN`), and `MeteoraAg/dynamic-bonding-curve`
source at commit `f552f20aa3c1c7631427c3827aeea7c58b902813` (2026-09-09).

## STATUS: PARTIAL / BLOCKED on on-chain data

The session's egress policy denies every Solana RPC host tried (`api.mainnet-beta.solana.com`,
`api.devnet.solana.com`, Helius, Alchemy, Ankr, PublicNode), plus Solscan. Explorer scraping
also failed. Because of that, items 1, 3 and 4 (reading the reference mint, pool and transactions)
**have not been done**. Nothing below about the reference token is VERIFIED.
Devnet is blocked as well, so Phase 4 can't run from this container until the policy changes.

## What *is* verified (DBC program source and IDL)

| # | Finding | Where |
|---|---|---|
| V1 | Hook pools use their own instruction set: `create_config_with_transfer_hook`, `initialize_virtual_pool_with_token2022_transfer_hook`, `swap2_with_transfer_hook`. Plain `swap`/`swap2` reject a `TransferHookPool` (`PoolTypeMismatch`). | `ix_swap.rs:70`, `ix_swap2_with_transfer_hook.rs` |
| V2 | At pool init, DBC sets mint `transfer_hook::authority = pool_authority` (`FhVo3mqL8PW5pH5U2CN4XE33DokiyZnUwuGpH2hmHLuM`) and `program_id = config.transfer_hook_program`. The partner/creator **cannot** repoint the hook program on the mint. | `ix_initialize_virtual_pool_with_token2022_transfer_hook.rs:46-47` |
| V3 | On the swap that completes the curve, DBC CPIs `transfer_hook::update(None)` and then `set_authority(TransferHookProgramId, None)`. The hook is permanently removed **before** migration. The source comment says DAMM v2 doesn't support active hooks. | `process_swap.rs:365-428` |
| V4 | `migration_damm_v2` accepts `TransferHookPool`. DAMM v1 does not (no Token-2022). | IDL docs |
| V5 | `swap2_with_transfer_hook` has exactly one signer, `payer` (the trader). Buy = vault→user transfer with authority `pool_authority` (PDA). Sell = user→`base_vault` transfer with authority = user. | `ix_swap2_with_transfer_hook.rs`, `utils/token.rs` |
| V6 | Extra accounts for the hook come in as remaining accounts, tagged `TransferHookBase`/`TransferHookBaseReferral` slices. DBC resolves them with `add_extra_accounts_for_execute_cpi`. If any are missing, the swap fails with error 6071. | `utils/remaining_accounts.rs`, `utils/token.rs:87-100` |
| V7 | If a fee rate-limiter is active, DBC enforces one swap per tx on that pool, and rejects CPI deeper than stack height 2. So router→DBC is OK, but router→router→DBC fails. | `process_swap.rs:201-211, 445-490` |
| V8 | The SDK has `createConfigWithTransferHook`, `createPoolWithTransferHook`, first-buy variants and `TransferHookAccountsInfo`. The SDK refuses DBC, Token and Token-2022 program IDs as the hook program. | SDK `validateTransferHookProgram` |

## Answers to item 5 (generic DBC, not yet confirmed for the reference token)

- **Can the hook authority change behaviour mid-curve?** Through the mint, no (V2): only DBC's PDA holds hook authority, and it only ever revokes. **But** the hook *program's* upgrade authority and any admin-controlled whitelist inside it *can* change behaviour mid-curve. That's the real trust surface, and it has to be checked on-chain for the reference hook (upgradeable? who's the upgrade authority?). (ASSUMED until RPC access.)
- **Does the gate apply to sells and wallet transfers?** Token-2022 calls the hook on **every** `transfer_checked`, so it hits buys, sells and W2W transfers alike. Which of those it *blocks* depends on the hook's own logic, which needs the on-chain read and disassembly. (Not verified.)

## Design implications for Phase 1 (derived from V1–V7)

1. **The "signer present" gate can't work through account-meta flags.** If the relayer is an
   `ExtraAccountMeta` with `is_signer=true`, *every* transfer needs the relayer to sign, which
   breaks exit. With `is_signer=false`, the hook always sees `is_signer=false`. The workable option
   is to read the Instructions sysvar and check that the relayer pubkey shows up as a signer in a
   top-level instruction of the tx. That's sound because a signature commits to the whole message.
   I'll check this against the reference FOMO txs once RPC is available.
2. **Exit guarantee (option c): recommended.** Allow any transfer whose destination is the pool's
   `base_vault` with the owner as authority (that's a sell, V5), and gate only transfers whose
   source is `base_vault` (buys). With that rule, holders can always sell straight through DBC even
   if the relayer or router is down. That makes honeypotting structurally impossible (short of a
   hook program upgrade). The vault pubkey gets stored in the whitelist PDA when the pool is
   created.
3. **Router mode is weak gating.** Anyone can build a tx that calls a whitelisted router directly,
   so a program-ID check only keeps out people who don't bother. A relayer signature is the only
   gate that actually holds.
4. **W2W transfers:** this is your call. Allowing them keeps CEX/wallet moves working, and a gated
   holder can then pass tokens to an ungated wallet. Blocking them doesn't trap anyone as long as
   (2) holds. Default proposal: allow.
5. **Aggregators/GMGN** only work if they emit `swap2_with_transfer_hook` with the right
   remaining-account slices (V1, V6). Generic Jupiter-style routing of plain `swap2` will fail.
   I'll confirm this from real GMGN txs in Phase 3.
6. The hook goes away automatically when the curve completes (V3). Gating is curve-only by design.

## ASSUMED / NOT VERIFIED

- That the reference mint is Token-2022 with a TransferHook, plus its extensions, hook program ID,
  and mint/freeze authorities.
- That the reference pool uses a `ConfigWithTransferHook`.
- That FOMO relayer `AgmLJBMDCqWynYnQiPCuj9ewsNNsBJXyzoUhD9LJzN51` is the gate, and how it's checked.
- Whether the reference hook is source-verified, upgradeable, or has an admin whitelist.
- Buy/sell/failed tx CPI trees and logs.
- How FOMO lists tokens.

## Unblock

Allow at least one Solana RPC host (mainnet for forensics, devnet for Phase 4) in the
environment's network settings, e.g. `api.mainnet-beta.solana.com` and `api.devnet.solana.com`
(or a keyed provider), and also `solscan.io` / `api.solscan.io` for verification status.
