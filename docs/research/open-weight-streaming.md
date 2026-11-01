# How open-weight streaming models expose incremental decoding

Research for issue #5 of the streaming ASR track map (issue #3). Checked 2026-11-01. Nothing was downloaded or run, so no model behaviour here has been measured.

Repo paths are at commit `edd6842`. Upstream `main` links can drift, so they name a file and function, not a line. The `NVIDIA-NeMo/NeMo` links now resolve to `NVIDIA-NeMo/Speech` on GitHub. Docs pages for vLLM, Moonshine Voice and the NVIDIA blog were read through a fetch-and-summarise tool, with quoted text checked only within that output. Everything else was read as source.

## Short answer

All four families have a first-party incremental path, but not every path runs on a GPU with several streams.

- **Nemotron:** NeMo has the fullest API: per-step partial and final text, word segments, end-of-utterance detection and multi-stream batching. Transformers has an experimental single-stream generator.
- **Voxtral Realtime:** vLLM's WebSocket API is the recommended path. Transformers has an experimental single-stream generator.
- **Kyutai:** `moshi` exposes a batched frame step, and `kyutai/run_eval.py` already uses it.
- **Moonshine:** only Moonshine Voice is incremental, and it defaults to CPU. The Transformers path that this repo uses has no incremental input.

The repo's Transformers runners feed whole clips and record batch wall time. They skip incremental input, partials and all per-word or per-chunk timing.

**Pilot candidate:** `nvidia/nemotron-speech-streaming-en-0.6b` through the NeMo pipeline, with a NeMo version bump.

## "Latency" means different things per model

These numbers should not share a column without a definition. This matters for the "named latency setting" question on the map.

| Family | Setting | What the number is | Source |
| --- | --- | --- | --- |
| Nemotron | `att_context_size` / `num_lookahead_tokens` | Chunk size, `(lookahead + 1) × 80 ms`. The model waits for a whole chunk. | [card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b), [`streaming_latency_ms`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/processing_nemotron_asr_streaming.py) |
| Voxtral Realtime | `transcription_delay_ms` | Minimum offset between acoustic evidence and the earliest text. Input chunks stay 80 ms, and a word is emitted once it is acoustically complete and the delay has passed. | [paper §2.3, Fig. 2](https://arxiv.org/abs/2602.11298) |
| Kyutai | fixed per checkpoint | Offset of the text stream behind the audio stream. | [card](https://huggingface.co/kyutai/stt-2.6b-en) |
| Moonshine | fixed by architecture | Encoder lookahead of 80 ms. Finalized encoder states need up to 320 ms more audio, and the decoder then runs token by token. | [paper §3.2, §3.4](https://download.moonshine.ai/docs/moonshine_streaming_paper.pdf) |

All four are algorithmic and exclude compute time. The Voxtral paper makes a related point about realtime APIs: "target delay" differs across them and depends on the audio input ([Table 3 caption](https://arxiv.org/abs/2602.11298)).

## Comparison

"HF" means Transformers. Sizes are safetensors files from the Hub listings.

| | Nemotron EN | Nemotron 3.5 | Voxtral Realtime | Moonshine streaming | Kyutai 2.6B / 1B |
| --- | --- | --- | --- | --- | --- |
| **Incremental input** | NeMo steps; HF generator (experimental) | NeMo steps; HF generator (experimental) | vLLM WebSocket; HF generator (experimental) | Moonshine Voice only; none in HF | `moshi` frame step |
| **Settings** | 80, 160, 560, 1120 ms | 80, 160, 320, 560, 1120 ms in NeMo; no 160 ms in HF | 480 ms default; other values by editing `tekken.json` | none | 2.5 s / 0.5 s, fixed |
| **Partial / final** | Separate fields in NeMo; text stream only in HF | Same | `transcription.delta` / `.done` | Line events | Token stream; no final marker |
| **Word timestamps** | Word segments in NeMo; token times in HF | Not separately documented | Not documented | `words` option | Yes |
| **Multi-stream** | NeMo `batch_size` | NeMo `batch_size` | vLLM sessions; no capacity figure | Separate streams; batching not documented | Yes |
| **Weights** | 2.47 GB | 2.55 GB | 8.86 GB | 0.18 / 0.56 / 1.06 GB | 5.23 + 0.38 GB / 1.98 + 0.38 GB |
| **RTX 5080 (16 GB)** | Likely, inferred | Likely, inferred | At the stated minimum | Yes in HF, but not streaming | Likely, inferred |

## Per model

### Nemotron: `nvidia/nemotron-speech-streaming-en-0.6b`

- **Incremental input:**
  - NeMo's [`speech_to_text_cache_aware_streaming_infer.py`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/asr_cache_aware_streaming/speech_to_text_cache_aware_streaming_infer.py) simulates streaming from a manifest with no real-time pacing.
  - NeMo's pipeline ([`base_pipeline.py`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/nemo/collections/asr/inference/pipelines/base_pipeline.py), `transcribe_step(requests)`) takes per-stream requests.
  - Transformers ≥ 5.13 accepts a generator of mel chunks as `input_features` ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)). The docs call this experimental and subject to change ([docs](https://huggingface.co/docs/transformers/en/model_doc/nemotron_asr_streaming)).
- **Chunk and lookahead:** `[70,0]`, `[70,1]`, `[70,6]` and `[70,13]` give 80, 160, 560 and 1120 ms, selectable at inference time without retraining ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)). Published WER averages on the leaderboard datasets are 8.43, 7.67, 7.07 and 6.93 ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)). Chunks must be exact sizes: `1 + 8 × lookahead` mel frames for the first and `8 × (lookahead + 1)` after that ([`_required_stream_chunk_frames`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/generation_nemotron_asr_streaming.py)).
- **Partial and final:**
  - NeMo's `TranscribeStepOutput` has `partial_transcript` (may still change), `final_transcript` (not modified further) and `current_step_transcript` ([`base_pipeline.py`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/nemo/collections/asr/inference/pipelines/base_pipeline.py)).
  - A final is cut at end of utterance. The endpointer defaults to an 800 ms window ([`cache_aware_rnnt.yaml`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/conf/asr_streaming_inference/cache_aware_rnnt.yaml)).
  - Transformers gives text pieces through `TextIteratorStreamer`, with no partial or final distinction ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)).
