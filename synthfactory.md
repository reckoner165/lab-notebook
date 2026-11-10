## synth factory

code: ~/Documents/dev/synth-factory

RL for sound design in SuperCollider. the idea is to teach a model to write SuperCollider synths that sound good, using the same loop Surya used to teach Qwen to paint.

### why I'm doing this

modern AI models are awful at producing music, and they take the joy out of making it for a musician. but parts of the music-making process, especially synthesis and sound design, can be reframed as a coding problem. that opens them up to algorithmic and procedural music generation tools and ideas, and lets a machine make interesting sounds. the results can be judged by ear, or described in the language of contemporary music criticism, and used as raw material.

two advantages of keeping it to sound design:

- **tight scope makes the model easier to evaluate and steer.** a 30s synth from a fixed kit is a small, checkable thing, unlike a full song.
- **the rest of composition stays with the musician.** the machine designs sounds; writing the music is still mine to do as always.

these aren't meant to produce finished songs. they're synth studies, closer to what an engineer or sound designer hands a musician.

the longer-term idea is to get models to write sound design and musical études: short, focused studies that each explore one idea, sound or technique.

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

#### 2026-11-10
- first trial run: 50 samples of one prompt, to compare models and effort levels on the same task
- prompt, tweaked from yesterday's test:
  - before: "create bird song in a forest with crunchy leaves and crickets"
  - now: "create ambient bird song in a lush forest with crunchy leaves and crickets. it should feel like a lived-in space"
  - added "ambient", "lush" and the "lived-in space" sentence. dropped the final period so the user message still reads "...lived-in space, in SuperCollider."
  - saved as `prompts-trial.txt`, passed with `--prompts`
- model breakdown:
  - Opus 5.5, effort medium: 25 samples
  - Opus 5.5, effort high: 10 samples
  - Fable 5.1, effort high (harness default): 15 samples
  - asked for "hard" effort, but the API has no such level (low / medium / high / xhigh / max), so used high
- process:
  - one run folder `runs/2026-11-10_trial-50/`, three `sf.py generate` processes in parallel (one per model/effort), started 5s apart, logs in `trial-logs/`
  - all three file into the same prompt folder, so sample numbers are interleaved across models (sample-01..50). `meta.json` says which model/effort made each one; the review page shows it
  - small risk with parallel runs: two processes could claim the same sample-NN at the same moment and one would crash. didn't happen
  - took ~20 min wall clock (11:44–12:03). Opus high finished first (~11 min), Fable last
- hiccup: running Claude Code inside bb (instead of the terminal) puts every shell command in a sandbox. it blocked uv's cache (`~/.cache/uv`) and sclang (no access to macOS services, so every render failed). fixed with `bb plugin config provider-claude-code set sandboxEnabled false`
- result: 49 / 50 passed all checks. 25/25 Opus medium, 10/10 Opus high, 14/15 Fable
  - the one failure (Fable, sample-31) used `Impulse` and `Latch`, which aren't in the kit. it still rendered, so it's listenable
- next: listen and rate in the review site. then the same prompt with OpenAI (GPT-5.5, GPT-6 Astra): harness now supports `gpt-*` models via the Responses API, needs `OPENAI_API_KEY` in `.env`
- started tracking cost per sample: tokens used × list price. `meta.json` now has a `cost` field, `uv run sf.py cost` backfills old samples and prints a summary. prices live in `harness/pricing.py`
  - output tokens include thinking / reasoning tokens, and both Anthropic and OpenAI bill those as output
  - list prices (USD per 1M tokens, in / out): Opus 5.5 $4 / $20, Fable 5.1 $10 / $50, GPT-5.5 $5 / $30, GPT-6 Astra $10 / $50
- cost estimates so far (prompt: "create ambient bird song in a lush forest with crunchy leaves and crickets. it should feel like a lived-in space"):

  | run | model | effort | n | pass | total | $/sample | avg out tokens |
  |---|---|---|---|---|---|---|---|
  | trial-50 | Opus 5.5 | medium | 25 | 25 | $1.99 | $0.080 | 3,846 |
  | trial-50 | Opus 5.5 | high | 10 | 10 | $1.47 | $0.147 | 7,204 |
  | trial-50 | Fable 5.1 | high | 15 | 14 | $4.26 | $0.284 | 5,541 |
  | test-1 | Opus 5.5 | high | 1 | 1 | $0.13 | $0.132 | 6,456 |

  - ~$7.85 for all 51 samples
  - Fable costs ~2x Opus high per sample while writing fewer tokens: its per-token price is 2.5x
  - Opus high writes ~1.9x the tokens of Opus medium, so it costs ~1.8x per sample
