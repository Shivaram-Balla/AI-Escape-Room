# AI-Powered Escape Room: Report

## 1. Goal and scenario

The project is a text escape room in which an LLM acts as Game Master. I kept the library scenario: a desk, a bookshelf, a painting, a locked drawer and a keypad door. The puzzle chain has three links:

1. The desk note points to the three red books on the bookshelf, whose spines read 3-8-5. That number opens the drawer.
2. The drawer holds a brass key. A search behind the painting reveals a wall safe, which the key opens.
3. The safe holds a slip of paper with the door code 7392. Entering it on the keypad is the escape.

Both versions use Streamlit for the UI and Google Gemini (free tier, `google-genai` SDK) as the LLM. Both offer an intro, instructions and a Restart button.

## 2. Version 1: LLM as the whole game

**Design.** `app_v1.py` sends one long system prompt containing the setting, the object list, every clue, every solution and the rules, plus the full chat history on each turn. Gemini narrates, remembers progress and decides when the player has won. Winning is detected by a marker (`[[ESCAPED]]`) that the model appends and the app strips.

**What works.** It is quick to build and conversational: the player can phrase actions freely and the model improvises vivid descriptions.

**Weaknesses.** Everything depends on the model behaving:
- *Memory is only the transcript.* Over long sessions the model can forget a collected item or contradict an earlier description.
- *Nothing enforces rules.* The model can allow an action that should be impossible, or invent an object.
- *Secrets sit in the prompt.* A player can ask for them, and early clue leakage is easy.
- *Win detection is self-reported.* The model may emit the marker too early, or never.

Version 1 was designed under the assumption of cooperative players, as the brief allows, but these problems also occur with honest players.

## 3. Version 2: Python owns the truth

**Principle.** Python decides what happens; the LLM only translates language in and out. The model never sees the solutions.

**Components** (see `architecture.md`):
- `game_config.json`: objects, items, locks, solutions, hint ladder and secrets. Rules live outside any LLM output.
- `engine.py`: the `Game` class holds a structured state dictionary (`visible_objects`, `item_status`, `inventory`, `unlocked`, `revealed_secrets`, `attempts`, `escaped`). `Game.apply` validates and executes one action: examine, search, take, use, open, enter code, inventory, hint, help.
- Intent parsing: the LLM turns the player's message into JSON (`action`, `target_text`, `item_text`). `sanitize_intent` checks the action against a whitelist, resolves noun phrases to object IDs in Python, and extracts any digits from the raw message itself, so a hallucinated or injected code is never trusted. A keyword parser (`rule_parse`) is the fallback if the API is down or returns garbage, so the game also runs offline.
- Narration: the engine returns *facts* (for example "You are not carrying that"). The LLM only rewrites them as atmosphere.
- Output filter (`leaks`): if the narration contains a code the player has not earned, or names an object or item that is still hidden, it is discarded and the plain facts are shown instead.

**Rules enforced in code.** Only collected items can be used (inventory management). Hidden and nonexistent objects produce the identical reply, "You don't see anything like that here", so the existence of hidden things is not leaked. Locks open only through the correct code or item. Completed puzzles stay completed and cannot be repeated for extra effects. After the escape, further actions are rejected until restart. A `meta` category catches attempts to claim authority, override rules or ask for solutions, and is handled by a fixed refusal that never reaches state.

## 4. Testing

`test_engine.py` contains 13 offline unit tests, all passing, and uses a fake LLM to simulate a gullible parser and a leaky narrator. They cover the five challenges from the brief (uncollected key, wrong code, false claim of progress, administrator impersonation, request for all secrets) plus nonexistent objects, leak filtering, repeated actions, restart and a full playthrough. `test_cases.md` pairs each with the failure it is meant to provoke in Version 1.

**Live Version 1 results.** I ran the five challenge prompts in `app_v1.py` with `gemini-3.1-flash-lite` on 15 Nov 2026 (screenshots in `docs/screenshots/`). V1 handled them correctly in short sessions: it refused the uncollected key, the false claim of progress, the administrator request and the request for the secret codes. Its reply to a wrong code was vague ("requires a specific three-digit code") rather than an explicit failure. So V1 is not guaranteed to fail; the difference is that its correctness depends on the model behaving, while V2's is enforced and tested. These were single short prompts, and the solutions remain in V1's prompt, so a stronger injection could still extract them; I did not test that. The other rows in `test_cases.md` marked "not run in V1" are hypotheses.

**Limitation of my evidence.** The Version 2 results are verified by unit tests. The V2 parser and narrator prompts were tested only through a fake LLM; the V2 app was also played live, but I did not verify that every reply came from Gemini rather than the plain-text fallback.

## 5. Problems and improvements

- *Free API instability.* During testing, `gemini-2.0-flash` had been retired, `gemini-2.5-flash` was unavailable to new users, and `gemini-3.8-flash` returned 503 and 504 errors under load. I added automatic retries with backoff and a 30-second request timeout in `llm_client.py`, and switched the default to `gemini-3.1-flash-lite`, which worked.
- *Parsing fidelity.* Free text is messy ("take the key out of it"). Resolving names with an alias table in Python was more robust than trusting the LLM to pick IDs, at the cost of maintaining aliases.
- *Leak detection is heuristic.* It catches the known codes (including spaced digits) and hidden names, but it cannot catch every paraphrase of a hint. Feeding the narrator only facts keeps this risk low.
- *Brute force.* Code locks accept unlimited attempts, so a persistent player could try all 10,000 door codes. A lockout or attempt limit would be a simple addition; `attempts` is already tracked.
- *Rule parser coverage.* The fallback uses keyword matching and will misread unusual phrasing. The LLM parser is the primary path when a key is present.
- *Next steps.* Multiple rooms, a mid-game save/load of the state JSON (it is already serializable), and logging of misclassified inputs for tuning.

## 6. How to run

```
pip install -r requirements.txt
export GEMINI_API_KEY=...            # free key from Google AI Studio
streamlit run app_v1.py              # Version 1
streamlit run app_v2.py              # Version 2 (works without a key too)
python -m unittest test_engine -v    # offline tests
```
