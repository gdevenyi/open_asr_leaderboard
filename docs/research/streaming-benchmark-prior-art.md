# How existing streaming ASR benchmarks define latency

Research for issue #6 of the streaming track map (#3). Sources were read on 2026-11-01. Bracketed numbers refer to [Sources](#sources).

## Short answer

There is no standard definition. Sources split into four families, and the end-of-speech reference point is what separates them.

| Family | Clock runs from | Clock stops at | Who uses it |
|---|---|---|---|
| A. Aligned speech | Forced-aligned or manually timestamped speech | Token, word, or endpoint event | Research papers, Picovoice |
| B. VAD end of speech | Silero VAD end of speech, live or precomputed | Final transcript | Pipecat, Artificial Analysis, Coval; Deepgram's EOT tool is close |
| C. Last audio sent | Last audio packet of the utterance | Final transcript | Gladia, ElevenLabs |
| D. No end of speech | Nothing: compares the audio sent with the audio transcribed | Each interim result | Deepgram transcript latency |

Family B is the closest thing to a current convention for voice agents: real-time pacing and wall-clock receipt at the client. Its members still disagree on VAD parameters, on what counts as "final" when a provider has no finalize signal, on aggregates, and on network handling.

A few sources fit no family: Simul-Whisper's DAL is measured against an ideal emission rate [7], and on-device Nemotron derives latency from chunk size and RTFx [8]. AssemblyAI's TTCT looks like family B, but it doesn't define its end-of-turn reference [20].

No public streaming benchmark I found publishes a stream capacity figure with a stated pass criterion.

## How to read the citations

- **Read directly:** arXiv PDFs converted with `pdftotext`, and repository files fetched raw at a pinned commit. Quotes from these are exact.
- **Checked against raw page:** Artificial Analysis, Deepgram, and AssemblyAI. The quotes used here were matched against the page source.
- **Fetch tool only:** marked **(F)** in the sources. These pages were read through a summarising fetch tool and not checked against the raw page, so treat their quotes as probably accurate.
- **Vendor-run** means the publisher sells an STT model, STT hosting, or a product in the comparison. Vendors are flagged in Table 1.
- Anything I concluded rather than read is labelled **Inference**.

## Table 1: Definitions

