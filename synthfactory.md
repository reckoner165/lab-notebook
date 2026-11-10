## synth factory

code: ~/Documents/dev/synth-factory

RL for sound design in SuperCollider. the idea is to teach a model to write SuperCollider synths that sound good, using the same loop Surya used to teach Qwen to paint.

### where the idea comes from

Surya Narreddi's post "Training AI to Paint with Code" (surya.website/rling-qwen-to-paint-with-code). they RL-trained Qwen to write p5.brush JavaScript that renders watercolour paintings:

1. model gets a prompt ("peach hibiscus in watercolour") and writes a full sketch
2. sketch is rendered headless (Puppeteer) to a PNG
3. a judge model compares the PNG against two references from a hand-rated pool and picks the better one
4. win rate becomes the reward, GRPO updates the model

lessons from the post that shape this:

- **pairwise judging beats absolute scoring.** 0–10 scores came out compressed and useless. "which of these is better" gave a usable signal.
- **too many overlapping reward signals hurt.** 9 signals (5 measuring the same thing) → collapsed to 4 (compile gate, length check, human-preference model, pairwise judge) and it trained 3x faster.
- **a hand-rated reference pool anchors taste.** Surya rated 1,664 generations love / okay / nope; the 117 "love" ones seeded the comparison pool.
- **short allowlists beat long API docs.** a 400-line API reference made models invent APIs. a strict list of 8 allowed methods fixed it.

### decisions so far

- **medium: SuperCollider, non-realtime (NRT) rendering.** sclang builds a Score (timed list of server commands), `scsynth -N` renders it straight to a WAV. no audio device, faster than realtime, one process per render so they parallelize.
  - picked over Python (fluent but raw DSP sounds thin) and Rust (models know it less, compiling every rollout is slow)
  - headless Linux needs `QT_QPA_PLATFORM=offscreen` for sclang
- **the model never handles rendering.** it writes the synth, a harness wraps, renders and checks it.

### current step: generate and listen

Surya's advice: skip training for now. generate tightly scoped synths, render 30s outputs, get a feel for what models produce. then have LLMs describe the audio in the vocabulary of music journalism / art criticism and see where to go from there.

**the constrained kit**, same for every model and every prompt, so differences come from the model's choices and not its tools:

- 2 oscillators
- 2 filters
- 1 noise source
- 1 envelope
- 2 LFOs
- 1 delay

LFOs and envelope matter: over 30s the interest comes from change. without modulation most outputs will be flat drones that all sound alike.

prompt shape: "You have 2 oscillators, 2 filters, 1 noise generator, an envelope, a delay and 2 LFOs. Use these to generate 30s of audio that <creative prompt>, in SuperCollider."

**enforcing the kit:**

- name the exact UGens allowed per part (oscillators: `SinOsc`, `Saw`, `Pulse`; filters: `RLPF`, `BPF`; ...), not just "2 oscillators". otherwise models reach for reverbs and extras.
- no arrays as UGen arguments. `SinOsc.ar([100, 150, 200])` silently makes three oscillators (multichannel expansion).
- after compiling, the SynthDef can list its UGens. check that list against the allowlist and count instances per type.

**generation run:**

- start with Opus 5.5, compare other models on the same prompts later
- 10–20 creative prompts, several outputs each, ~100 total to start
- render each to a 30s WAV through the harness
- mechanical checks (all automatable): rendered? within kit? silent? clipping? NaN? wrong length? failures saved with the real error output
- save each output with prompt, model, .scd source and WAV, grouped by prompt for listening

### later (open, not decided)

- **listening + rating.** I listen and rate love / okay / nope. small local page: one render at a time with its prompt, keyboard to rate.
- **LLM descriptions.** audio-capable LLM (e.g. Gemini) describes each output in music / critical vocabulary. does it match what I hear? shows which qualities an automated judge could pick up.
- **judge agreement test.** before any RL: ~50 pairs I've ranked, how often does a pairwise judge agree? well below ~80% → RL just learns the judge's mistakes. fallback: train a small reward model on my own ratings.
- **possible reward:** pairwise judge vs hand-rated reference pool (biggest weight) + CLAP text-audio similarity + the mechanical gates.
- **tooling:** Prime Intellect's `verifiers` can package prompts + render harness + scoring as an environment for evals and RL (prime-rl). not needed yet, a plain script is fine until it's clear what to score.
- **training:** GRPO on a Qwen coder base model.

---

## log

#### 2026-11-09
- carried context over from the planning conversation (handover doc) into Claude Code
- made `~/Documents/dev/synth-factory`, empty so far, not a git repo yet
- SuperCollider.app is installed in /Applications, but `sclang` / `scsynth` aren't on PATH. will call them by full path inside the app bundle
- started this notebook entry

- agreed on how outputs are saved. a "sample" = one synth the model wrote + its one 30s WAV. several samples per prompt, because the same prompt gives a different synth each time:
  ```
  runs/<date>_<run-name>/
    kit.json, system_prompt.md      (fixed for the whole run)
    <prompt-slug>/sample-01/
      prompt.txt, meta.json, response.txt, synth.scd, render.scd, output.wav, render.log, checks.json
  ```
- NRT smoke test works on the mac: sclang → scsynth -N → WAV, ~1.6s per 30s render. sclang prints the SynthDef's UGen list, so the kit check works
- built the harness (python + uv): `sf.py generate | add | recheck`, kit allowlist in `kit.json`, system prompt template in `kit_prompt.md`, prompts go in `prompts.txt`
- tested with 3 hand-written synths: valid one passes; one with FreeVerb + `SinOsc.ar([110,165,220])` fails as "not in kit: FreeVerb" and "too many oscillators: 3 > 2"; one with a syntax error fails as "compile failed"
- gotchas found:
  - `1.exit` in sclang doesn't stop the script immediately, so later code still ran and printed a second, wrong error. fixed by nesting the stages
  - `.clip` and `.linexp` create their own UGens (`Clip`, `LinExp`), not just math. added them (+ `LinLin`) to the always-allowed list
- no refusal fallback on API calls on purpose: a fallback swaps in a different model, which would mix models in an eval
- first API test: 1 sample for "create bird song in a forest with crunchy leaves and crickets". passed all checks, used the full kit, ~70s / 6.4k output tokens
- Claude wrote `claude-review.md` for it from a spectrogram + measurements (it can't hear audio). main finding: a 4.7 kHz cricket whine dominates all 30s, birds only ~8s, no low end. the model's code comments describe intent, not the result
- built the review site: `uv run sf.py review` → http://127.0.0.1:8765. listen, rate love / neutral / hate (keys 1/2/3), notes, LLM reviews collapsed. saves `rating.json` per sample
- API key now lives in `.env` (template `.env.example`)
- signal flow diagram on the review page: sclang dumps the compiled SynthDef graph to `graph.json`, math UGens are folded away, kit parts drawn as boxes (Mermaid). solid = audio, dashed = modulation, labels = input reached (freq, in, amp) + shaping (abs, ^32, tanh). `uv run sf.py graph` backfills old samples without re-rendering

TODO (next):
- review the UGen allowlist in kit.json (my first draft) and the system prompt in kit_prompt.md
- write 10–20 creative prompts in prompts.txt
- set ANTHROPIC_API_KEY, then first run: `uv run sf.py generate --run first --samples 5`