- **Word timestamps:**
  - NeMo: `asr_output_granularity: word|segment`, with `final_segments` holding per-word or per-segment metadata ([yaml](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/conf/asr_streaming_inference/cache_aware_rnnt.yaml), [`base_pipeline.py`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/nemo/collections/asr/inference/pipelines/base_pipeline.py)).
  - Transformers: `generate` returns `durations`, and `processor.decode(durations=...)` gives token-level times ([`processing_nemotron_asr_streaming.py`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/processing_nemotron_asr_streaming.py)). `durations` comes back when `generate` returns. Live per-token times from the streamer are not documented.
- **Multi-stream:**
  - NeMo's script supports multi-stream with `batch_size>1`, and the pipeline has `streaming.batch_size: 64` and `num_slots: 256` ([script](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/asr_cache_aware_streaming/speech_to_text_cache_aware_streaming_infer.py), [yaml](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/conf/asr_streaming_inference/cache_aware_rnnt.yaml)).
  - Transformers batched streaming is not documented. `generate` has a batch dimension but pulls the next chunk only when every row is exhausted ([`_update_model_kwargs_for_generation`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/generation_nemotron_asr_streaming.py)), and `TextStreamer` raises on batch size above 1 ([`streamers.py`](https://github.com/huggingface/transformers/blob/main/src/transformers/generation/streamers.py)).
  - NVIDIA reports 560 concurrent streams at 320 ms on an H100 and a 24 ms median time-to-final in Daily's pipeline ([blog](https://huggingface.co/blog/nvidia/nemotron-speech-asr-scaling-voice-agents)). Those are vendor figures, and the blog's method was not checked.
- **End of audio:**
  - Transformers has no flush. A short final chunk raises an error, and the message says to pad it ([`_validate_stream_chunk`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/generation_nemotron_asr_streaming.py)). The published example's `while end_idx < audio.shape[0]` loop doesn't pad, so from reading the code it drops the trailing partial chunk ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)).
  - NeMo handles it: `is_last` forces end of utterance, the last step uses `keep_all_outputs=True` with right padding, and `flush_size_in_secs` optionally appends silence, defaulting to 0.0 ([`cache_aware_rnnt_pipeline.py`](https://github.com/NVIDIA-NeMo/NeMo/blob/main/nemo/collections/asr/inference/pipelines/cache_aware_rnnt_pipeline.py), [yaml](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/conf/asr_streaming_inference/cache_aware_rnnt.yaml)).
- **RTX 5080:** VRAM needs are not documented. Weights are 2.47 GB in float32, and the pipeline computes in bfloat16 ([Hub files](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b/tree/main), [yaml](https://github.com/NVIDIA-NeMo/NeMo/blob/main/examples/asr/conf/asr_streaming_inference/cache_aware_rnnt.yaml)). The card lists Blackwell as compatible, but its test hardware doesn't include a GeForce card ([card](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b)). A single stream fitting is inferred from weight size. Stream count must be measured.

### Nemotron: `nvidia/nemotron-3.5-asr-streaming-0.6b`

It uses the same architecture and API as the English model, plus a language prompt. Differences:

- **Settings:** `[56,0]`, `[56,1]`, `[56,3]`, `[56,6]` and `[56,13]` in NeMo ([card](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b)). Transformers supports only `{3: 320, 0: 80, 6: 560, 13: 1120}` ([`processor_config.json`](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b/blob/main/processor_config.json), [docs](https://huggingface.co/docs/transformers/en/model_doc/nemotron3_5_asr)). So 160 ms exists in NeMo but not in Transformers, even though the Transformers docs overview lists 160 ms.
- **Default lookahead:** 3 (320 ms), compared with 13 (1120 ms) for English ([3.5 config](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b/blob/main/config.json), [EN config](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b/blob/main/config.json)).
- **Throughput:** 240 streams at 80 ms and 2,400 at 1120 ms on one H100 ([card](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b)).
- **English:** NVIDIA recommends the English model for English-only use ([card](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b)). The map keeps v1 to English.
- **Word timestamps:** not separately documented. The processor's `decode(durations=...)` signature is the same ([docs](https://huggingface.co/docs/transformers/en/model_doc/nemotron3_5_asr)).

### Voxtral Realtime: `mistralai/Voxtral-Mini-4B-Realtime-2602`

- **Incremental input:**
  - vLLM is the recommended path, at `ws://host/v1/realtime`, taking base64 PCM16 at 16 kHz mono through `input_audio_buffer.append` ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602), [vLLM docs](https://docs.vllm.ai/en/latest/serving/online_serving/speech_to_text/#realtime-api)).
  - Transformers ≥ 5.2 also accepts a generator. The docs call it experimental ([docs](https://huggingface.co/docs/transformers/main/en/model_doc/voxtral_realtime)).
  - The card contradicts itself, with a warning that the model is "only" supported in vLLM above a Transformers section ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)).
- **Chunk and delay:** the decoder takes one step per 80 ms of audio ([paper §2.3](https://arxiv.org/abs/2602.11298)). The default is 480 ms, read from `transcription_delay_ms` in [`tekken.json`](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602/blob/main/tekken.json), and changing it means editing that file ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)). No runtime argument is documented on the Transformers processor ([source](https://github.com/huggingface/transformers/blob/main/src/transformers/models/voxtral_realtime/processing_voxtral_realtime.py)). The stated range is inconsistent on one card: "240ms to 2.4s", "from 80ms to 2.4s", and "any multiple of 80ms between 80 and 1200, as well as 2400" ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)). FLEURS average WER is 12.60, 10.80, 8.72, 7.70 and 6.73 at 160, 240, 480, 960 and 2400 ms ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)).
- **Partial and final:** `transcription.delta` carries incremental text and `transcription.done` carries the final text and usage stats ([vLLM docs](https://docs.vllm.ai/en/latest/serving/online_serving/speech_to_text/#realtime-api)). The docs don't define partial or final semantics beyond those two events. vLLM's streaming-input blog says generally that tokens between input chunks "may be revised" ([blog](https://vllm.ai/blog/2026-01-31-streaming-realtime)). Whether that applies to Voxtral's deltas is not documented.
- **Word timestamps:** not documented as an output of vLLM or Transformers. Emission is frame-synchronous, with a `[W]` token at each word group ([paper §3.1](https://arxiv.org/abs/2602.11298)), so times could probably be derived from token position. That is my inference and is untested.
- **Multi-stream:** vLLM keeps a session per connection with paged KV caches, and `--max-num-batched-tokens` trades throughput for latency ([paper §4](https://arxiv.org/abs/2602.11298), [card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)). No concurrency figure is published. Idle sessions are not preempted and hold memory ([blog](https://vllm.ai/blog/2026-01-31-streaming-realtime)). Transformers documents batched offline only, and its streamer raises on batch size above 1 ([`streamers.py`](https://github.com/huggingface/transformers/blob/main/src/transformers/generation/streamers.py)).
- **End of audio:**
  - vLLM: send `input_audio_buffer.commit` with `final: true` ([vLLM docs](https://docs.vllm.ai/en/latest/serving/online_serving/speech_to_text/#realtime-api)). The server then appends right padding itself ([`buffer_realtime_audio`](https://github.com/vllm-project/vllm/blob/main/vllm/model_executor/models/voxtral_realtime.py)).
  - The padding is `(delay tokens + 1) + 10` tokens, where the 10 is a buffer for word length ([`n_right_pad_tokens`](https://github.com/mistralai/mistral-common/blob/main/src/mistral_common/tokens/tokenizers/audio.py)). At 480 ms that is 17 tokens, about 1.36 s of audio (my arithmetic).
  - Transformers: generation stops when the generator is exhausted ([`_has_unfinished_sequences`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/voxtral_realtime/modeling_voxtral_realtime.py)), with no flush. The published example computes a padded `xaudio` and then uses the unpadded `audio` in every later line, so it appears to omit the padding it says is required ([docs](https://huggingface.co/docs/transformers/main/en/model_doc/voxtral_realtime)). Harness code should not copy it.
- **RTX 5080:** this is the only model with a stated memory figure: "a single GPU with >= 16GB memory" under vLLM ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)). A 16 GB card meets that with no stated headroom. Weights are 8.86 GB in bfloat16 ([Hub files](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602/tree/main)). Lowering `--max-model-len` reduces memory for pre-computed RoPE frequencies ([card](https://huggingface.co/mistralai/Voxtral-Mini-4B-Realtime-2602)).

### Moonshine streaming: `usefulsensors/moonshine-streaming-{tiny,small,medium}`

The runtime decides eligibility here. Moonshine Voice emits text before audio ends. The Transformers path, as documented, does not.

- **Incremental input:**
  - Transformers has no incremental input. The docs show whole-clip `generate` only, and the modeling file has no generator handling ([docs](https://huggingface.co/docs/transformers/main/en/model_doc/moonshine_streaming), [source](https://github.com/huggingface/transformers/blob/main/src/transformers/models/moonshine_streaming/modeling_moonshine_streaming.py)). The card says this path "does not yet implement fully efficient streaming" ([card](https://huggingface.co/usefulsensors/moonshine-streaming-medium)).
  - The [Moonshine Voice](https://moonshine-voice.readthedocs.io/en/latest/api/classes/) runtime has `create_stream()`, `add_audio()` and `update_transcription()`. It lists the same models and WER figures ([models](https://moonshine-voice.readthedocs.io/en/latest/models/available-models/)).
- **Chunk and lookahead:** encoder windows are `(16,4)` on the first and last two layers and `(16,0)` elsewhere, giving 80 ms lookahead and up to 320 ms for a finalized state ([paper §3.2](https://download.moonshine.ai/docs/moonshine_streaming_paper.pdf)). There is no setting to change this. In Moonshine Voice, `transcription_interval` defaults to 0.5 s ([options](https://moonshine-voice.readthedocs.io/en/latest/api/options/)). Measured first-token times of 13.5, 65.1 and 129.8 ms are at 1 s of audio on an Apple M3 with ONNX Runtime ([paper Table 3](https://download.moonshine.ai/docs/moonshine_streaming_paper.pdf)), so they don't transfer to an H200 or RTX 5080.
- **Partial and final:** line events `on_line_started`, `on_line_updated`, `on_line_text_changed` and `on_line_completed` ([API](https://moonshine-voice.readthedocs.io/en/latest/api/classes/)). `decode_incomplete_lines` defaults on, and speculative decoding re-verifies the previous hypothesis, so partial text can change ([options](https://moonshine-voice.readthedocs.io/en/latest/api/options/), [paper §3.2](https://download.moonshine.ai/docs/moonshine_streaming_paper.pdf)). VAD splits lines, with a 15 s default maximum ([options](https://moonshine-voice.readthedocs.io/en/latest/api/options/)).
- **Word timestamps:** a `word_timestamps` option fills each line's `words` array and needs an extra decoder asset ([options](https://moonshine-voice.readthedocs.io/en/latest/api/options/)). Not documented for Transformers.
- **Multi-stream:** `create_stream()` gives one stream per source on one model copy ([API](https://moonshine-voice.readthedocs.io/en/latest/api/classes/)). Batching across streams is not documented.
- **End of audio:** `stop()` closes any open segment and fires its events ([API](https://moonshine-voice.readthedocs.io/en/latest/api/classes/)). `MicTranscriber.stop()` also flushes in-flight audio.
- **RTX 5080:** weights are small, at most 1.06 GB ([Hub files](https://huggingface.co/usefulsensors/moonshine-streaming-medium/tree/main)), so the Transformers path should fit, but it isn't streaming. Moonshine Voice defaults to CPU-only `ort_providers` and recommends that ([options](https://moonshine-voice.readthedocs.io/en/latest/api/options/)). GPU streaming is not documented. Stream capacity on an H200 would not be meaningful for this runtime without one.

### Kyutai: `kyutai/stt-2.6b-en`, `kyutai/stt-1b-en_fr`

- **Incremental input:** `moshi` steps one Mimi frame at a time: `mimi.encode(chunk)`, then `lm_gen.step(tokens)` inside `mimi.streaming(bsz)` and `lm_gen.streaming(bsz)` ([`stt_from_file_pytorch.py`](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/scripts/stt_from_file_pytorch.py)). Frames run at 12.5 Hz ([card](https://huggingface.co/kyutai/stt-2.6b-en)), so each step is 80 ms. Kyutai's Rust server is its stated production path ([README](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/README.md)). The Transformers port documents offline inference only ([docs](https://huggingface.co/docs/transformers/en/model_doc/kyutai_speech_to_text)).
- **Delay:** fixed per checkpoint: 2.5 s for 2.6B and 0.5 s for 1B ([2.6B config](https://huggingface.co/kyutai/stt-2.6b-en/blob/main/config.json), [1B config](https://huggingface.co/kyutai/stt-1b-en_fr/blob/main/config.json)). A leaderboard entry per setting therefore means one entry per checkpoint.
- **Partial and final:** the first-party script prints each non-padding token as it arrives, with no partial or final distinction ([script](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/scripts/stt_from_file_pytorch.py)). 1B adds a semantic VAD through `step_with_extra_heads`, loaded from `kyutai/stt-1b-en_fr-candle` in that script.
- **Word timestamps:** yes. Word boundaries come from `end_of_padding` tokens, with times computed from frame offset minus the silence prefix and delay ([script](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/scripts/stt_from_file_pytorch.py)). The Rust example has `--timestamps` ([README](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/README.md)).
- **Multi-stream:** "a H100 can process 400 streams in real-time", and the Rust server handles 64 connections at 3x real time on an L40S, with batch size set in its config according to GPU memory ([README](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/README.md)).
- **End of audio:** append `audio_delay_seconds` of silence and keep stepping ([script](https://github.com/kyutai-labs/delayed-streams-modeling/blob/main/scripts/stt_from_file_pytorch.py)). So final text arrives one delay after speech ends, unless silence is fed faster than real time.
- **RTX 5080:** VRAM is not documented, and no consumer-GPU figure is given. Weights plus Mimi are about 5.6 GB (2.6B) and 2.4 GB (1B) ([2.6B files](https://huggingface.co/kyutai/stt-2.6b-en/tree/main), [1B files](https://huggingface.co/kyutai/stt-1b-en_fr/tree/main)). Fit is inferred from weight size.

### Other streaming models

The only streaming-named open-weight ids in the repo's scripts are the ones above. `meta/muse-voice-transcribe-streaming` is an API provider, covered by another ticket. Two NVIDIA models are not in the repo's scripts:

- [`nvidia/parakeet_realtime_eou_120m-v1`](https://huggingface.co/nvidia/parakeet_realtime_eou_120m-v1) has 80–160 ms latency and emits an `<EOU>` token. It reports 9.30% average WER on the leaderboard datasets at 160 ms, plus end-of-utterance latency percentiles, which makes it a useful reference for finalization latency.
- [`nvidia/multitalker-parakeet-streaming-0.6b-v1`](https://huggingface.co/nvidia/multitalker-parakeet-streaming-0.6b-v1) is a multi-speaker model and is out of scope here.

I could not read the leaderboard results CSV (`hf-audio/open-asr-leaderboard-results` returned 401), so the live table may list others.

## What the repo's runners bypass

`--streaming` in these scripts is lazy dataset loading, not model streaming ([`transformers/run_eval.py:628`](../../transformers/run_eval.py)).

### `transformers/run_eval.py`

**All three families:**
- Every clip is complete before processing starts. There is no chunk loop, generator or streamer.
- Audio padding depends on the longest clip in the batch.
- Timing is one wall-clock number for the batch, divided by batch size ([`:239-243`](../../transformers/run_eval.py), [`:452-458`](../../transformers/run_eval.py)). No emission time, partial, or per-word time is recorded.

**Nemotron:**
- The offline processor call and `model.generate(**inputs)` skip the generator path ([`:253-263`](../../transformers/run_eval.py), [`:363-366`](../../transformers/run_eval.py)).
- It does not skip the lookahead limit. The processor adds `num_lookahead_tokens` to its output for offline calls too ([source](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/processing_nemotron_asr_streaming.py)), and the encoder applies a `chunked_limited` attention mask from that value ([source](https://github.com/huggingface/transformers/blob/main/src/transformers/models/nemotron_asr_streaming/modeling_nemotron_asr_streaming.py)). Batch WER should therefore approximate streaming WER at that setting, though this hasn't been tested.
- The two ids run at different effective latencies without saying so: 1120 ms for English and 320 ms for 3.5, because the runner keeps each checkpoint's default. Neither `--batch_size` nor any other flag changes it, and `submit_jobs_nemotron.sh` passes no latency option. Comparing their current leaderboard WERs compares different latency settings.

**Voxtral Realtime:**
- The plain processor call uses `StreamingMode.OFFLINE` ([`:272-274`](../../transformers/run_eval.py), [source](https://github.com/huggingface/transformers/blob/main/src/transformers/models/voxtral_realtime/processing_voxtral_realtime.py)). The conv front end embeds the whole clip up front ([`_prepare_model_inputs`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/voxtral_realtime/modeling_voxtral_realtime.py)), and generation length is derived from the known audio length.
- The delay of 480 ms is still applied, along with right padding. The encoder stays causal by architecture ([paper §2.1](https://arxiv.org/abs/2602.11298)). `submit_jobs_voxtral_realtime.sh` has no delay option.

**Moonshine:**
- Plain batched `generate`, with a token cap of 6.5 tokens per second plus 2 ([`:370-387`](../../transformers/run_eval.py)). The cap uses each clip's known duration, which a true streaming run wouldn't have. The cap comes from the card's hallucination guard.
- Encoder attention windows still apply ([`sliding_window_mask_function`](https://github.com/huggingface/transformers/blob/main/src/transformers/models/moonshine_streaming/modeling_moonshine_streaming.py)), so the encoder is limited to its lookahead. But the decoder sees the encoder output for the whole clip before its first token, so this is not streaming decoding, and the batch WER may not match streaming WER. Whether it does is not documented.
- All model lines in `submit_jobs_moonshine.sh` are commented out. `run_moonshine.sh` still lists them.

### `transformers/run_eval_ml.py`

Same pattern. It also cuts Nemotron clips to 30 s to avoid an indexing limit with long padded batches ([`:240-245`](../../transformers/run_eval_ml.py)). Multilingual is out of scope for v1.

### `kyutai/run_eval.py`

This is the only runner using the model's real incremental API ([`:116-121`](../../kyutai/run_eval.py)). What it skips:
- Clips are resampled and padded up front, and zero-padded to the longest in the batch ([`:46-72`](../../kyutai/run_eval.py)).
- All streams start together and step in lock-step until the longest ends, so there is no staggered arrival.
- Tokens are kept and decoded at the end ([`:123-129`](../../kyutai/run_eval.py)). No per-frame time is stored, and `end_of_padding` boundaries are not used for word times.
- Time covers the whole batch, including padding and silence ([`:132-135`](../../kyutai/run_eval.py)).

It does keep the correct silence prefix and delay suffix from `stt_config`. The fallback of 5.0 s doesn't apply to either checkpoint, since both configs set the value. Only `kyutai/stt-2.6b-en` appears: `run_kyutai.sh` lists it, and its line is commented out in `submit_jobs.sh` ([`:29`](../../kyutai/submit_jobs.sh)). `stt-1b-en_fr`, the only sub-second Kyutai checkpoint, is not in the repo.

## Pilot candidate: `nvidia/nemotron-speech-streaming-en-0.6b`

Run it through the NeMo pipeline, with the Transformers generator as a single-stream cross-check.

**Why this one:**
- **Named settings:** one checkpoint has four documented settings from 80 to 1120 ms, which matches the map's "model plus named latency setting" entries.
- **Headline metrics:** `is_last`, end-of-utterance finals and partials are already separated, so streaming WER and finalization latency can be read from the step output. The config also mentions RTFx and LAAL calculation.
- **Stream capacity:** `batch_size` is a documented multi-stream path, so capacity can be tried in the pilot.
- **Raw partials:** `partial_transcript`, `current_step_transcript` and word segments cover the map's kept-for-later caption metrics.
- **Cross-checks:** the card publishes WER per setting on the leaderboard datasets, and the existing batched run should approximate the 1120 ms setting. A large gap would point to a harness bug.
- **Resources:** its 2.47 GB of weights are well below Voxtral's 8.86 GB and Kyutai 2.6B's 5.6 GB. Kyutai 1B is similar in size but isn't in the repo. So an RTX 5080 should have room for several streams, though that is inferred and needs measuring.

**Why not the others:**
- **Voxtral Realtime** is the runner-up and the best second model. Its accuracy claims are strong, but it sits at its stated 16 GB minimum, needs a vLLM server, changes delay by editing a tokenizer file, and has no documented word timestamps.
- **Kyutai** has the simplest API, but 2.6B has one 2.5 s setting, which rules out voice-agent use, and 1B isn't in the repo.
- **Moonshine** has no documented incremental GPU path.
- **Nemotron 3.5** adds a language prompt and nothing else that v1 needs, and NVIDIA recommends the English model for English.

**Caveats for the pilot:**
- `requirements/requirements_nemo.txt` pins NeMo commit `6294bc7` from July 2025. `nemo/collections/asr/inference/pipelines` does not exist at that commit, although the older cache-aware script does. The card lists NeMo 25.11 or higher, so the pilot needs its own environment. `requirements/requirements_jobs.txt` says `transformers>=4.40`, while Nemotron needs ≥ 5.13.
- Mistral's paper reports weak Nemotron long-form results of 14.29 and 13.02 at 560 and 1120 ms, compared with 7.73 for Voxtral at 480 ms ([Table 3](https://arxiv.org/abs/2602.11298)). That is a competitor's measurement. It doesn't affect choosing a pilot for testing the method.
- The pilot's cross-check should verify the 800 ms endpointer's effect. Finals cut by end of utterance and finals forced by `is_last` will give different finalization latencies, so the spec needs to pick one.

## For later tickets

- **Harness interface:** a family has to supply four things: push audio, read new text as it appears, signal end of audio, and read final text. All four families can do this in a first-party runtime, but the end-of-audio contract differs. It can mean padding a final chunk, sending `final: true`, appending silence, or calling `stop()`.
- **Finalization latency:** model-required padding sets a floor unless it's fed faster than real time: about 1.36 s of padding for Voxtral at 480 ms and 2.5 s for Kyutai 2.6B. Whether padding counts in real time needs a decision, since it decides what the metric measures.
- **Revisions:** Nemotron's NeMo partials and Moonshine's partials can change. Raw partials need to be stored as snapshots, not deltas.
- **Eligibility:** the behavioural test should be applied to a model plus a runtime. Moonshine in Transformers and Moonshine Voice would get different answers.

## Not documented or not verified

- VRAM use and stream capacity on an RTX 5080 for any model. The only stated GPU memory figure is Voxtral's 16 GB.
- Concurrency for Voxtral under vLLM.
- Batched streaming in Transformers for Nemotron and Voxtral.
- Word timestamps for Voxtral in vLLM or Transformers.
- Moonshine Voice GPU support for streaming, and its batching.
- Whether batch WER equals streaming WER at the same setting, for any model.
- Whether the vLLM server feeds right padding faster than real time after `final: true`.
- The contents of the leaderboard results CSV, which returned 401.
- NVIDIA's throughput and 24 ms time-to-final figures, which are vendor-reported.