| Source | Run by | Metric and exact definition | End-of-speech reference | Word timing ground truth |
|---|---|---|---|---|
| FastEmit [1] | Google (own models) | **PR latency:** "when the last token is emitted in the finalized recognition result" minus end of speech. **EP latency:** predicted end of query minus end of speech | "estimated by forced alignment" | Forced alignment; aligner not named in the passage read |
| Better and Faster [2] | Google (own models) | **EP latency:** user finishes speaking to predicted end of query. **Prefetch latency:** first correct prefetch to user finishing | User finishes speaking; source of the timing not given in the passage read | Not stated in the passage read |
| JOIST [3] | Google (own models) | **Partial latency:** "when the first correct partial hypothesis is generated" relative to the user finishing. **EP latency:** as above | User finishes speaking | Not stated in the passage read |
| Dissecting UPL [4] | Facebook AI (own models) | **First token emission delay:** speech start to first non-blank token. **UPL:** user finishes speaking to the decision to close the microphone, made of endpointer lag and decoder catchup lag | "when the user finishes speaking" | Notes that earlier work "typically" uses forced alignments |
| Fast-slow cascaded [5] | Meta AI (own models) | **Emission delay "(or finalization delay)":** audio duration from finishing a token to its surfacing "in the 1-best partial hypothesis" | Per token, not per utterance | Not stated in the passage read |
| Whisper-Streaming [6] | Academic (own system) | Time between the ASR emitting a word and "the corresponding gold word" being spoken, "excluding words deleted by the ASR" | Per word | ESIC corpus manual word timestamps, aligned with edit distance (`edlib`) |
| Simul-Whisper [7] | Academic, with a Sinovoice co-author (own system) | **DAL:** token lag against an ideal policy that emits every Na/Nt seconds, with no spoken-word reference | None: ideal emission rate | None |
| On-device Nemotron [8] | Microsoft | **Algorithmic delay** from chunk and right context; **effective latency** ≈ delay + chunk duration / RTFx | None | None |
| Picovoice repo [9][10] | **Vendor** (Cheetah is in the table) | **Word emission latency:** "delay from the point a word has finished being spoken to when its transcription is emitted" | Per word end | Montreal Forced Aligner, `english_us_arpa` [11] |
| Pipecat stt-benchmark [12][13][14] | Framework maintainers (Daily); not an STT vendor (see note) | **TTFS:** `final TranscriptionFrame receipt − (VADUserStoppedSpeakingFrame timestamp − stop_secs)` | Live Silero VAD, `stop_secs` 0.2 | None. Reference text is Gemini-generated and human-reviewed |
| Artificial Analysis [15][16] | Third party; not an STT vendor (see note) | **Time to Final:** end of speech to next final, with a fallback ladder. **Time to First Partial:** same start, stopping on the first partial | Silero VAD; parameters not stated | None |
| Coval [17][18][19] | Third party (states it sells no TTS or STT models [35]) | **TTFS:** `audio_to_final − speech_end_offset`, clamped at 0. **TTFT:** "first partial" | Precomputed offline Silero offset per clip; script defaults 300 ms min silence, 100 ms pad | None. `stt-v3` references are model-generated |
| AssemblyAI docs [20] | **Vendor** | **Emission latency:** "when a word is spoken to when that word is returned." **TTCT:** "end of speaker turn to receipt of the finalized transcript" | Not defined | Not stated; "internal evaluation set" |
| Deepgram docs [21] | **Vendor** | **Transcript latency:** audio seconds sent minus audio processed, X − Y, interims only. **EOT latency:** user stops speaking to EOT event | X − Y has none. EOT uses VAD via a Deepgram tool | None |
| Gladia blog [22] (F) | **Vendor** | **TTLB:** "final segment received" minus "last audio packet sent." Also a TTFS glossary entry | Last audio packet sent | Not stated |
| ElevenLabs blog [23] (F) | **Vendor** | **Time to first partial:** first chunk sent to first non-empty partial. **Partial-to-final lag:** last chunk of utterance to final | Last chunk of utterance | Not stated |
| Soniox wiki [24] (F) | **Vendor** | **First-token latency**, **finalization latency** (word spoken to word final), **endpointing delay**, and network as separate terms | Per word | Not stated |
| Picovoice blog [25] (F) | **Vendor** | **Word emission latency** and **text finalization latency** ("time to get the stable (i.e., final) version of the word") | Per word | Points to the repo |

Notes on "Run by":

- Daily publishes Pipecat. The fetched blog lists Deepgram, a benchmarked service, as a partner [14]. **Inference:** this is a smaller conflict than selling STT, but it isn't arm's-length.
- Artificial Analysis uses a proprietary dataset, AA-AgentTalk, weighted 50% [15], so it can't be fully reproduced.
- Gradium, an STT vendor, republishes Coval numbers and gives two incompatible definitions of TTFT on one page: from audio arrival, and from the user finishing [26] (F). Labels drift even when the underlying harness is open.

## Table 2: Mechanics

