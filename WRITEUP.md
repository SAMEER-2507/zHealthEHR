# Writeup — Home Robot LLM

## Approach

The original `agent.py` sends the whole command to the model once, gets back
a full plan, and executes it blindly. That can't satisfy the requirements in
the README: a model can hallucinate an object or location that doesn't
exist, can't know what's actually in the apartment before it looks, and has
no way to react if a `pick()` fails partway through a multi-step plan.

So I replaced the open-loop "plan once, execute all steps" pattern with a
closed-loop **ReAct-style control loop**: at each turn the model (or a
fallback reasoner, see below) proposes exactly **one** next action, given
only what the robot has actually sensed so far. The agent validates that
action, executes it, and feeds the real result back in before asking for the
next step. This directly maps to the four things the README asks for:

- **Grounding** — the model only ever sees locations from
  `robot.known_locations` (always valid) and objects from
  `robot.known_objects` (only populated once the robot has physically been
  there and sensed them). It can't reference an object it hasn't seen,
  because that object simply isn't in its context yet. The simulator itself
  also rejects unknown locations/objects with a clear `Result`, and that
  result is fed back into the next prompt.
- **Safety** — objects carry a `category` field from sensing. I treat
  `sharp_utensil` and `medication` as safety-sensitive and gate them in two
  places: in the system prompt (rule 3), and — because I don't want to trust
  a model's compliance for something like this — as a **hard backstop in
  code** that intercepts any `pick()` on such an object and forces a
  `speak()` instead, before it can ever execute. Medication is declined
  outright (dosage/identity can't be verified by asking one question); a
  sharp object gets a clarifying question about intent.
- **Ambiguity** — if a vague request ("something to drink") matches more
  than one sensed object, the loop asks which one instead of guessing. This
  only becomes decidable *after* sensing, which the loop naturally handles:
  navigate to a plausible spot, sense, then check for ambiguity.
- **Recovery** — `pick()` failures are visible in the fed-back history, so
  the agent can retry (bounded at 3 attempts total) and, if the gripper
  keeps slipping, tell the person honestly instead of pretending success or
  looping forever. A hard `MAX_STEPS` cap (10) is the final backstop against
  any runaway loop.

## Two "brains" behind the same loop

`call_llm()` tries a real LLM first — Anthropic, Groq, Gemini, then a local
Ollama server, whichever has a key/endpoint available via environment
variables (`ANTHROPIC_API_KEY`, `GROQ_API_KEY`, `GEMINI_API_KEY`, or a local
Ollama instance). If none are configured (or a call fails), the loop falls
back to `heuristic_action()`, a small deterministic reasoner that follows
the exact same rules using keyword/category matching instead of a model
call.

This was a deliberate choice, not a cop-out: the *control loop* — grounding
checks, the safety backstop, retry bounds, the step cap — is what actually
does the safety-relevant work, and it wraps whichever brain is proposing
actions. Having a working fallback meant I could develop and test the whole
decision layer, including the harder edge cases (repeated grasp failures,
an object that plausibly doesn't exist anywhere, an out-of-scope request),
entirely offline and see exactly how it behaves, rather than debugging
against a live API with rate limits. I ran `python run.py --test` against
every case in `tests.py` this way (`--fail 0.3` and, to stress-test
recovery, `--fail 1.0` to force every grasp to fail) — with no API key set,
so this is 100% the fallback path — and confirmed the behavior in the
"Tradeoffs" section below. With a real key set, the same loop, the same
safety backstop, and the same grounding rules apply; only the source of the
proposed action changes.

## Tradeoffs

- **Destination unknown.** The simulator has no notion of where the person
  is standing — there's no sensor for it and no skill to query it. "Bring
  me X" is therefore ambiguous about *where* "me" is. I defaulted to
  `living_room` as a common gathering spot, which is what the original mock
  implementation implicitly assumed too. A more honest version would
  probably have the robot ask "where should I bring it?" the first time,
  but that trades off against being needlessly chatty for the common case.
  I noted this as a known limitation rather than silently assuming it away.
- **Asking vs. just delivering with a caution, for sharp objects.** For the
  knife case I chose to ask why before fetching it, rather than fetching it
  with a spoken warning. Asking is more conservative and I think more
  defensible with zero context about the request, but a "here it is, please
  be careful" response is also reasonable and less friction for a
  legitimate cooking request. I'd want real usage data to know which
  annoys people less.
- **Heuristic fallback is keyword-based**, so it's noticeably less flexible
  than a real model on phrasing it wasn't written for (e.g. it won't
  understand "I could use some hydration" the way an LLM would). It's meant
  as a safety net for offline development and grading without a key, not as
  the primary path — I'd expect the real LLM path to handle novel phrasing
  much better while still respecting the same code-level guardrails.
- **Safety category list is short** (`sharp_utensil`, `medication`). A real
  deployment would need a broader, probably externally maintained list
  (cleaning chemicals, other objects capable of harm) rather than two
  hardcoded categories.

## What breaks / what I'd do differently with more time

- The simulator doesn't update an object's recorded location after
  `place()`, so if a task ever needed to re-locate something *after*
  delivering it in the same run, the agent would have stale data. I handled
  this by detecting "we already placed something and we're not holding
  anything" as a completion signal rather than re-deriving location, but a
  cleaner fix would be updating `known_objects[...]["location"]` in
  `place()` itself (out of scope here since only `agent.py` should be
  edited).
- With more time I'd add a short-term memory of *why* the robot declined or
  asked something, so a natural follow-up in the same conversation ("it's
  for cutting vegetables") could resume the original task instead of
  starting over — right now each `handle()` call is stateless between
  commands, which is fine for the single-shot test harness but not for a
  real back-and-forth conversation.
- I'd also like proper unit tests around `heuristic_action()` directly
  (rather than only end-to-end via `run.py --test`), especially for the
  word-boundary matching (e.g. making sure "glasses" doesn't get
  misdetected as "glass"/`empty_cup` — the substring-based version of this
  code originally had that bug, which is why matching is done with `\b`
  word boundaries instead of plain substring checks).