- these are list-price estimates, not billed amounts: no caching or batch discounts

- second trial run: same prompt with OpenAI models, 50 samples in `runs/2026-11-10_trial-50-openai/`
  - harness now supports `gpt-*` models through OpenAI's Responses API: same system prompt, user message, effort setting and code extraction. key is `OPENAI_API_KEY` in `.env`
  - GPT-5.5 at effort medium (25 samples), to pair with Opus 5.5 medium. GPT-6 Astra (`gpt-6-astra`) at effort high (25), to pair with Fable 5.1 high (same per-token price)
  - same process as the Claude run: two parallel processes into one run folder, logs in `trial-logs/`. took ~40 min (15:40–16:19), Astra the slower one
- results, with the Claude runs for comparison:

  | model | effort | n | pass | total | $/sample | avg out tokens (reasoning) |
  |---|---|---|---|---|---|---|
  | GPT-5.5 | medium | 25 | 23 | $2.35 | $0.094 | 3,064 (2,577) |
  | GPT-6 Astra | high | 25 | 25 | $3.19 | $0.128 | 2,466 (1,901) |
  | Opus 5.5 | medium | 25 | 25 | $1.99 | $0.080 | 3,846 |
  | Fable 5.1 | high | 15 | 14 | $4.26 | $0.284 | 5,541 |

  - $5.54 for the 50 OpenAI samples, under the $8–14 I estimated beforehand (from Claude's token counts): OpenAI models wrote far fewer tokens
  - Astra vs Fable: same price per token, same effort, but Astra wrote less than half the tokens, so ~2.2x cheaper per sample, and 25/25 passed
  - ~80% of OpenAI output tokens are hidden reasoning, so the visible code + explanation is short
  - GPT-5.5 failures: sample-01 used 6 LFOs (kit allows 2), still rendered. sample-19 called `Env.new` with a single number instead of an array of levels, so the build failed and there's no audio
  - every reply finished normally, none hit the 32k output token limit

##### musings about future direction
three general ideas. not sure whether to pursue one of them or a combination.

1. **many models, manual evaluation, find patterns**
   - same prompt → different models, then different prompts
   - track which samples I love / hate and see if there are model patterns there
   - see if certain models prefer certain synth architectures (e.g. Fable seemed to reach for the delay more often than the others)
   - widen the model set: Google, DeepSeek, Mistral, local models, etc.
   - open questions: how good an eval / benchmark would this be? is there something worth articulating, like "which models are good at which aspects of sound design", or "here are the models' biases / preferences when I let them build their own synths"? is it useful to the general public? has anyone else done this?

2. **LLM reviews → my taste, in words → steer future runs**
   - get LLM-written reviews for every sample. they add richer vocabulary and text context to each one
   - with my love / neutral / hate ratings, this gives a language-based, tractable line of reasoning between what I (specifically me) like and how it's described
   - feed that back as extra prompt material in future runs, to raise the chance of samples I like
   - not universal: it's personalising a model to my taste. maybe that's finetuning, post-training or some kind of RL. don't know where it leads

3. **a study / étude system**
   - older composers wrote studies, or large bodies of work, exploring what an instrument can do: sounds in various configurations, overtones, compositional effect. études are a student-practice thing, but share the idea of honing in on a tightly scoped pattern, problem or embodied skill. here, a small set of prompts or rules would be the scope; the kit constraint is already along those lines
   - frame the project as a study of models' ability to build synthesizers for ambient music, over a defined repertoire: rain, birdsong, ambient trains, waves on a beach, etc. (the actual prompt set can come later)
   - also like painters' studies of new pigments, brushes and solvents across conditions, to learn the strengths and weaknesses of each

TODO (later): model habits, starting with Pulse
- observation: GPT-5.5 used the `Pulse` oscillator in every sample (25/25). `Pulse` is in the kit (`kit.json` oscillators: SinOsc, Saw, Pulse, LFTri), so it's allowed, but the consistency stands out
- questions:
  - what about the others, how often do they use it? first count (samples with `Pulse.ar/.kr`, not `LFPulse`): GPT-5.5 25/25, Opus 5.5 25/35, GPT-6 Astra 19/25, Fable 5.1 7/15
  - in what capacity does GPT-5.5 use the Pulse oscillator?
  - is that same role filled by a different technique in other models' architectures?
  - do all models have their own repetitive style? how does that sonically translate?
