## 0. CHOMP / stomp.gg

**stomp.gg** is the game; **CHOMP** (Credibly Hackable On-chain Monster PvP) is the on-chain engine that powers it. Throughout this doc "CHOMP" refers to the engine and its surrounding contracts in this repo — what stomp.gg players are actually battling on under the hood.

CHOMP is a turn-based PvP monster-battling game that runs entirely on-chain in Solidity. It's inspired by Pokémon Showdown and M.U.G.E.N: two players each bring a **team of "mons"** (creatures with HP, stamina, speed, attack/defense stats, one or two elemental types out of 16, four moves, and one ability), and battle one mon at a time until one side is wiped out.

**Objective:** knock out every mon on the opposing team. Damage depends on the move's power, attacker stats, defender stats, and the type chart. Moves cost **stamina** (the global `StaminaRegen` effect refunds 1/turn) and have **priority** (higher goes first; speed breaks ties). Many moves apply **status effects** (burn, frostbite, sleep, etc.) or **stat boosts** that persist across turns via the engine's lifecycle hooks.

Players come in two flavors: humans drive moves through the commit-reveal flow described below; CPU contracts (`src/cpu/`, implementing `ICPU`) pick moves algorithmically. The engine treats them identically.

You — the agent reading this — are here to **drive a player through a full match**: pick a team, start a battle, and run the commit/reveal/execute loop turn after turn until `Engine.getWinner(battleKey)` returns a non-zero address.

---

## 1. The 30-second mental model

A battle is a sequence of **turns**. Each turn, both players pick one of:
- a **regular move** (index `0..3`) — must have stamina
- a **switch** (index `125`, with `extraData = target mon index`)
- a **no-op** (index `126`)

Moves are submitted via **commit-reveal** so neither player sees the other's choice early. Once both moves are revealed, anyone calls `Engine.execute(battleKey)` to resolve the turn. The engine runs priority/speed ordering, damage, effects, KO checks, and forced switches, then advances to the next turn.

Battle ends when one side has no living mons. `Engine.getWinner(battleKey)` returns the winning address (or `address(0)` if still in progress).

---

## 2. Key contracts

| Role | Contract | Notes |
|---|---|---|
| Battle engine | `Engine` (`IEngine`) | State, execution, all getters |
| Matchmaker | `DefaultMatchmaker` or `SignedMatchmaker` | Propose/accept/start |
| Commit manager | `DefaultCommitManager` or `SignedCommitManager` | Per-turn move submission |
| Team registry | `DefaultTeamRegistry` / `LookupTeamRegistry` / `GachaTeamRegistry` (`ITeamRegistry`) | Stores teams per player |
| Mon registry | `DefaultMonRegistry` (`IMonRegistry`) | Stats, legal moves, abilities per mon id |
| Validator | `DefaultValidator` (`IValidator`) | Team-size, move-legality, timeout rules |

Deployed addresses live in the env produced by `processing/createAddressAndABIs.py` (see deployment artifacts under `out/` and `processing/`).

---

## 3. Where to get mon info

**Off-chain (preferred for browsing):** read `drool/mons.csv`, `drool/moves.csv`, `drool/abilities.csv`, `drool/types.csv`. These are the source of truth for stats and balance.

