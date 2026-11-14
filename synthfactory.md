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

##### architecture patterns: first pass (not on the review site yet, so listening stays about the sound alone)
from the compiled UGen lists (`checks.json`) and the code (`synth.scd`) of the 100 trial samples. one prompt only, so these are habits on this prompt, not general traits. Opus medium + high combined (35), Fable 15, GPT-5.5 25 (24 built), Astra 25.

1. **everyone fills the whole kit, and the skeleton is the same.** 98 / 99 built samples use every part at its max (2 osc, 2 filters, noise, env, 2 LFOs, delay). every sample has a sine-oscillator bird (`SinOsc.ar`) and `.tanh` soft-clipping at the end. so the delay isn't a Fable habit after all: every model uses one in nearly every sample. models differ in which parts they pick, not how many
2. **crickets are made two different ways, and that's where Pulse comes in.** all four models use `Pulse` for the crickets, but in different roles:
   - OpenAI (GPT-5.5 25/25, Astra 19/25): Pulse *is* the cricket. a narrow square wave at ~4–5.7 kHz, e.g. `Pulse.ar(4600 + (drift * 220), 0.18)`. likely sound: a pitched, buzzy tone
   - Claude (Opus 25/35, Fable 7/15): Pulse at ~30 Hz is a *trill gate* chopping a band of filtered noise, e.g. `trill = Pulse.ar(31, 0.35)` on `BPF.ar(noise, 4700, ...)`. likely sound: a breathy, shimmering chirp
   - so the same kit part plays a different role, and the "missing" Pulse in other samples is mostly the same gate done with `LFPulse`
3. **noise colour splits by company.** Opus: white noise 34/35. OpenAI: pink noise (GPT-5.5 20/24 plus 3 brown, Astra 24/25). Fable in between: 8 white / 7 pink. pink / brown is darker and softer; white is brighter, more hiss for leaves and crickets
4. **Opus has a template.** almost every Opus sample has the same five parts: `BPF` (35/35), white noise (34/35), `CombL` delay (35/35), sine bird, and an `LFSaw` LFO (34/35) that sweeps the bird into a repeating downward chirp. `LFSaw` is nearly Opus-only (Astra 9/25, GPT-5.5 and Fable 0). expect Opus birds to sound alike from sample to sample
5. **delay flavour and code style.**
   - GPT-5.5 is the only one that prefers the allpass delay (`AllpassL` 15/24, smears / diffuses); everyone else uses the comb delay (`CombL`, audible echo / ringing): Opus 35/35, Fable 15/15, Astra 24/25
   - GPT-5.5 writes no comments at all (avg 0 per sample); Opus and Fable ~9–11; Astra ~4. Astra writes the longest code (~37 lines vs 20–24 for the rest)

caveats: counts say which parts are *present*, not their role. only Pulse was checked by reading the code. "likely sound" lines are guesses from the code, to be checked by ear after rating.

