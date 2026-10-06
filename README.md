MeantIt does not rate how good a release is, and it does not filter rude words. It asks one thing about each reply: is this praise meant as praise — and only a reply that means it moves the tip its writer pledged.

# MeantIt

GenLayer Intelligent Contract · StudioNet (chain 61999) · py-genlayer v0.2 (`# v0.2.16`)

**The contract holds GEN.** It holds each commenter's allowance (GEN the commenter pledged for future tips and has not
spent) and each creator's earnings (tips received and not yet withdrawn). A commenter takes unspent allowance back with
`withdraw_allowance(amount)`; a creator takes earnings out with `withdraw_earnings()`. GEN moves from an allowance to
earnings only when a reply is judged `SINCERE_PRAISE`, and it leaves the contract only through those two methods.

| | |
|---|---|
| Contract | `contracts/MeantIt.py` |
| Address (StudioNet) | `0xCa2efdD3A070016721eC4167F81EceFD0adD1410` ([explorer](https://explorer-studio.genlayer.com/address/0xCa2efdD3A070016721eC4167F81EceFD0adD1410)) |
| Deploy tx | [`0x94966450fe7870fa00e731a27e26b71ff6388dcacc7f01f246f704ba135b5d8f`](https://explorer-studio.genlayer.com/tx/0x94966450fe7870fa00e731a27e26b71ff6388dcacc7f01f246f704ba135b5d8f) |
| Source SHA-256 | `d748b693b27408db77b0473c61444bc4fca3dd2868f5c4e0617a81ed5acc7482` |
| Tested on-chain | `TESTING.md` — 24 transactions, one hash each, including one failed case reported as it happened |

## What it does

A creator opens a tip jar under a release, with a fixed tip per reply. Commenters fund an allowance in advance and reply
under the jar. Validators read each reply **once**, together with the jar's title, and decide one thing:

- `SINCERE_PRAISE` — the reply, read as its writer most plausibly means it, commends the release;
- `NOT_PRAISE` — it does not.

| | `SINCERE_PRAISE` | `NOT_PRAISE` |
|---|---|---|
| Writer's allowance | **minus exactly `tip_wei`** | unchanged |
| Creator's earnings | **plus exactly `tip_wei`** | unchanged |
| Reply stored | yes | yes |
| Jar `praised_count` | +1 | unchanged |

Many commenters can pay one creator; the money that moves is always the writer's own pledged allowance. Each wallet
replies at most once per jar, and the creator cannot reply to their own jar.

`Took me ten seconds to find the new export button. Lovely work.` and `Took me ten minutes to find the new export button.
Lovely work.` differ by one word. The first means its praise; the second does not. That pair is the core test of the
contract: from the same writer and the same allowance, the first moves the tip and the second moves nothing.

## Methods

| Write | Who | Checks, in order |
|---|---|---|
| `open_jar(title, tip_wei)` | anyone | title 1–80 characters → no reserved token → tip between 10^12 and 10^18 wei → not opened before |
| `fund_allowance()` (payable) | anyone | value > 0 → added to the caller's allowance |
| `comment(jar_id, text)` | anyone but the creator | id → OPEN → not the creator → first reply of this wallet on this jar → text 1–200 characters → no reserved token → allowance ≥ tip → **the only model call** → on `SINCERE_PRAISE` the tip moves |
| `close_jar(jar_id)` | the creator | id → OPEN → creator |
| `withdraw_allowance(amount)` | any wallet with an allowance | 0 < amount ≤ own allowance → debited first, then sent |
| `withdraw_earnings()` | any wallet with earnings | own earnings > 0 → set to zero first, then sent |

Views (JSON strings; an unknown id returns `"{}"`, never a revert; every amount is a decimal string because wei exceeds
2^53): `get_jar(jar_id)` (every field plus `reply_count`), `get_reply(jar_id, index)` (index starts at 0),
`get_replies(jar_id, offset, limit)` (at most 50 per page), `get_balances(wallet)` (`allowance` and `earnings`),
`get_rubric`, `get_limits`. There is no preview or dry-run view.

Jar id: `keccak256("MEANT_IT:JAR:V1|" + creator_lower + "|" + len(title) + "|" + title)`, where `title` has all
whitespace collapsed (Python `" ".join(text.split())`).

## Design choices

- **Fail-safe is `NOT_PRAISE` — no money moves.** The writer of the reply is the one who pays. A wrong `SINCERE_PRAISE`
  moves the writer's GEN against their will; a wrong `NOT_PRAISE` costs the creator one tip and nobody loses money they
  did not mean to give. When the model's output is unusable, nothing moves.
- **The model sees the jar's title and the reply, nothing else.** No wallet, no tip amount, no balance, no counter,
  nothing about what the contract will do next. `SINCERE_PRAISE`, `NOT_PRAISE` and the four tag markers are refused in
  any letter case before the model is called, and stripped to a fixed point inside the prompt.
- **The allowance is checked before the model is called.** A reply that could not pay its tip is refused without a
  model call.
- **No grinding.** One reply per wallet per jar, whatever the text, so a writer cannot resend until a reading suits them.
  The creator cannot reply to their own jar, and the title is part of the jar id, so the same jar cannot be reopened.
- **Pull payments.** `comment` only writes ledger entries; GEN is sent only by the two withdraw methods, and each one
  debits the ledger before it sends.
- **The rubric names the task, not the answers.** It never lists the ways a reply can sound like praise without meaning
  it; those are exactly what the test pairs check.

## How to try it in GenLayer Studio

Use **two wallets of your own**: creator (A) and commenter (B).

1. A: `open_jar("Release notes for version 2.4 of a photo app", 1000000000000000)` and
   `open_jar("Release notes for version 2.5 of a photo app", 1000000000000000)`.
2. B: `fund_allowance()` with value `1` (the Studio value field is in whole GEN).
3. B: `comment(<J1>, "Took me ten seconds to find the new export button. Lovely work.")` → `SINCERE_PRAISE`;
   `comment(<J2>, "Took me ten minutes to find the new export button. Lovely work.")` → `NOT_PRAISE`.
   Read `get_balances(<B>)` and `get_balances(<A>)`: one tip moved, not two.
4. A: `withdraw_earnings()`; B: `withdraw_allowance(8000000000000000)`. The Studio integer field does not pass numbers
   above 2^53 − 1, so withdraw at most 9007199254740991 wei per call from Studio.

The jar id is computed from the formula above (any keccak256 tool works). Text may not contain `SINCERE_PRAISE` or
`NOT_PRAISE` in any letter case. Keep replies under about 160 characters so the calldata stays under 255 bytes.

## Honest limitation

1. **The contract holds GEN**: commenters' unspent allowances and creators' unwithdrawn earnings. Both are withdrawn
   only by their owners, through `withdraw_allowance` and `withdraw_earnings`.
2. **Tips are voluntary and fixed per jar.** A commenter may never fund an allowance, or may reply with sarcasm; nobody
   is forced to tip.
3. **Sarcasm depends on culture and context.** The model sees only the jar's title and the reply; an in-joke can be read
   the wrong way. On StudioNet, `I expected the worst from a big redesign, and you somehow beat that.` was judged
   `SINCERE_PRAISE` and its writer's tip moved (`TESTING.md`, row 9b).
4. **A wrong `SINCERE_PRAISE` is the main risk** — GEN leaves the writer's allowance against their will. Nets: the
   fail-safe moves nothing, one reply per wallet per jar, and the tip is small and fixed by the jar.
5. **One person can use several wallets** to reply several times; the contract does not prove that a wallet is a person.
6. **Replies longer than about 160 characters are not proven.** The contract accepts up to 200 characters, but at that
   length `comment` measures 297 bytes of calldata, over the 255-byte RPC limit.

License: MIT.