**On-chain (for what's actually deployed):**
```solidity
IMonRegistry reg = teamRegistry.getMonRegistry();
uint256 count = reg.getMonCount();
uint256[] memory ids = reg.getMonIds(0, count);

// One mon: stats + legal move addresses + legal ability addresses
(MonStats memory stats, uint256[] memory moves, uint256[] memory abilities)
    = reg.getMonData(monId);

// Batch
(MonStats[] memory s, uint256[][] memory m, uint256[][] memory a)
    = reg.getMonDataBatch(ids);
```

`MonStats` = `{ hp, stamina, speed, attack, defense, specialAttack, specialDefense, type1, type2 }`. Move/ability entries are addresses in their lower 160 bits (upper bits may pack inline data — treat as opaque ids unless you need to call them directly).

To inspect a player's existing teams:
```solidity
ITeamRegistry tr = ITeamRegistry(...);
uint256 n = tr.getTeamCount(player);
Mon[] memory team = tr.getTeam(player, teamIndex);
uint256[] memory monIds = tr.getMonRegistryIndicesForTeam(player, teamIndex);
```

---

## 4. Starting a battle (DefaultMatchmaker)

Three-step propose / accept / confirm flow. P0 hides their team behind a hash so P1 can't counterpick.

**4a. P0 proposes** (commits to a team hash):
```solidity
bytes32 salt = randomBytes32();
uint96  p0TeamIndex = ...;
uint256[] memory p0Indices = teamRegistry.getMonRegistryIndicesForTeam(p0, p0TeamIndex);
bytes32 p0TeamHash = keccak256(abi.encodePacked(salt, p0TeamIndex, p0Indices));

ProposedBattle memory pb = ProposedBattle({
    p0: p0,
    p0TeamIndex: 0,                // ignored in 3-step flow (confirmBattle re-supplies it); set to the real index for fast-battle
    p0TeamHash: p0TeamHash,
    p1: p1,                        // or address(0) for an open challenge
    p1TeamIndex: 0,                // matchmaker overwrites
    teamRegistry: teamRegistry,
    validator: validator,
    rngOracle: rngOracle,
    ruleset: ruleset,
    moveManager: address(commitManager),
    matchmaker: matchmaker,
    engineHooks: hooks
});
bytes32 battleKey = matchmaker.proposeBattle(pb);
```

**4b. P1 accepts** with their team index and an integrity hash of the proposal:
```solidity
bytes32 integrity = matchmaker.getBattleProposalIntegrityHash(pb);
bytes32 keyToUse  = matchmaker.acceptBattle(battleKey, p1TeamIndex, integrity);
// keyToUse == battleKey unless this was an open (p1=address(0)) proposal
```

**4c. P0 confirms** by revealing salt + team index. This calls `Engine.startBattle()` and the battle is live.
```solidity
matchmaker.confirmBattle(keyToUse, salt, p0TeamIndex);
```

**Fast-battle shortcut:** if P0 sets `p0TeamIndex` to the real index in `proposeBattle` and uses `p0TeamHash = FAST_BATTLE_SENTINAL_HASH`, `acceptBattle` starts the battle immediately and `confirmBattle` is skipped. Only do this when hiding the team is unnecessary.

`SignedMatchmaker` is the EIP-712 variant — same conceptual flow, but proposals are signed off-chain. See `src/matchmaker/SignedMatchmaker.sol` for the typed-data layout.

---

## 5. The turn loop

Before doing anything each turn, fetch context:
```solidity
CommitContext ctx = engine.getCommitContext(battleKey);
// ctx.turnId, ctx.playerSwitchForTurnFlag, ctx.winnerIndex, ctx.startTimestamp, ctx.p0, ctx.p1
```

Decision tree:
- `ctx.startTimestamp == 0` → battle not started yet.
- `ctx.winnerIndex != 2` → battle is over; `engine.getWinner(battleKey)` for the address.
- `ctx.playerSwitchForTurnFlag == 0` → **only P0 acts this turn** (forced switch after KO). P0 reveals directly, no commit.
- `ctx.playerSwitchForTurnFlag == 1` → **only P1 acts this turn**. Same — P1 reveals directly.
- `ctx.playerSwitchForTurnFlag == 2` → **normal two-player turn**. The committer is `P0 if turnId % 2 == 0 else P1`; the other player is the revealer.

### Building a move

```solidity
uint8   moveIndex;  // 0..3, 125 (SWITCH), or 126 (NO_OP)
bytes32 salt = randomBytes32();
uint240 extraData;  // for SWITCH: target mon index. For most attacks: 0.

bytes32 moveHash = keccak256(abi.encodePacked(moveIndex, salt, extraData));
```

Constants (`src/Constants.sol`): `SWITCH_MOVE_INDEX = 125`, `NO_OP_MOVE_INDEX = 126`. Regular move slots are `0..3`. Some moves require an `ExtraDataType` target — check the move's `extraDataType()` (see CLAUDE.md "Move Implementation Patterns").

### Picking a legal move

```solidity
ValidationContext vc = engine.getValidationContext(battleKey);
bool ok = engine.validatePlayerMoveForBattle(battleKey, moveIndex, playerIndex, extraData);
```
The validator checks stamina, KO state, and any move-specific rules. Always validate before committing — a bad reveal will revert and waste a turn slot.

---

## 6. Commit / reveal — `DefaultCommitManager`

Standard 3-tx flow on a normal two-player turn (committer = C, revealer = R):

```solidity
// TX 1 — committer hides their move
commitManager.commitMove(battleKey, moveHash);

// TX 2 — revealer submits their plaintext move (no commit needed; their move is now public)
commitManager.revealMove(battleKey, rMoveIndex, rSalt, rExtraData, /*autoExecute=*/ false);

// TX 3 — committer reveals preimage; autoExecute=true runs Engine.execute() in the same tx
commitManager.revealMove(battleKey, cMoveIndex, cSalt, cExtraData, /*autoExecute=*/ true);
```

On a **single-player turn** (`playerSwitchForTurnFlag` is 0 or 1), the active player just calls `revealMove(...)` directly. No commit phase. Pass `autoExecute=true` to advance immediately.

Ordering rules enforced by the contract:
- The revealer cannot reveal before the committer has committed (turn ≥ 1).
- The committer cannot reveal before the revealer has revealed.
- Each player can only reveal once per turn.
- `playerSwitchForTurnFlag` decides who's allowed to commit at all.

Inspect state:
```solidity
(bytes32 hash, uint256 turnId) = commitManager.getCommitment(battleKey, player);
uint256 revealed = commitManager.getMoveCountForBattleState(battleKey, player);
uint256 lastTs   = commitManager.getLastMoveTimestampForPlayer(battleKey, player);
```

If `Engine.execute()` is not auto-called, anyone can call it once both moves are revealed.

---

## 7. Commit / reveal — `SignedCommitManager`

`SignedCommitManager` **extends** `DefaultCommitManager`, so the 3-tx flow above still works as a fallback. Its value-add is two off-chain signing flows.

### 7a. Dual-signed reveal (1 tx instead of 3)

Both players sign their moves off-chain; the committer submits everything in one transaction.

1. **Committer (A) signs** their move hash off-chain (EIP-712 `SignedCommit`):
   ```
   SignedCommit(bytes32 moveHash, bytes32 battleKey, uint64 turnId)
   ```
   A sends `(moveHash, signature)` to B out-of-band.

2. **Revealer (B) signs** their move + A's hash off-chain (EIP-712 `DualSignedReveal`):
   ```
   DualSignedReveal(
     bytes32 battleKey,
     uint64  turnId,
     bytes32 committerMoveHash,   // A's hash, copied from above
     uint8   revealerMoveIndex,
     bytes32 revealerSalt,
     uint240 revealerExtraData
   )
   ```
   B sends the plaintext move + signature back to A.

3. **A submits the single tx:**
   ```solidity
   signedCommitManager.executeWithDualSignedMoves(
       battleKey,
       aMoveIndex, aSalt, aExtraData,
       bMoveIndex, bSalt, bExtraData,
       bSignature
   );
   ```
   The contract verifies B's signature against A's freshly-computed move hash, then calls `Engine.executeWithMoves(...)` directly. No on-chain commit/reveal storage is touched.

EIP-712 domain: `name = "SignedCommitManager"`, `version = "1"`, chainId + verifyingContract per usual. Type hashes live in `src/commit-manager/SignedCommitLib.sol`.

**Security note:** A commits to her hash before seeing B's move. B signs *over* A's hash, so even though A sees B's move before submitting, A cannot swap her own move (the preimage must match the hash B signed).

### 7b. Forced commit fallback

If A signs a `SignedCommit` and then stalls, **anyone** can publish A's commitment on-chain so the normal reveal flow can proceed:
```solidity
signedCommitManager.commitWithSignature(battleKey, moveHash, aSignature);
```
Then continue with `revealMove(...)` as in §6.

If B refuses to sign at all, A just falls back to `commitMove(...)` (the inherited Default flow) and eats the 3-tx cost.

---

## 8. Reading battle state mid-turn

Common queries during play:

```solidity
// Whose mon is active right now?
uint256[] memory active = engine.getActiveMonIndexForBattleState(battleKey); // [p0Active, p1Active]

// Current HP / stamina / stat deltas
int32 hpDelta = engine.getMonStateForBattle(battleKey, playerIndex, monIndex, MonStateIndexName.Hp);
MonStats memory base = engine.getMonStatsForBattle(battleKey, playerIndex, monIndex);

// What moves does this mon have? (returns the move address packed in a uint256)
uint256 moveAddr = engine.getMoveForMonForBattle(battleKey, playerIndex, monIndex, slot);

// KO bitmap — bit i set ⇒ mon i on this side is knocked out
uint256 koBits = engine.getKOBitmap(battleKey, playerIndex);

// Effects on a mon (or global, with targetIndex = 2)
(EffectInstance[] memory effs, uint256[] memory extras) =
    engine.getEffects(battleKey, playerIndex, monIndex);

// Active player's most-recent revealed move (after reveal, before/after execute)
MoveDecision memory md = engine.getMoveDecisionForBattleState(battleKey, playerIndex);
```

For richer context in one call, prefer `getBattleContext`, `getDamageCalcContext`, or `getValidationContext` over many small getters.

### Events to subscribe to

A reactive bot should listen for these instead of polling. All include `bytes32 indexed battleKey`.

| Contract | Event | Fires when |
|---|---|---|
| `Matchmaker` | `BattleProposal(battleKey, p0, p1, isFastBattle, p0TeamHash)` | P0 calls `proposeBattle` |
| `Matchmaker` | `BattleAcceptance(battleKey, p1, updatedBattleKey)` | P1 calls `acceptBattle` |
| `Engine`     | `BattleStart(battleKey, p0, p1)` | Battle goes live (after `confirmBattle` or fast accept) |
| `CommitManager` | `MoveCommit(battleKey, player)` | A player commits this turn — your cue to reveal |
| `Engine`     | `EngineExecute(battleKey, turnId, playerSwitchForTurnFlag, priorityPlayerIndex)` | Turn resolved — check the new flag and decide whether you owe a commit or a reveal |
| `Engine`     | `BattleComplete(battleKey, winner)` | Game over |

**Gotcha:** `MoveReveal` is declared but commented-out in `DefaultCommitManager`, so reveals do **not** emit a dedicated event. Detect them via `EngineExecute` (turn advanced ⇒ both moves revealed) or by polling `getMoveCountForBattleState(battleKey, player)` against the current `turnId`.

---

## 9. Constants & gotchas (cheat sheet)

- **Move-index storage quirk:** regular moves are stored on-chain as `+1` to disambiguate from "unset", but **the value you pass to `commit`/`reveal` is the raw `0..3`**. Don't double-shift.
- **Switch target rules:** `extraData = target mon index` and the target must not be KO'd or currently active.
- **Priority:** `DEFAULT_PRIORITY = 3`; switches use `SWITCH_PRIORITY = 6`. Higher goes first, speed breaks ties.
- **Stamina:** `DEFAULT_STAMINA = 5`. Each move costs its `STAMINA_COST`; `StaminaRegen` refunds 1/turn.
- **Crit:** `DEFAULT_CRIT_RATE = 5` (out of 100), crit multiplier `3/2`.
- **Claiming a timeout win:** if the opponent stalls past `TIMEOUT_DURATION` (or the whole battle exceeds `MAX_BATTLE_DURATION = 1 hours`), call `Engine.end(battleKey)` — anyone can call it. It runs `validator.validateTimeout` for both players and ends the battle in the non-stalling player's favor (or P0's favor on the absolute-duration fallback).
- **`autoExecute`:** safe to set `true` on the *second* reveal of a two-player turn or the only reveal of a single-player turn — the contract gates the `Engine.execute()` call internally.
- **Salt reuse:** never reuse a salt. Use a CSPRNG. The hash is `keccak256(abi.encodePacked(uint8 moveIndex, bytes32 salt, uint240 extraData))` — note `abi.encodePacked` and the exact types.
- **KO check:** trust `MonState.isKnockedOut`, not HP. After a clear, the HP delta sits at `CLEARED_MON_STATE_SENTINEL` (`int32.max - 1`) — don't compare HP to zero blindly.

---

## 10. Pointers

- `CLAUDE.md` — codebase conventions, building mons/effects, repo layout
- `src/Engine.sol` — authoritative source for state mutation and execute ordering
- `src/commit-manager/SignedCommitLib.sol` — EIP-712 type hashes for signed flows
- `drool/*.csv` — game data source of truth
- `docs/outline.md` — design notes
- `test/abstract/BattleHelper.sol` — concrete end-to-end example of the propose → accept → confirm → commit-reveal-execute loop in Solidity