#### 2026-11-12
##### new idea: personas (2am thoughts)
- prompted by [this tweet](https://x.com/jkeatn/status/2119874007382299087): someone painting in code with Fable 5.2 had Claude invent *personas*, artists with a back story and a consistent style roughly placed in time. this was a response to the critique that Opus 5.5's paintings only emulated real art. over weeks they gave comments and steers, Claude revised the brush / paint engines, and went deeper on paintings they liked. each painting is a 1–2k line Python program placing every stroke through a simulated paint engine. no image model, no rendering or tracing in advance
- second persona tweet to read alongside the first: [@aliceisplaying](https://x.com/aliceisplaying/status/2120519664711385185)
- same critique applies here: "bird song in a forest" asks for a sound effect, and gets a sign that says "bird". a persona changes the question from "what does a forest sound like?" to "what does *this composer* make of a forest?"
- a persona is like an expansion of the kit: the kit limits the materials (the pigments and brushes), the persona limits the choices (the hand). in the tweet the engines grew per persona, so each persona ended up with its own kit
- but a persona can still work with a fixed kit. real musicians work with limited gear all the time, and that's often where a voice shows most
- ways to try it, smallest first:
  1. persona on the shared kit: does a voice change the sound when the materials are the same? same harness and checks, only the prompt changes
  2. persona with its own kit variation (e.g. no noise but 3 LFOs)
  3. a persona that changes its own kit over time with my steers ("studio" mode, like the tweet). not comparable across models, and that's fine
- two ways of working, keep them separate: *lab* = one-shot, controlled, comparable samples (what I have now). *studio* = one agent, iterating with me over weeks
- open bits: the agent can't hear its own work, so my ears are the feedback loop (spectrogram / measurements as a rough self-check). a persona might want longer forms than 30s. fits the étude idea: a persona's body of studies
- Fable 5.2 isn't in the harness yet: check the model ID before using it

##### blog post ideas
- central question candidate: **when a model makes sound under constraints, is the result an aesthetic, or just engineering? and does the difference matter if it sounds good?**
- framing: models may not be artists, but they're probably good sound engineers. like a studio with a "house sound": the kit is the gear, the model's habits are the house band + engineer, the prompt is the artist walking in
- the "sound" of studios often came from practical engineering decisions, not taste:
  - **Motown:** mixes checked on small car-radio-like speakers, a cramped basement room, an improvised echo chamber, few tracks to record on, tambourine and claps loud enough to cut through AM radio. practical fixes that together became "the Motown sound"
  - **Ilaiyaraaja:** often wrote for tea stalls playing AM radio through narrow-band mono loudspeakers. that directly shaped his composition and arrangement
- my harness has the same kind of constraints: mono, exactly 30s, peaks below 1.0, a fixed kit, no reverb. my AM radio and track count
- fits my listening notes already: remarks like "good texture", "too synthetic but still sounds good" are exactly the aesthetic-vs-engineering line
- other "house sound" parallels to maybe use: Stax vs Motown (same era and genre, different studios), Sun Studio's slapback, Rudy Van Gelder / Blue Note, King Tubby (the desk as instrument), BBC Radiophonic Workshop (sound design to a brief with limited tools), Steve Albini (engineer, not producer: tried *not* to impose a sound)
- possible test while listening: can I guess which model made a sample before looking?
- sets up the persona post: if the default sound comes from constraints and habits, a persona is the first deliberate attempt at taste
- structure (rough): setup and process → what I heard + ratings → architecture patterns (after listening) → my taste vs token / cost → what this says about models and about me
- follow-up question: **if we create a meaningful enough constraint on what the model is allowed to use, will that lead to creative decisions?** (H1)
  - *meaningful* = it removes the obvious solution. arbitrary limits ("only 3 parts") might just give smaller versions of the default answer. e.g. "crunchy leaves" with no noise source: the crunch has to be invented from oscillators + modulation
  - parallels: Oulipo (Perec's novel without the letter "e"), Eno's Oblique Strategies, Dogme 95. block the default and something new has to happen
- alternative hypothesis: **taste isn't stored in a single tool but in their composability** (H2)
  - musicians support this: the same Stratocaster or 808 sounds completely different depending on who uses it. the tool is shared, the taste is in how it's used and combined
- they may fit together: take tools away and the only place left for choices is the wiring (what modulates what, what feeds what). H1 could be the mechanism, H2 the explanation
- title / line candidate: **"deaf machines making sounds"**
  - the models can't hear what they make. a deaf engineer can only make engineering decisions: what usually works, what the code is meant to do
  - so any aesthetic that comes out is either inherited (absorbed from everything written *about* sound), a function of the tools (the kit and constraints), or accidental
  - the model acts on its intentions, not on results. seen on day one: the code comments describe what the model meant, not what came out
- post arc: question (aesthetic or engineering?) → hypothesis (where does taste live?) → experiment

TODO (next)
- **first: finish listening to and rating the current 100 samples (trial-50 + trial-50-openai). everything below waits on this**
- then: look at the architecture patterns (2026-11-10 entry) next to my ratings and taste notes, and per-model love rate vs token use / cost
- test H2 (composability): do the samples I love share *parts* or share *wiring*? data already exists in each sample's `graph.json`
- test H1 (meaningful constraint): small run on the same prompt with one constraint that removes the obvious solution (e.g. no noise source), compared with the normal kit. listen: more surprising, or just broken?
- later: Pulse habits TODO (2026-11-10), persona mini-project (2026-11-12)
- reading: literature review below, in priority order. read #1 only after rating (it has per-model results). not all of it needs reading
- search the music-tech conferences properly (NIME, ICMC, DAFx, ISMIR) for anything closer

##### literature review
in priority order: closest to my question and method first. found with three web searches, not a full review.

1. **[The Goldborg Variations: algorave attractor states of LLMs](https://www.lesswrong.com/posts/DWtMPR8797aEDYXei/the-goldborg-variations-algorave-attractor-states-of-llms)** (LessWrong)
   - several frontier models get the same Strudel seed (a Goldberg Variations bass line) and repeatedly "evolve" it. each drifts toward its own default style, its "attractor state". an LLM judge classifies the final pieces
   - overlap: almost exactly my "do models have a sound?" question, in live coding instead of synthesis. differs: no fixed kit, a judge model instead of a human ear, evolving one piece instead of one-shot samples
   - ⚠ has per-model results: read after rating
2. **[Benchmarking LLM Code Generation for Audio Programming with Visual Dataflow Languages](https://arxiv.org/pdf/2409.00856)** (2024)
   - 10 tasks (5 specific techniques: FM, AM, LFO, filtered noise…; 5 open-ended scenes: church bell, babbling brook…), 600 samples, judged by ear plus code correctness / complexity
   - overlap: closest *method*: models writing audio code, judged by listening. their technique-vs-scene split could shape my étude repertoire. differs: visual languages, not text code; no fixed kit; no taste or cost angle
3. **[AutoSynth: Learning to Generate Editable Synthesizer Programs from Audio and Text](https://arxiv.org/html/2610.10774v1)** (2026)
   - predicts editable synth programs (notes, synth parameters, modulation routes) from audio or text. supervised learning + GRPO reinforcement learning
   - overlap: the RL / post-training direction in my notes (Surya-style loop), and "editable program" = my synth-as-code. differs: trains its own model; aims at matching a target, not taste
4. **[A Text-Steerable Instrument for Sketching Procedural Soundscapes via Language Models](https://arxiv.org/pdf/2607.00309)** (2026)
   - an LLM generates layered soundscape settings ahead of time; they're retrieved at runtime from text
   - overlap: ambient / procedural soundscapes, like my repertoire idea (rain, birdsong, trains, waves). differs: LLM fills in settings for a fixed engine, doesn't write the synth
5. **[CTAG: Creative Text-to-Audio Generation via Synthesizer Programming](https://www-prod.media.mit.edu/publications/ctag-neurips/)** (MIT Media Lab)
   - repeatedly adjusts a 78-parameter modular synth until an audio-language model judges it a good match for a text prompt
   - overlap: text → synth, and an automatic audio-language judge (like the CLAP idea in my notes). differs: optimisation turning knobs, not a model writing code; scored by a judge, not a person
6. **[TidalCycles autonomous agent paper](https://core.ac.uk/download/477975526.pdf)** + **[Live Coder in the Loop](https://zenodo.org/records/17306276)** (2025 performance proposal)
   - an agent that makes TidalCycles patterns and develops a sense of value for them; a human live coder accepts or rejects its suggestions
   - overlap: the *studio* / persona mode: an agent with its own sense of value, steered by a human. differs: live performance, patterns not synth design
7. **[SonicRAG: high-fidelity sound effects synthesis based on retrieval-augmented generation](https://arxiv.org/pdf/2505.03244)**
   - notes LLMs often write complex audio scripts that fail to run, and proposes a higher-level language for sound effects
   - overlap: the reliability side: my compile / build checks and failure rates. differs: sound effects, not études; adds its own language layer
8. **Synth sound matching** (infer synth settings from audio): [Audio Spectrogram Transformers, Native Instruments](https://arxiv.org/pdf/2407.16643) ([DAFx](https://www.dafx.de/paper-archive/2024/papers/DAFx24_paper_95.pdf)), [flow matching](https://arxiv.org/pdf/2506.07199), [FM synth audio-parameter embeddings](https://arxiv.org/pdf/2608.18226)
   - overlap: links synth settings to sound, the reverse of my direction; possibly useful later for predicting sound from code (the JEPA-ish idea). low priority
9. **Tools** (not studies): [claude-collider](https://github.com/jeremyruppel/claude-collider), [SuperColliderMCP](https://github.com/Tok/SuperColliderMCP), [strudel-tui](https://awesome.ecosyste.ms/projects/github.com%2Fthedavidweng%2Fstrudel-tui)
   - overlap: an LLM driving SuperCollider / Strudel live. possible infrastructure for persona / studio mode, not evidence for my question
10. **Informal:** ["Can Claude compose?" blog](https://blog-5e2.pages.dev/ll-ms-as-music-synthesizers-can-claude-compose), [notes on generating synth patches with AI (gist)](https://gist.github.com/0xdevalias/5a06349b376d01b2a76ad27a86b08c1b)
    - overlap: anecdotes, one person's impressions. background only

where my project stands out (as far as these searches show): models write the synth as code; a fixed kit across models; many frontier models on the same brief, with cost; one musician's taste as the measure, framed as "aesthetic or engineering?". the closest work does one or two of these, not all
