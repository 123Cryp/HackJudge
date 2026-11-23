# HackJudge

A hackathon prize pool judged by an LLM against a public rubric, with appeals, on GenLayer.

A sponsor locks a prize pool in GEN and publishes a rubric of up to five weighted criteria. Teams enter with a GitHub repository, a demo link and a small bond. After the deadline anyone calls `judge()`: validators fetch each README and demo page, and the LLM scores every criterion from 0 to 10 with a short reason. The ranking splits the prize 50 / 30 / 20. A team can appeal its own score once by posting a bond. `finalize()` pays the winners; unclaimed shares go back to the sponsor.

## Files

| File | Purpose |
| --- | --- |
| `hackjudge.py` | Intelligent Contract (`HackJudge`) |
| `index.html` | Single-file frontend: home, hackathons, host, hackathon and entry pages, my account |
| `tests/test_hackjudge.py` | Offline test suite with a GenLayer SDK stub |
| `.github/workflows/tests.yml` | GitHub Actions workflow that runs the tests |
| `.github/workflows/unzip.yml` | GitHub Actions workflow that unpacks an uploaded zip inside the repo |

## Live deployment

| | |
| --- | --- |
| Contract | `ADDRESS` |
| Explorer | https://explorer-studio.genlayer.com/address/ADDRESS |
| Network | GenLayer Studio |

## Flow

1. **Create.** `create_hackathon` with the prize attached as value. Arguments: title, brief, rubric JSON, submission minutes, appeal minutes, entry bond (GEN), appeal bond (GEN).
2. **Enter.** `submit_entry` with the entry bond attached. One entry per address and per repository. An optional full 40-character commit SHA pins the README that is judged.
3. **Judge.** After the deadline anyone calls `judge`. Each call scores up to two entries (`judge_batch`), so call it again until nothing remains. The prompt carries the rubric, the README (up to 6000 characters) and the demo page (up to 3000). The page text is treated as untrusted. If neither page can be read the entry is `UNREACHABLE` and its bond joins the prize pool.
4. **Appeal.** During the appeal window the team that owns an entry can call `appeal` once with the appeal bond. The re-score runs from scratch. If the weighted total moves by 10 points or more (on the 100-point scale), the re-score replaces the score and the bond is sent back. Otherwise the original score stands and the bond joins the prize pool.
5. **Finalize.** After the appeal window anyone calls `finalize`. The top three scoring entries get 50, 30 and 20 percent of the pool, entry bonds of scored entries are refunded, and any unused share returns to the sponsor.

Scores are stored on a 0 to 1000 scale (weight x score), shown as 0 to 100 points.

## Consensus

Leader and validators each fetch the pages and call the model with the same prompt. Output is normalized to JSON with integer scores 0 to 10. Validators accept the leader when the result type matches, the weighted total is recomputed correctly, every criterion is within 2 points of their own score and the total is within 10 points.

## Payments

Calls that carry GEN and fail validation return a refund credit instead of reverting, and the value is credited to the caller's balance. Overpayments are credited too. Withdraw with `withdraw`. Prize, bond and sponsor payouts are sent directly by `finalize`, `cancel_hackathon` and a successful `appeal`.

## Views

`get_config`, `get_stats`, `get_accounting`, `get_activity`, `list_hackathons`, `get_hackathon`, `get_entries`, `get_entry`, `get_leaderboard`, `get_account`, `get_owed`.

`get_accounting` reports the contract balance against GEN held in hackathons plus refunds owed; the home page shows the result as the Balanced check.

## Run the tests

```
python3 -m unittest discover -s tests -v
```

## Deploy and try it

1. Open GenLayer Studio, deploy `hackjudge.py` (no constructor arguments) and copy the address.
2. In `index.html` set `DEFAULT_CONTRACT_ADDRESS` to that address, or open the page with `?contract=0xYourAddress`, or paste the address in the wallet menu.
3. Host a hackathon with a short submission window (2 minutes is the minimum) and walk through the flow.

## Uploading from a phone

The workflow in `.github/workflows/unzip.yml` unpacks a project zip: upload the project zip to the repository root. The workflow unpacks it, removes the zip and commits the files. Do the same with `ci/tests.yml` to run the tests on every push.