| Source | Pacing and clock | Concurrency | Network | Aggregates and runs |
|---|---|---|---|---|
| FastEmit [1] | Not stated. Claims PR latency doesn't depend on hardware or system optimisation. **Inference:** audio clock | None | Not applicable | P50, P90 |
| Better and Faster [2], JOIST [3] | Not stated | None | Not applicable | P50, P90 |
| Dissecting UPL [4] | Actual runtime on Android, single core, "under controlled conditions" | None | On-device | P50 shown in tables |
| Fast-slow cascaded [5] | Defined in audio duration | None | Not applicable | Mean, P99 |
| Whisper-Streaming [6] | Simulated real-time feed with real compute on a shared A40 cluster | Not measured; one GPU blocked per simulation | Not applicable | Mean within document, mean ± SD across documents; 10 repeat runs of one setup |
| Simul-Whisper [7] | Reports both computation-unaware and computation-aware latency | None | Not applicable | Mean over tokens |
| On-device Nemotron [8] | Derived from chunk size and measured RTFx at batch size 1 | None | On-device | RTFx utterance-level average |
| Picovoice repo [9][10] | **Cloud:** real wall clock, `apply_delay=True`. **Cheetah and Nemotron:** audio clock, so compute is ignored | A process pool per CPU; not measured | Cloud rows include it, unlabelled | Mean only; 100 LibriSpeech test-clean files |
| Pipecat [12][13][27] | Real-time, 20 ms chunks; silence after the audio until the service goes quiet | Sequential, 0.1 s between samples | Included, single US location, business hours [14] | Median, P95, P99; 1,000 samples |
| Artificial Analysis [15] | Real-time, "20ms chunks, or as recommended by the public model config" | Not mentioned on the page | "included in all latency measurements" | Sample-weighted within a dataset; run count not read |
| Coval [17][18][19] | 1x, paced to an absolute deadline; end-of-audio frame sent on schedule | STT not stated. "Concurrency of 8" appears only under the HTTP TTS cohort | Handshake excluded before t0; runner in `us-east-1`, so transit is included | Latency aggregate not established by what I read; runs about every 30 minutes on a random sample |
| AssemblyAI [20] | Not stated | Not stated | Not stated | P50, P90 |
| Deepgram [21] | Advises 20 to 100 ms buffers | Not stated | Included in X − Y; advises estimating transit from TCP connect time | Advises P50, P95, P99 |
| Gladia [22] (F) | Sample code uses about 128 ms chunks | Claims "concurrent load"; no method given | Claims "multi-region network paths"; no method given | Advises P50, P90, P99, at least 100 utterances |
| ElevenLabs [23] (F) | Advises real-time, for example 100 ms chunks | Not specified | Advises stating harness location | Advises P50, P95, about 10 runs per file |

## Findings by question

### Emission or partial latency

Sources measure at least four different things under this name:

1. Per-word lag behind aligned speech: Picovoice [9], Whisper-Streaming [6], AssemblyAI [20], Meta's emission delay [5].
2. Utterance-level lag against the end of speech: FastEmit [1], JOIST [3]. These can be negative, because a model can emit before the audio is spoken [1].
3. Lag of processed audio behind sent audio: Deepgram [21].
4. Time to the first partial after end of speech: Artificial Analysis [15].

Coval's TTFT is described as "first partial" [18], but I did not read the code that sets its clock origin. AssemblyAI rejects time to first token because "some providers emit tokens before any audio is actually spoken" [20] (vendor claim). Coval withholds TTFT for providers that emit on a fixed interval, since the number then measures the wait for the next tick [17].

### Finalization latency

The map's definition, end of speech to final text, matches family B and AssemblyAI's TTCT [20]. The same words mean something else elsewhere: Meta's "finalization delay" is per token and stops at the 1-best *partial* [5], and Soniox and Picovoice's blog use per-word definitions [24][25].

What counts as "final" differs even within family B:

- **Pipecat:** an explicitly finalized frame if the service confirms one, otherwise the last transcript received within a 2.0 s timeout after VAD stop [12][27].
- **Artificial Analysis, forced endpoint supported:** the timer starts when "the audio through the speech-end point has actually been streamed and the force-endpoint signal sent," and stops on the next final. If a final arrived before VAD fired, the most recent one is used [15].
- **Artificial Analysis, not supported:** first natural final within 2 s, else first partial after 2 s, else the latest partial before 2 s [15].
- **Coval:** `audio_to_final` is overwritten on each final. An end-of-audio frame is sent, then the harness waits up to 5 s [19].

**Inference:** the length of these 2 s and 5 s windows can change which event gets timed, so results from different harnesses may not be timing the same event.

### End-of-speech reference

