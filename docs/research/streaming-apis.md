# How streaming ASR APIs expose partials, finals, timestamps and finalization

Research for issue #4 of the streaming track map (#3). Read on 2026-11-01.

**Everything about vendor behaviour here is documented, not tested.** No API keys were available, so nothing has been run against a live service. "Not documented" means the first-party pages listed in the row don't say; it isn't proof the feature is absent. Timestamps and clock statements are quoted as the vendor wrote them. Anything I inferred is labelled as an inference.

**Scope:** providers in `api/providers/` and upstream PR 166 (PyAI Hear). Providers with no documented streaming endpoint are listed under [Gaps](#gaps-and-things-i-could-not-verify).

## Short answer

All 16 providers below take audio incrementally and return interim text followed by stable finals (Sprag and OpenAI `gpt-transcribe` only per closed turn), but they agree on little else:

- Pacing rules range from "the server closes you" to "no rule stated".
- Several have no word timestamps on the streaming path.
- Almost no server field tells you when text was produced, so finalization latency has to be measured from the client's receive clock.
- Ten document a per-turn force-finalize, Meta and Modulate have none, and Gladia, Rev.ai, Azure and Zoom document none on the pages read. Some vendors document an accuracy cost for forcing.
- Most streaming endpoints name a different model from the one on the batch leaderboard.

## Summary table

"Force" means ending one turn while the session stays open. "EOS" means ending the whole input. Audio positions are the fields the API reports for where text sits in the audio.

| Provider | Pacing rule | Partials on by default? | Finest timestamp on stream | Force a turn | End of input | Latency settings | Server timing fields |
|---|---|---|---|---|---|---|---|
| Meta | Real time required; closes on >5 s backlog or below real time | Yes | Turn (`audioProcessedMs`, "not a precise acoustic boundary") | **None** | `endStream` | Mode only | `audioProcessedMs` |
| AssemblyAI | Pace about 1x; throttled at ~1.25x; closes at 5 min buffered | Yes | Word, partial and final | `ForceEndpoint` | `Terminate` | `mode`, turn silence, EOT threshold | `Heartbeat`, `Termination` |
| ElevenLabs | Not documented (examples sleep; delay "optional") | Yes | Word, committed only, opt-in | `commit` | Not documented beyond close | `commit_strategy`, VAD silence | None documented |
| Speechmatics | Faster than real time buffered, then backpressure | No | Word, partial and final | `ForceEndOfUtterance` | `EndOfStream` | `max_delay`, `max_delay_mode`, EOU trigger | `AudioAdded` ack only |
| Soniox | Not documented | Yes | Token | `finalize` | Empty text frame | Endpoint level, sensitivity, max delay | `final_audio_proc_ms`, `total_audio_proc_ms` |
| Gladia | Not documented | No | Utterance (word schema unconfirmed) | Not documented | `stop_recording` | `endpointing`, max duration | `created_at` on messages |
| Rev.ai | Faster than real time allowed | Yes | Word, final; partial only with `detailed_partials` | Not documented | `"EOS"` | `priority`, `max_segment_duration_seconds` | None documented |
| OpenAI | Not documented | Yes (`gpt-live-transcribe`) | None (`gpt-live-transcribe`) | `input_audio_buffer.commit` (required) | Not documented | `delay` minimal to xhigh | None documented |
| Gemini Live | Not documented (100 ms chunks) | Yes | Not word level | `audio_stream_end` or `activityEnd` | `audio_stream_end` | VAD silence and padding | `usageMetadata` tokens |
| Azure | Not documented | Yes | Word, final only | Not documented | Stop recognition | Segmentation silence, stable partial threshold | None documented |
| Smallest Pulse | Chunk every 50-100 ms | Yes | Word, opt-in | `finalize` | `close_stream` | `eou_timeout_ms`, `endpointing`, `max_words` | None in the stream response |
| Reson8 | Not documented (examples unpaced) | No | Word, opt-in | `flush_request` | Socket close | None on `realtime`; turn probabilities on `turns` | None documented |
| Modulate | "Any size"; no rate rule | Multilingual no; English yes | Utterance; word opt-in (English) | **None** | Empty text frame | `endpointing` on/off (English) | `done.duration_ms` |
| Zoom Scribe Live | ~100 ms frames; no rate rule | No | Item | Not documented | `session.close` | Not documented | `transcription_latency_ms` (undefined) |
| Sprag | Not documented | Deltas documented, but turn is transcribed after it closes | None in realtime | `commit` (with `turn_detection: null`) | Not documented | `silence_duration_ms` | `usage` audio duration |
| PyAI Hear (PR 166) | ~20 ms frames; no rate rule | `interim_results` in examples | Hypothesis (`t_ms`) | `commit` | Socket close flushes | `endpointing_ms` | `t_ms`, `audio_ms`, `endpoint_reason`, `usage` |

## What this means for benchmark design

These are suggestions for the later tickets, not decisions.

1. **Pacing has to be real time, and runs cost wall-clock time.**
   - Meta closes with `1008` for "backlog or below-real-time ingress" ([close codes](https://dev.meta.ai/docs/api-reference/voice/realtime)).
   - AssemblyAI warns that sending in a tight loop "can produce inconsistent Turn messages" ([API spec](https://assemblyai.com/docs/streaming/api-spec/streaming-websocket.md)).
   - Two vendors explicitly permit faster-than-real-time audio. Rev.ai says it is processed and catches up ([requests](https://docs.rev.ai/api/streaming/requests/)), and AssemblyAI allows it but throttles at ~1.25x ([message sequence](https://assemblyai.com/docs/streaming/message-sequence.md)). Speechmatics warns it can end in a closed connection ([API reference](https://docs.speechmatics.com/api-ref/realtime-transcription-websocket)). The rest state no rule on the pages read.
   - Billing can follow the same clock. AssemblyAI bills `session_duration_seconds` ([message sequence](https://assemblyai.com/docs/streaming/message-sequence.md)), Soniox charges "for the **full stream duration,** not just the audio processed" ([manual finalization](https://docs.soniox.com/stt/rt/manual-finalization)), and Rev.ai bills the larger of audio and stream duration ([streaming overview](https://docs.rev.ai/api/streaming/)). So trailing silence and slow draining cost money.
2. **Finalization latency must come from the client's receive clock.** Reported timestamps locate text in the audio, not when it was produced. The exceptions are thin:
   - Gladia puts `created_at` on messages, in wall-clock form but undefined ([quickstart](https://docs.gladia.io/chapters/live-stt/quickstart.md)).
   - Zoom reports `transcription_latency_ms`, which neither the [Live page](https://developers.zoom.us/docs/ai-services/scribe/live-mode/) nor the part of the [API reference](https://developers.zoom.us/docs/api/ai-services/) I could read defines.
   - A harness can timestamp each send and receive, then compute arrival time minus the send time of the last speech sample. This is the shape of the vendor-neutral metric Smallest describes: "time between a speaker stopping and the final transcript arriving" ([measuring latency](https://docs.smallest.ai/models/speech-to-text/benchmarks/measuring-latency.md)).
3. **"Raw timestamped partials" can't be uniform, and asking for them can change the result.**
   - Meta, OpenAI `gpt-live-transcribe`, Gemini Live and Sprag have no word timings on the stream. ElevenLabs, Azure and Modulate English give them on finals only. Client arrival time is the only timing common to all.
   - Rev.ai documents "a slight degradation of 1% in word error rate" on finals when `detailed_partials` is on ([requests](https://docs.rev.ai/api/streaming/requests/)).
   - Partials may not come from the same model as finals: Gladia says partials "use a faster, smaller model" ([partial transcripts](https://docs.gladia.io/chapters/live-stt/features/partial-transcripts.md)). Speechmatics says partial accuracy is "usually 10-25% lower" ([output](https://docs.speechmatics.com/speech-to-text/realtime/output)).
   - So partial-on versus partial-off should be recorded per entry, because partials are off by default on several APIs.
4. **Forcing a final isn't available everywhere and isn't neutral.**
   - Meta: "`realtime` has no client-side turn commit" ([migration notes](https://dev.meta.ai/docs/speech-to-text)). Modulate only has end of audio. Gladia, Rev.ai, Azure and Zoom document no per-turn control on the pages read.
   - OpenAI is the reverse: `gpt-live-transcribe` has no server VAD, so the client must decide when a turn ends ([realtime transcription](https://developers.openai.com/api/docs/guides/realtime-transcription)). A harness therefore has to supply an endpointer for that one, which makes part of the latency the harness's own.
   - Forcing has documented costs. Soniox says to finalize only after "approximately 200ms of silence" or accuracy can degrade ([manual finalization](https://docs.soniox.com/stt/rt/manual-finalization)). ElevenLabs says rapid commits can degrade performance ([commit strategies](https://elevenlabs.io/docs/developers/guides/cookbooks/speech-to-text/realtime/transcripts-and-commit-strategies)).
   - A consistent method is probably "append paced trailing silence and let the server endpoint", with forcing as a separately labelled setting. Silence has to be streamed, because endpoint timers run on received audio: PyAI's "timer advances from incoming frames; if audio stops, the pause timer stops too" ([streaming guide](https://docs.pyai.com/guides/streaming-stt.md)), and Meta says to keep sending PCM silence ([session rules](https://dev.meta.ai/docs/speech-to-text)).
5. **"The final" needs a per-provider definition, or the latency clock stops at different points.**
   - AssemblyAI Universal Streaming with `format_turns=true` sends two `end_of_turn: true` messages per turn ([message sequence](https://assemblyai.com/docs/streaming/message-sequence.md)).
   - PyAI sends `speech_final`, then a corrected `final` ([streaming guide](https://docs.pyai.com/guides/streaming-stt.md)).
   - Meta sends `speechEnd`, then a post-processed `speechComplete` that can differ from the last partial; `transcript` with `final: true` only completes `PUSH_TO_TALK` ([endpointing](https://dev.meta.ai/docs/speech-to-text)).
   - Speechmatics `flexible` mode can hold a final until an entity is complete ([output](https://docs.speechmatics.com/speech-to-text/realtime/output)).
6. **The defaults aren't the low-latency settings, so a "default" entry would be misleading.**
   - Speechmatics defaults to `max_delay` 4 s, with partials off and end-of-utterance detection off ([API reference](https://docs.speechmatics.com/api-ref/realtime-transcription-websocket)).
   - Modulate English returns one final at end of stream unless `endpointing=true` ([English streaming](https://docs.modulate.ai/api-reference/stt/streaming-vfast.md)).
   - Meta's default `PUSH_TO_TALK` is a single turn ([modes](https://dev.meta.ai/docs/speech-to-text)), and the repo's comment says it "leads to much worse results" (`api/providers/meta_provider.py`, not independently verified).
7. **The streaming model usually isn't the leaderboard model.** This bears on the map's "batch WER shown beside it as a reference" decision, because the two numbers often describe different models.

   | Leaderboard id (`api/run_api.sh`) | Streaming model the docs name | Source |
   |---|---|---|
   | `meta/muse-voice-transcribe` | Same model: "Both endpoints run the same model" | [Meta](https://dev.meta.ai/docs/speech-to-text) |
   | `gladia/solaria-3` | `solaria-1` only | [Gladia](https://docs.gladia.io/chapters/live-stt/quickstart.md) |
   | `soniox/stt-async-v5` | `stt-rt-v5` | [Soniox](https://docs.soniox.com/stt/api-reference/websocket-api) |
   | `elevenlabs/scribe_v2` | `scribe_v2_realtime` | [ElevenLabs](https://elevenlabs.io/docs/api-reference/speech-to-text/v-1-speech-to-text-realtime) |
   | `openai/gpt-transcribe` | `gpt-live-transcribe` recommended; `gpt-transcribe` usable per committed turn | [OpenAI](https://developers.openai.com/api/docs/guides/realtime-transcription) |
   | `gemini/gemini-3.5-transcribe` | `gemini-3.5-transcribe-live`, with a different feature table | [Gemini](https://ai.google.dev/gemini-api/docs/models/gemini-3.5-transcribe) |
   | `zoom/scribe_v2.1_pro` | `zoom-scribe-multilingual` or `whisper` | [Zoom](https://developers.zoom.us/docs/ai-services/scribe/live-mode/) |
   | `revai/fusion`, `revai/machine` | `machine`, `machine_v2`; `fusion` not mentioned on the pages read | [Rev.ai](https://docs.rev.ai/api/streaming/requests/) |
   | `modulate/multilingual` | Separate `velma-2-stt-streaming` endpoint | [Modulate](https://docs.modulate.ai/api-reference/stt/streaming.md) |
   | `sprag/symphony` | Examples use `rhythm`; the model list was not checked | [Sprag](https://sprag.ai/docs/stt/api/realtime) |
   | `speechmatics/enhanced` | `enhanced` (same name; sameness of model not stated) | [Speechmatics](https://docs.speechmatics.com/speech-to-text/realtime/output) |
   | `microsoft/azure-speech-07-2026`, `reson8/*` | Mapping not documented on the pages read | |

8. **Behavioural eligibility has edge cases.**
   - `gpt-transcribe` begins "after a committed audio turn" ([OpenAI](https://developers.openai.com/api/docs/guides/realtime-transcription)).
   - Sprag says transcripts arrive after `speech_stopped`, "not word by word" ([Sprag](https://sprag.ai/docs/stt/api/realtime)).
   - Both emit text before the *audio* ends only if the clip has more than one turn. The test on the map ("emits text before the audio ends") may need to say whether that counts.
   - Modulate English partials arrive only about every 1.5 s, and ElevenLabs does not begin processing until the first ~2 s of audio ([Modulate](https://docs.modulate.ai/api-reference/stt/streaming-vfast.md), [ElevenLabs](https://elevenlabs.io/docs/developers/guides/cookbooks/speech-to-text/realtime/transcripts-and-commit-strategies)).
9. **Session caps constrain long-form audio.** Gemini Live 10 min, Meta 60 min, Sprag 1 h, Zoom 120 min, Soniox 300 min, AssemblyAI, Gladia and Rev.ai 3 h, Speechmatics 48 h (sources in the provider tables). Long clips would need a stitching rule for some providers.

## Notes on code in this repo

- **`meta_provider.py` is the only streaming provider.** It paces 80 ms frames, uses `ENDPOINTING`, sends trailing silence until `speechComplete` (up to 10 s), then sends `endStream`. It drops timing: `audioProcessedMs` is never read, and `emitAudioProgress` is `False`.
- **Possible bug, found by reading only, not tested.** `state["complete"]` becomes true on the *first* `speechComplete`, and the trailing-silence loop runs only while it's false. In a multi-turn clip, an earlier turn finishing mid-clip means no trailing silence is sent for the last turn. The code's own comment says that loses the post-processed text. Because the function returns `finals` whenever any exist, the last partial would also be dropped.
- **Handshake order differs from the docs.** The docs say to "Wait for the handshake acknowledgement before sending audio" ([lifecycle](https://dev.meta.ai/docs/speech-to-text)); the provider starts sending immediately after the handshake.
- **PR 166 tests batch, not streaming.** Per its description, the provider uses the multipart POST to `/v1/audio/transcriptions`, even though the docs publish a separate WebSocket ([PR 166](https://github.com/huggingface/open_asr_leaderboard/pull/166), [PyAI](https://docs.pyai.com/guides/streaming-stt.md)). The PR says `pyai/hear-v4` while the docs say `pyai-hear`, and its body and code disagree on free minutes (10,000 versus 1,000,000 a month).

## Provider details

### Meta Muse Voice (`meta/muse-voice-transcribe-streaming`)

Source for all rows unless noted: [Speech to text](https://dev.meta.ai/docs/speech-to-text).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.meta.ai/v1/asr/realtime`. Raw PCM16 mono at 24 or 16 kHz in binary frames; frame boundaries carry no meaning. "Pace audio in real time"; no more than five seconds ahead of processing. Close code `1008` covers "backlog or below-real-time ingress" ([close codes](https://dev.meta.ai/docs/api-reference/voice/realtime)). |
| Partials and finals | `transcript` with `final`. `CUMULATIVE` partials replace the previous one and "can revise or remove earlier text". Partials have no `turnId`. Completion is `final: true` in `PUSH_TO_TALK`, `speechComplete` per `turnId` otherwise. Turns can overlap. |
| Timestamps and clock | "turn-level timestamps, not word-level". `audioProcessedMs` is "milliseconds from the start of the stream" and "Not a precise acoustic boundary" ([schemas](https://dev.meta.ai/docs/api-reference/voice/schemas)). `startMs`/`endMs` per turn appear in the `transcribe` response. |
| Finalization | "`realtime` has no client-side turn commit." `{"type":"endStream"}` ends the whole session and flushes; read until close `1000`. Closing the socket instead "can discard pending events". |
| Latency settings | Not documented. `mode` is fixed per session, and `DIARIZATION` is "not tuned for low-latency". The API reference lists nine handshake fields; I read the guide's fields, not the full schema list. |
| Server timing | `audioProcessedMs`, plus `audioProgress` events by default. |
| Limits | 60 min session; 256 concurrent streams; 16,000 streams per hour. |

### AssemblyAI (`assemblyai_provider.py`; not in the run scripts' model lists)

Source: [message sequence](https://assemblyai.com/docs/streaming/message-sequence.md) unless noted.

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Binary frames, ~50 ms PCM. "throughput is throttled at ~1.25× real time"; over five minutes buffered closes with `3007`. Chunks must be 50-1000 ms, and tight-loop sending "can produce inconsistent Turn messages" ([API spec](https://assemblyai.com/docs/streaming/api-spec/streaming-websocket.md)). |
| Partials and finals | `Turn` with `end_of_turn: false` then `true`. Each `Turn` supersedes the previous. Complete when `end_of_turn` and `turn_is_formatted` are both true. Universal Streaming can send two finals per turn. |
| Timestamps and clock | `words[]` with `start`, `end` in milliseconds "relative to the beginning of the audio stream" ([API spec](https://assemblyai.com/docs/streaming/api-spec/streaming-websocket.md)), and `word_is_final`, present on partials too. `SpeechStarted.timestamp` on Universal-3.6 Pro only. |
| Finalization | `ForceEndpoint` responds "right away, without waiting for silence or punctuation". `Terminate`, then read until `Termination`; closing early "silently discards your last transcript". |
| Latency settings | `mode` = `min_latency`, `balanced`, `max_accuracy` ([optimizing](https://assemblyai.com/docs/streaming/getting-started/optimizing-accuracy-and-latency.md)). `min_turn_silence`, `max_turn_silence`, `interruption_delay` default per mode ([turn detection](https://assemblyai.com/docs/streaming/turn-detection.md)). `end_of_turn_confidence_threshold` for Universal Streaming. Mid-session updates are allowed. |
| Server timing | `Heartbeat` every 5 s with `total_audio_received_ms`, `total_duration_ms`, `realtime_factor` (with `session_heartbeat=true`). `Termination` has audio and session durations. |
| Limits | Sessions auto-close at 3 h. Unknown query parameters are ignored, not rejected, so check the `Begin` echo. |

### ElevenLabs Scribe (`scribe_v2_realtime`)

Sources: [API reference](https://elevenlabs.io/docs/api-reference/speech-to-text/v-1-speech-to-text-realtime) (AR), [commit strategies](https://elevenlabs.io/docs/developers/guides/cookbooks/speech-to-text/realtime/transcripts-and-commit-strategies) (CS), [events](https://elevenlabs.io/docs/eleven-api/guides/how-to/speech-to-text/realtime/event-reference) (EV).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | JSON `input_audio_chunk` with base64 audio (AR). Send 0.1-1 s chunks; the server buffers the first ~2 s; without client messages it closes after 15 s (CS). No rate rule; `queue_overflow` and `chunk_size_exceeded` errors have no thresholds (EV). |
| Partials and finals | `partial_transcript`, then `committed_transcript` (AR). A session has several commits (CS). |
| Timestamps and clock | `committed_transcript_with_timestamps` with `include_timestamps=true`; `start`, `end` in seconds (AR). Zero reference not documented. Can't be combined with `filter_background_audio` (CS). |
| Finalization | `commit: true` on a chunk (AR). Commit every 20-30 s is advised, with an auto-commit at about 36 s of audio; rapid commits can hurt performance; `commit_throttled` exists (CS, EV). |
| Latency settings | `commit_strategy` = `manual`, `vad`, `turn_prediction` (AR). `vad_silence_threshold_secs` 0.192-3.0 (CS). |
| Server timing | Not documented. |
| Limits | Session cap not stated on the pages read. |

### Speechmatics (`speechmatics/enhanced`)

Sources: [API reference](https://docs.speechmatics.com/api-ref/realtime-transcription-websocket) (AR), [output](https://docs.speechmatics.com/speech-to-text/realtime/output) (OUT), [turn detection](https://docs.speechmatics.com/speech-to-text/realtime/turn-detection) (TD), [limits](https://docs.speechmatics.com/speech-to-text/realtime/limits) (LIM).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Binary `AddAudio`, acknowledged by `AudioAdded` with `seq_no`. "When clients send audio faster than real-time, the server may read data slower than it's sent", which can fill TCP buffers and close the connection (AR). |
| Partials and finals | `AddPartialTranscript` (needs `enable_partials`, default false) and `AddTranscript`. "On each Final transcript you will immediately receive a Partial transcript with any remaining words" (OUT). |
| Timestamps and clock | `metadata` and per-word `start_time`, `end_time` in seconds (AR). The zero reference is not stated for these fields; `ForceEndOfUtterance.timestamp` is "seconds since the beginning of the audio", so audio-relative is my inference. |
| Finalization | `ForceEndOfUtterance` returns `AddTranscript` then `EndOfUtterance` with `forced: true` (TD). `end_of_utterance_silence_trigger` is 0-2 s; 0 disables; keep it below `max_delay` (TD). `EndOfStream` with `last_seq_no`, then `EndOfTranscript` (AR). |
| Latency settings | `max_delay` 0.7-4 s, default 4, defined as the delay "between the end of a spoken word and returning the Final transcript"; `max_delay_mode` `flexible` or `fixed` (OUT). Partials latency is not affected by `max_delay`. Vendor guidance gives <5% degradation at 0.7-1.5 s and about 1% at 2 s, relative to batch (OUT). |
| Server timing | None for latency; `Info` types documented are `recognition_quality` and `concurrent_session_usage` (AR). |
| Limits | 48 h session; 1 h without audio; 3 min without audio or ping (LIM). |

### Soniox (`stt-rt-v5`)

Sources: [real-time](https://docs.soniox.com/stt/rt/real-time-transcription) (RT), [WebSocket API](https://docs.soniox.com/stt/api-reference/websocket-api) (WS), [endpoint detection](https://docs.soniox.com/stt/rt/endpoint-detection) (EP), [manual finalization](https://docs.soniox.com/stt/rt/manual-finalization) (MF), [limits](https://docs.soniox.com/stt/rt/limits-and-quotas) (LIM), [keepalive](https://docs.soniox.com/stt/rt/connection-keepalive) (KA).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Binary audio; no rate rule on the pages read. Without audio or keepalive for 20 s the stream ends with 408 (KA). |
| Partials and finals | Token stream with `is_final`. Non-final tokens may change or disappear; final tokens "are sent only once and never repeated" (RT). So the client accumulates finals and replaces non-finals. |
| Timestamps and clock | Token `start_ms`, `end_ms`, in milliseconds (WS). The example in RT reads `final_audio_proc_ms` as audio position; a zero reference for token times is not stated explicitly. |
| Finalization | `{"type":"finalize"}` returns final tokens and a `<fin>` marker; call after ~200 ms of silence; not too often (MF). Endpoint detection returns `<end>` (EP). Empty text frame ends the stream, then `finished: true` (WS). |
| Latency settings | `enable_endpoint_detection`; `endpoint_latency_adjustment_level` 0-3; `endpoint_sensitivity` -1 to 1; `max_endpoint_delay_ms` 500-3000, default 2000, "no endpoint is emitted later than the selected maximum delay after speech has ended" (EP). |
| Server timing | `final_audio_proc_ms` and `total_audio_proc_ms` on every response (WS). The gap between them is how much audio is still non-final. |
| Limits | 300 min stream; 10 concurrent by default (LIM). |

### Gladia (live; `solaria-1` only)

Sources: [quickstart](https://docs.gladia.io/chapters/live-stt/quickstart.md) (QS), [partials](https://docs.gladia.io/chapters/live-stt/features/partial-transcripts.md) (PT), [endpointing](https://docs.gladia.io/chapters/live-stt/features/endpointing.md) (EP), [init](https://docs.gladia.io/api-reference/v2/live/init) (INIT), [recommended parameters](https://docs.gladia.io/chapters/live-stt/recommended-parameters) (REC).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `POST /v2/live`, then WebSocket; binary or base64 JSON chunks (QS). Pacing not documented. |
| Partials and finals | `receive_partial_transcripts`, default false (INIT). `is_final` on `transcript`; same `data.id` per utterance; partials use "a faster, smaller model" (PT). |
| Timestamps and clock | `utterance.start`, `end` in the examples; `created_at` ISO time on the message (QS). Units and reference not stated in the pages read. Word-level schema: the [transcript message page](https://docs.gladia.io/api-reference/v2/live/message/transcript) returned only a stub, so unconfirmed. |
| Finalization | `stop_recording`, then post-processing (QS). "Finalizing an utterance (`is_final: true`) is not the same as ending the live session" (QS). Per-utterance force not documented. |
| Latency settings | `endpointing` 0.01-10 s, default 0.05; `maximum_duration_without_endpointing` 5-60, default 5 (EP). "Neither setting guarantees that a final transcript arrives within that many seconds" (EP). Voice agents: 0.05-0.1 and 15 (REC). |
| Server timing | `created_at` only; meaning not defined. |
| Limits | 3 h session (QS). |

### Rev.ai (`machine`, `machine_v2`)

Sources: [requests](https://docs.rev.ai/api/streaming/requests/) (REQ), [responses](https://docs.rev.ai/api/streaming/responses/) (RESP), [overview](https://docs.rev.ai/api/streaming/) (OV).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Binary chunks of 250 ms or more; smaller "may experience increased transcription latency". "Audio will be processed faster than real-time and eventually partials will catch up" (REQ). |
| Partials and finals | `partial` and `final` hypotheses; `elements` is stated as present on finals (RESP). |
| Timestamps and clock | Final elements' timestamps are "relative to the start of the audio input" (RESP). Timestamps on partials need `detailed_partials`, at a ~1% WER cost on finals (REQ). `start_ts` offsets `ts` and `end_ts` (REQ). |
| Finalization | `"EOS"` text message returns a final and a close; any other casing closes with `1007` (REQ). No per-turn force found. |
| Latency settings | `priority` = `speed` (default) or `accuracy` (English and Spanish, `machine_v2` only); `max_segment_duration_seconds` 5-30 (REQ). |
| Server timing | Not documented. |
| Limits | 3 h per stream; 10 concurrent by default; billed on larger of audio and stream duration, minimum 15 s (OV). |

### OpenAI (`gpt-live-transcribe`, `gpt-transcribe`)

Sources: [realtime transcription](https://developers.openai.com/api/docs/guides/realtime-transcription) (RT), [VAD](https://developers.openai.com/api/docs/guides/realtime-vad) (VAD).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.openai.com/v1/realtime?intent=transcription`; base64 PCM in `input_audio_buffer.append` (RT). No rate rule; the Go example appends in a loop without sleeping (RT). |
| Partials and finals | `conversation.item.input_audio_transcription.delta` then `.completed`, keyed by `item_id`; completions across turns may arrive out of order (RT). Deltas are fragments, and the guide says UIs must decide how to revise earlier text (RT). |
| Timestamps and clock | `gpt-live-transcribe` "doesn't return word-level timestamps, speaker labels, or transcription confidence scores" (RT). For `gpt-transcribe` in a session: not documented. |
| Finalization | `input_audio_buffer.commit` at the end of each turn; `gpt-live-transcribe` must have `turn_detection` omitted or null, and the guide says to use client-side VAD (RT, VAD). |
| Latency settings | `audio.input.transcription.delay` = `minimal`, `low`, `medium`, `high`, `xhigh`. "The exact delay in milliseconds can vary by model configuration" (RT). The closest documented fit to the map's "named latency setting". |
| Server timing | Not documented on the pages read. |

### Gemini Live (`gemini-3.5-transcribe-live`)

Source: [live transcription](https://ai.google.dev/gemini-api/docs/live-api/live-transcribe) unless noted.

| Topic | Documented behaviour |
|---|---|
| Input and pacing | PCM16 16 kHz; "chunks of 100ms (1,024 to 2,048 frames)". `send_realtime_input` favours responsiveness over ordering ([Live guide](https://ai.google.dev/gemini-api/docs/live-guide)). No rate rule. |
| Partials and finals | `interim_input_transcription`, then `input_transcription`, "authoritative". `SMART` mode cleans the final; `VERBATIM` is default. |
| Timestamps and clock | "Word-level timestamps are not supported over the Live API." The page claims "utterance-level timestamps", but the response field table lists no timing field. |
| Finalization | Hybrid VAD: `audio_stream_end` is "an immediate turn finalization prompt". Manual VAD: `activity_start`, `activity_end`. |
| Latency settings | `silenceDurationMs`, `prefixPaddingMs` appear in the [Live guide](https://ai.google.dev/gemini-api/docs/live-guide) VAD config. |
| Server timing | `usageMetadata` token counts ([Live guide](https://ai.google.dev/gemini-api/docs/live-guide)). |
| Limits | 10 min session. |

### Azure Speech (Speech SDK real-time)

Sources: [recognize speech](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-recognize-speech) (HOW), [results](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/get-speech-recognition-results) (RES), [captioning](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/captioning-concepts) (CAP), [input streams](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/how-to-use-audio-input-streams) (STR). The repo's provider uses a different, HTTP transcription endpoint.

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Push or pull stream of PCM16 mono at 8 or 16 kHz (STR). Pacing not documented on the pages read. |
| Partials and finals | `Recognizing` (intermediate), then `Recognized` (final) (RES). "Punctuation of partial results isn't available" (CAP). |
| Timestamps and clock | Offset in 100 ns ticks from "the first audio byte processed by the SDK" (RES). Word offsets on `Recognized` only, after `RequestWordLevelTimestamps`; "not available while recognition is in progress" (RES). Offsets can differ between `Recognizing` and `Recognized` (RES). |
| Finalization | Utterance ends on silence (RES). Stop with `stop_continuous_recognition` (HOW). Per-utterance force not documented. |
| Latency settings | `Speech_SegmentationSilenceTimeoutMs` 100-5000, typically 500 (HOW). `SpeechServiceResponse_StablePartialResultThreshold`, which can increase latency (CAP). |
| Server timing | Not documented for SDK results. The Speech CLI output has a `result.latency` column, undefined on that page (RES). |

### Smallest Pulse (`smallest_provider.py`; not in the run scripts' model lists)

Sources: [endpointing](https://docs.smallest.ai/models/speech-to-text/features/endpointing.md) (EP), [response format](https://docs.smallest.ai/models/speech-to-text/realtime-web-socket/response-format.md) (RF), [best practices](https://docs.smallest.ai/models/speech-to-text/realtime-web-socket/best-practices.md) (BP), [word timestamps](https://docs.smallest.ai/waves/documentation/speech-to-text-pulse/features/word-timestamps.md) (WT), [keep-alive](https://docs.smallest.ai/models/speech-to-text/features/keep-alive.md) (KA).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.smallest.ai/waves/v1/stt/live?model=pulse` (EP). 4096-byte chunks every 50-100 ms; avoid under 20 ms or over 200 ms (BP). At 16 kHz PCM16 a 4096-byte chunk is 128 ms, so that guidance is faster than 1x (my arithmetic). |
| Partials and finals | `is_final`, `is_last`, `from_finalize`. Interim per chunk; finals fire on silence, word count, EOU timer or client message (RF, EP). During long speech, more than one final can arrive before `finalize` (EP). |
| Timestamps and clock | `word_timestamps=true` returns `words` with `start`, `end`, seconds in the examples; zero reference not stated (WT, RF). Also `sentence_timestamps` (RF). |
| Finalization | `{"type":"finalize"}` keeps the socket open and emits exactly one `from_finalize` final; `{"type":"close_stream"}` ends with `is_last` (RF, EP). |
| Latency settings | `endpointing`, `eou_timeout_ms` 100-10000 (600 with endpointing on, 800 off), `finalize_on_words`, `max_words`. `pulse-2` uses built-in detection with "no tuning knobs" (EP). |
| Server timing | None in the streaming response. Vendor formula subtracts network, client and finalize RTT from EOT ([measuring latency](https://docs.smallest.ai/models/speech-to-text/benchmarks/measuring-latency.md)). |
| Limits | 20 min inactivity by default; billed on audio duration (KA). |

### Reson8

Source: [full docs](https://docs.reson8.dev/llms-full.txt) (sections Realtime API, Realtime guide, Turns guide, Audio Formats).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.reson8.dev/v1/speech-to-text/realtime`; binary, any chunk size. Rate rule not documented; the Python example sends file chunks without sleeping. Not reading messages for 10 s closes with `4000`. |
| Partials and finals | Finals only by default; `include_interim=true` adds interims and `is_final`. |
| Timestamps and clock | `include_timestamps` gives `start_ms`, `duration_ms`; `include_words` gives per-word detail. Zero reference not stated. |
| Finalization | `flush_request` returns a final, then `flush_confirmation`. Then close the socket. No separate end-of-stream message. |
| Latency settings | None on `realtime`. The separate `turns` endpoint has `eager_turn_probability`, `final_turn_probability`, and `turn_end_candidate` then `turn_end`. |
| Server timing | Not documented. |

### Modulate Velma-2

Sources: [multilingual streaming](https://docs.modulate.ai/api-reference/stt/streaming.md) (ML), [English streaming](https://docs.modulate.ai/api-reference/stt/streaming-vfast.md) (EN).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | Binary frames of any size (ML, EN). Rate rule not documented; the ML Node sample pipes a file straight to the socket. |
| Partials and finals | ML: `partial_utterance` with `partial_results=true`, then `utterance`; they interleave across utterances, so correlate by `utterance_uuid`. EN: partial about every 1.5 s; a single final at end of stream unless `endpointing=true`. |
| Timestamps and clock | `start_ms` is "from the beginning of the stream", with `duration_ms` (ML). EN `time_stamps=true` gives per-word timing on finals only, "approximate", and can be missing without a marker. |
| Finalization | Empty text frame, then `done` (ML, EN). No per-turn force. |
| Latency settings | EN `endpointing` on or off only; no pause threshold documented. |
| Server timing | `done.duration_ms`, total audio only. |

### Zoom Scribe Live

Sources: [Live mode](https://developers.zoom.us/docs/ai-services/scribe/live-mode/) (LM), [API reference](https://developers.zoom.us/docs/api/ai-services/) (API; only the first ~100,000 characters read).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.zoom.us/v2/aiservices/scribe/live`. PCM16 16 kHz mono; "approximately 100 ms per binary frame" (LM). No rate rule. |
| Partials and finals | `config.interim_results` enables `transcription.delta`, whose `delta` is "the accumulated partial text", despite the name. `transcription.completed` is the final (LM). |
| Timestamps and clock | `audio_start_ms`, `audio_end_ms` per item; reference not stated (LM, API). |
| Finalization | `session.close`, read until `session.closed` (LM). Per-turn force not documented. |
| Latency settings | Not documented. API mentions voice activity detection config with no fields shown. |
| Server timing | `transcription_latency_ms` on completed items, undefined (LM). |
| Limits | 120 min session; 30 s idle without audio (LM). |

### Sprag

Source: [realtime transcription](https://sprag.ai/docs/stt/api/realtime).

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.sprag.ai/v1/audio/transcriptions`; base64 PCM16 in `input_audio_buffer.append`. Pacing not documented. |
| Partials and finals | `.delta`, then `.completed` (OpenAI-style). Page says transcripts arrive after `speech_stopped`, "not word by word". |
| Timestamps and clock | Word timings not available in realtime. |
| Finalization | `input_audio_buffer.commit` with `turn_detection: null`, minimum 100 ms. |
| Latency settings | `silence_duration_ms` default 500, `threshold`, `prefix_padding_ms`. |
| Server timing | `usage` audio duration on `completed`. |
| Limits | 1 h session. |

### PyAI Hear (PR 166, not on the leaderboard)

Source: [streaming guide](https://docs.pyai.com/guides/streaming-stt.md) (states "Last verified 2026-06-16").

| Topic | Documented behaviour |
|---|---|
| Input and pacing | `wss://api.pyai.com/v1/audio/transcriptions/stream`. Binary PCM16 at 8 or 16 kHz, ~20 ms frames. No faster-than-real-time rule documented; [limits page](https://docs.pyai.com/errors-and-limits) gives only a per-plan concurrency cap. |
| Partials and finals | `partial`, `partial_stable`, then `speech_final`, then `final` (a "corrected full-context transcript"). Key by `utterance_id`; don't append both. |
| Timestamps and clock | `t_ms` is "the audio position of the hypothesis"; `audio_ms` is active-speech length. Word timings not documented on the stream (batch jobs have them). |
| Finalization | `{"type":"commit"}`; closing also flushes. Timer advances only on received frames, so silence must be sent. |
| Latency settings | `endpointing_ms`, settable mid-session and acknowledged by `config_ack`. An utterance "always completes within `max(endpointing_ms, 1500 ms)` of streamed silence". |
| Server timing | `t_ms`, `audio_ms`, `endpoint_reason`, `usage`. |

## Gaps and things I could not verify

- **Nothing was tested.** No keys. Every behaviour above, including close codes and throttles, is the vendor's claim.
- **Summaries, not always raw text.** Meta, AssemblyAI, Soniox, OpenAI, PyAI, Reson8, Modulate, Gemini Live and some Speechmatics, Gladia and Smallest pages were read as raw Markdown. The Speechmatics API reference (partly), ElevenLabs API reference, Rev.ai, Zoom, Sprag and Gladia init were read through a page-to-Markdown summariser, so exact wording from those deserves a second look before being quoted upstream.
- **Clock references** are stated by AssemblyAI, Azure, Modulate, Rev.ai (final elements), Meta and PyAI. For ElevenLabs, Soniox, Gladia, Smallest, Reson8, Speechmatics and Zoom the zero point isn't stated on the pages read, though audio-relative is the obvious reading.
- **Gladia word timestamps:** unconfirmed; schema page was a stub.
- **Azure pacing:** not found for the cloud service. A [container FAQ](https://learn.microsoft.com/en-us/azure/ai-services/speech-service/speech-container-faq) surfaced in search as mentioning 2x SDK throttling for file input, but I saw only a search snippet, and it concerns containers.
- **Meta handshake:** the guide refers to nine fields; the schema page was searched for timing fields rather than read in full for a latency field.
- **Zoom API reference:** the last ~25,000 characters were not read, so a turn-detection or latency field could exist there. A search-engine summary mentioned turn-detection thresholds and partials being "in preview", but neither appeared in the pages I read, so I don't rely on them.
- **Providers with no streaming information I could use:**
  - Fonada: its [llms.txt](https://fonadalabs.ai/llms.txt) says "WebSocket: Available for real-time streaming" but the [STT page](https://fonadalabs.ai/docs/speech-to-text) documents only REST, so not documented.
  - Aqua: documented as batch-only: "Avalon is batch-only" ([API reference](https://aquavoice.com/docs/api)).
  - p99lab: the [model card](https://huggingface.co/p99lab/audrey-0.7) gives `wss://api.p99lab.com/v1/stream`, 16 kHz PCM, partials and per-turn finals, and a vendor claim of ~150 ms to final; `p99lab.com/docs` was not read, so the rest is not documented here.
  - Sophea and OtterAI: no first-party streaming documentation found.
- **Not covered:** Deepgram and others not in `api/providers/`.