- Research papers use forced alignment [1] or manual corpus timestamps [6].
- Current voice-agent benchmarks use Silero VAD, because their datasets have no word timings. Deepgram says EOT latency is "difficult to measure precisely without ground truth timestamps" [21].
- VAD parameters move the anchor. Pipecat subtracts `stop_secs` to recover the speech end [12]. Coval's anchor includes a speech pad by construction [19]. Artificial Analysis doesn't state its parameters [15].
- Artificial Analysis deliberately starts from the wall-clock send moment rather than the audio-file timestamp, to avoid "pacing slip" [15]. Coval takes the opposite route, subtracting an audio-file offset and pacing to absolute deadlines to keep the two aligned [18][19].

### Word timing ground truth and alignment

The only two public evaluations I found that compare emitted words with reference word timings and describe the method are Picovoice and Whisper-Streaming. Both drop words that do not align: Picovoice counts only exact word matches and skips files with more than 20 errors [9], and Whisper-Streaming excludes deleted words [6]. **Inference:** both bias latency toward words the model got right.

Picovoice's cloud engines stamp a word when it first appears in a partial, and earlier words keep their timestamp when later partials arrive [10]. That measures first appearance and says nothing about when the word became stable. Its Nemotron engine differs: a revised word gets a new, later stamp [10].

### Real versus simulated pacing

- Family B paces in real time and measures at the client [12][15][18].
- Simul-Whisper reports both clocks [7].
- Picovoice mixes the two in one results table. Cloud engines get wall-clock timing with real sleeps, while Cheetah and Nemotron stamp a word with the audio position at which it appeared, so compute time is ignored [10]. The README's latency section doesn't mention this [9]. For Vosk, Moonshine, and whisper.cpp I only confirmed that the reference times use the same audio-time form [10].

### Concurrency

No streaming source in this survey defines capacity with a stated pass criterion.

- **NVIDIA (vendor), Nemotron 3.5 model card:** throughput is "the number of real-time streams sustainable in parallel," and latency is "median final-token latency at a given level of concurrency," but no threshold or harness is described. It reports 240 versus 14 streams at 80 ms on one H100 [28] (F).
- **NVIDIA (vendor), blog:** 560 streams at 320 ms on an H100 against a 180-stream baseline, plus a Modal test with 127 WebSocket clients. "Peak capacity" is undefined [29] (F).
- **Baseten (vendor):** finalization latency of 98 ms at 8 streams and 138 ms at 100, paced at real time, with "zero errors" as the only stated condition. Percentile and client location aren't given [30] (F).
- **E2E Networks (vendor):** says load under concurrent streams "was not tested." Its 13-instance figure appears to come from VRAM headroom [31] (F).
- **MLPerf Inference:** the Server scenario reports the maximum Poisson throughput under a 99th-percentile latency bound, found by binary search [32]. That's the most rigorous capacity definition here, but Whisper is Offline-only with no server bound, and queries are 30 s clips rather than streams [32][33].

### Network handling for APIs

- **Included and stated:** Artificial Analysis [15].
- **Included, location disclosed:** Pipecat [14], Coval [18].
- **Split out:** Coval excludes the handshake [17]. Deepgram separates connection latency from per-message latency, and says transcription latency "cannot be measured directly from the client" [21].
- **Included, unlabelled:** Picovoice's cloud rows [10].
- **Not addressed:** AssemblyAI [20].

No source that addresses network handling says it subtracts transit from published figures. The Coval README names the problem directly: "providers serving from other regions carry that round-trip" [18].

### Aggregates

Reporting is inconsistent: P50/P90 [1][2][3][20], mean/P99 [5], median/P95/P99 [12], mean only [9], and sample-weighted within a dataset [15]. Pipecat also publishes a transcript coverage column alongside latency [12].

## How the Open ASR Leaderboard frames speed

- RTFx is defined as total audio duration over transcription time, and the paper states "Higher values indicate faster inference (i.e., lower latency)" [34]. **Inference:** this equates throughput with latency, which is true for batch work and doesn't carry over to streaming.
- It was measured on an A100 at batch size 64, reduced adaptively. The paper says absolute values don't transfer across hardware, but relative comparison is meaningful [34].
- The paper defines no emission or finalization metric, and "streaming" appears only in a cited title [34].
- API rows show "–" for RTFx in the paper's multilingual table [34]. In this repo, `api/run_eval.py` still computes a wall-clock RTFx, and `scripts/open_results_pr.py` writes it as blank or `-1` for API models depending on the sheet.

## What this suggests for later tickets

All of this section is **inference**, not something a source says.

- **#9 (finalization latency endpoints):** the map's term is safe, but the spec should define it explicitly, given the collision with [5]. The open choices are VAD or alignment for the anchor, and what ends the clock for models without a finalize signal. No source validates VAD anchors against forced alignment; a pilot could measure that gap directly.
- **#8 (real or simulated clock):** open-weight runs on H200 could use either, but Picovoice shows the cost of mixing clocks in one table. If a simulated clock is used anywhere, it should be labelled computation-unaware, as in [7].
- **#12 (API latency reporting):** a stated client location and a label on the column would match the best practice here [15][18]. Excluding the handshake [17] is reasonable. Nothing surveyed makes API and local latency comparable, which supports the map's decision.
- **Stream capacity:** there's no definition to adopt. The MLPerf pattern, the highest load that meets a tail-latency bound found by search [32], is the closest template, but it would need adapting to continuous streams.

## What I could not verify

- **No hands-on testing.** All harness behaviour comes from reading code and documents. Nothing was run, and no API figure was reproduced.
- **Pipecat core version.** The TTFS logic in [27] was read at `main`. I didn't check which Pipecat version the benchmark commit locks.
- **Coval TTFT clock origin.** I read `final()` and the pacing code but not `set_first_token`.
- **Coval's STT concurrency, chunk size, and latency aggregate.** None is established by what I read.
- **Fetch-tool sources.** Those marked (F) were not checked against the raw page.
- **Artificial Analysis concurrency.** The page's HTML has no occurrence of "concurrent", but the running harness isn't public as far as I could tell.
- **Modal's own write-up.** I didn't find it, so [29] is the only account of that test.
- **Sources not opened, so not used:** SimulStreaming and whisper_streaming repositories, LAAL papers, and the reference [9] cited inside [5] that defines emission delay. The `simulstream` latency scorer page was opened, but it gives no definition.
- **Coverage.** This is not an exhaustive survey, and I found no Speechmatics methodology document in the searches I ran.

## Sources

**Papers (read directly)**

- [1] Yu et al., FastEmit, https://arxiv.org/abs/2010.11148, section 3.1
- [2] Li et al., A Better and Faster End-to-End Model for Streaming ASR, https://arxiv.org/abs/2011.10798, section 3
- [3] Sainath et al., JOIST, https://arxiv.org/abs/2210.07353, latency metrics in the experiments section
- [4] Dissecting User-Perceived Latency of On-Device E2E Speech Recognition, https://arxiv.org/abs/2104.02207, sections 1 and 2
- [5] Mahadeokar et al., Streaming parallel transducer beam search with fast-slow cascaded encoders, https://arxiv.org/abs/2203.15773, metrics section
- [6] Macháček et al., Turning Whisper into Real-Time Transcription System, https://arxiv.org/abs/2307.14743, evaluation section
- [7] Simul-Whisper, https://arxiv.org/abs/2406.10052, section 4.2
- [8] Banfic et al., Pushing the Limits of On-Device Streaming ASR, https://arxiv.org/abs/2604.14493, section 3

**Repositories (read directly, pinned)**

- [9] Picovoice README and `benchmark_latency.py`, https://github.com/Picovoice/speech-to-text-benchmark/tree/aea7914706d44d3163c1c0283e05700900abe7af
- [10] Picovoice `engine.py`, https://github.com/Picovoice/speech-to-text-benchmark/blob/aea7914706d44d3163c1c0283e05700900abe7af/engine.py
- [11] Picovoice `script/README.md`, https://github.com/Picovoice/speech-to-text-benchmark/blob/aea7914706d44d3163c1c0283e05700900abe7af/script/README.md
- [12] Pipecat stt-benchmark README, https://github.com/pipecat-ai/stt-benchmark/blob/94f567299a21f8d617039bc69f137a131f5dedd5/README.md
- [13] Pipecat stt-benchmark `docs/measuring-ttfs.md`, `pipeline/benchmark_runner.py`, and `pipeline/synthetic_transport.py`, https://github.com/pipecat-ai/stt-benchmark/tree/94f567299a21f8d617039bc69f137a131f5dedd5
- [17] Coval `docs/methodology.md`, https://github.com/coval-ai/benchmarks/blob/b33fd255eb105f515d18060145030ed03da82993/docs/methodology.md
- [18] Coval README and `providers/stt/_pacing.py`, https://github.com/coval-ai/benchmarks/tree/b33fd255eb105f515d18060145030ed03da82993
- [19] Coval `metrics/ttft.py`, `providers/stt/_streaming.py`, and `scripts/precompute_vad_offsets.py`, https://github.com/coval-ai/benchmarks/tree/b33fd255eb105f515d18060145030ed03da82993/runner
- [27] Pipecat core `services/stt_service.py`, https://github.com/pipecat-ai/pipecat/blob/6f777a0ef3bb98163d10d708525a0db21cda5fbf/src/pipecat/services/stt_service.py
- [32] MLPerf Inference rules, https://github.com/mlcommons/inference_policies/blob/d3eba2f21026d868ad65cdcad2bb81e4a17ce3d3/inference_rules.adoc

**Methodology pages (checked against raw page)**

- [15] Artificial Analysis, Speech to Text Benchmarking Methodology, https://artificialanalysis.ai/speech-to-text/methodology
- [20] AssemblyAI, Streaming benchmarks, https://www.assemblyai.com/docs/streaming/benchmarks.md
- [21] Deepgram, Measuring Streaming Latency, https://developers.deepgram.com/docs/measuring-streaming-latency

**Fetch tool only (F)**

- [14] Daily, Benchmarking STT for Voice Agents, https://www.daily.co/blog/benchmarking-stt-for-voice-agents/
- [16] Artificial Analysis, AA-WER Streaming, https://artificialanalysis.ai/articles/new-streaming-speech-to-text-benchmark-aa-wer-streaming
- [22] Gladia, Latency benchmarks for streaming speech-to-text, https://www.gladia.io/blog/latency-benchmarks-for-streaming-speech-to-text-ttlb-and-p99
- [23] ElevenLabs, Real-time Speech to Text latency guide, https://elevenlabs.io/blog/real-time-speech-to-text-under-200ms
- [24] Soniox, Speech-to-text latency, https://soniox.com/wiki/speech-to-text-latency
- [25] Picovoice, Speech-to-Text Latency, https://picovoice.ai/blog/speech-to-text-latency/
- [26] Gradium, STT API Benchmark 2026, https://gradium.ai/content/stt-api-benchmark-2026-latency-accuracy
- [28] NVIDIA, Nemotron 3.5 ASR streaming model card, https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b
- [29] NVIDIA, Nemotron Speech ASR scaling voice agents, https://huggingface.co/blog/nvidia/nemotron-speech-asr-scaling-voice-agents
- [30] Baseten, Introducing NVIDIA Nemotron 3.5 ASR Streaming, https://www.baseten.co/blog/introducing-nvidia-nemotron-35-asr-streaming/
- [31] E2E Networks, Benchmarking ASR models on NVIDIA L4, https://www.e2enetworks.com/blog/benchmarking-asr-models-nvidia-l4-parakeet-whisper-nemotron.md
- [33] MLCommons, speech2text reference implementation, https://github.com/mlcommons/inference/tree/master/speech2text
- [35] Coval, Voice AI TTS and STT benchmark, https://www.coval.ai/blog/voice-ai-tts-stt-benchmark/

**Open ASR Leaderboard**

- [34] Srivastav et al., Open ASR Leaderboard, https://arxiv.org/abs/2510.06961, sections 2.3 and 3 and Table 4 (read directly)
