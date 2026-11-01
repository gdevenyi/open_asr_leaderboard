# Which English public datasets can support word-level latency measurement?

Research for [issue #7](https://github.com/gdevenyi/open_asr_leaderboard/issues/7), part of the streaming ASR track map ([issue #3](https://github.com/gdevenyi/open_asr_leaderboard/issues/3)). Data gathered 2026-11-01.

## Short answer

- **No set on the Hub carries word timestamps.** The only timing on the Hub is the chunk boundaries in Earnings22-Cleaned-AA-chunked and the segment span inside AMI clip ids.
- **Upstream word timing exists for four sets, with limits:**
  - AMI: complete in the 16 word files checked, and it joins to our clips.
  - LibriSpeech: third-party Montreal Forced Aligner (MFA) alignments.
  - Earnings-22: Rev's force-aligned references, but only 72% of tokens have times, and 2 of the 6 AA calls are at 1% and 21%.
  - CORAAL: MFA alignments for 4 of 8 components, from an older corpus version.
- **The rest need forced alignment:** Earnings-21, SPGISpeech, GigaSpeech, VoxPopuli and URGENT2024. MFA is the most accurate option in the one head-to-head comparison found. NeMo Forced Aligner (NFA) is the practical choice for hour-long files.
- **Finalization latency may not need word timing.** Artificial Analysis measures it from a Silero VAD end of speech, not from word times. That is a question for #9.
- **Size and cost:** the sets the scripts run total **481.7 hours**. A 1x real-clock run costs **$2,409 at 1 stream, $308 at 8 and $97 at 32**, per model entry. CORAAL and Earnings make up 320 of those hours.

Labels used below: **M** is measured, meaning read directly from a Hub column, the viewer API or a file. **D** is derived, meaning computed from measured values. The method is stated each time.

## Scope: what the scripts actually run

| Family | Configs | Source |
| --- | --- | --- |
| Short-form | `ami_cleaned`, `gigaspeech_cleaned`, `voxpopuli_cleaned_aa`, `librispeech` (`test.clean`, `test.other`), `spgispeech`, `urgent2024`, `urgent2024_clean`, `earnings22_cleaned_aa_chunked` | `transformers/submit_jobs_nemotron.sh` |
| Long-form | `earnings21`, `earnings22`, CORAAL `ATL DCA DCB DTA LES PRV ROC VLD` | `transformers/submit_jobs_whisper_longform.sh`, `normalizer/eval_utils.py` |

Excluded from the totals:

- **Hub configs the scripts above do not run:** `ami`, `gigaspeech`, `voxpopuli`, `earnings22` (short), `common_voice`, `tedlium`, and long-form `tedlium`.
- **Earnings22-Cleaned-AA (parent):** it holds the same audio as the chunked set, and its rows carry only the reference transcript and metadata. Counting both would double-count.
- **Monsoon:** `VoiceArena/Monsoon_en_IN_test` is in the short-form script but outside this ticket. The Hub API redirects it to `VoiceArena/MonsoonASR-Open-ASR-leaderboard-en-IN`, and the viewer rejects the old name. It has 2,102 clips and 5.62 h (D, viewer mean × n), which is $28.12 at 1 stream.

## Hours and clip lengths

Short-form values are exact, summed from each set's own length column with column-pruned parquet reads, so no audio was fetched.

| Set | Clips | Hours | Min s | p5 | Median | p95 | Max s |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| ami_cleaned | 7,805 | 8.016 | 1.00 | 1.12 | 2.88 | 8.96 | 26.17 |
| gigaspeech_cleaned | 18,768 | 35.147 | 1.00 | 1.54 | 6.51 | 13.35 | 22.02 |
| voxpopuli_cleaned_aa | 628 | 1.978 | 5.02 | 5.51 | 9.94 | 21.32 | 38.44 |
| librispeech test.clean | 2,620 | 5.403 | 1.28 | 2.31 | 5.79 | 17.84 | 34.95 |
| librispeech test.other | 2,939 | 5.342 | 1.25 | 2.28 | 5.15 | 15.76 | 34.51 |
| spgispeech | 39,341 | 100.002 | 5.01 | 5.40 | 8.85 | 13.98 | 15.00 |
| urgent2024 | 1,000 | 1.925 | 2.51 | 3.22 | 6.95 | 10.26 | 14.73 |
| urgent2024_clean | 1,000 | 1.925 | 2.51 | 3.22 | 6.95 | 10.26 | 14.73 |
| Earnings22-Cleaned-AA-chunked | 341 | 1.917 | 2.32 | 4.25 | 24.42 | 29.44 | 29.98 |
| **Short-form total** | **74,442** | **161.66** | | | | | |

The chunked set's chunk duration histogram (M, viewer) is bimodal: 79 of 341 chunks are under 10.6 s and 211 are over 21.7 s. The chunker combines speech segments up to a 30 s limit, so most chunks land near that limit.

| Long-form set | Files | Hours | Basis | Min s | Median | Max s |
| --- | ---: | ---: | --- | ---: | ---: | ---: |
| earnings21 | 44 | 39.263 | M, `audio_length` column | 1,097 | 3,281 | 5,741 |
| earnings22 | 125 | 119.886 | D, audio bytes ÷ 32,000 | 875 | 3,552 | 7,407 |
| CORAAL ATL | 14 | 8.621 | D, audio bytes ÷ 88,200 | 997 | 2,393 | 2,887 |
| CORAAL DCA | 74 | 34.042 | D | 227 | 1,659 | 2,997 |
| CORAAL DCB | 63 | 46.059 | D | 273 | 3,054 | 4,912 |
| CORAAL DTA | 40 | 25.144 | D | 1,338 | 2,356 | 2,894 |
| CORAAL LES | 15 | 8.444 | D, agrees with viewer | 58 | 2,229 | 3,157 |
| CORAAL PRV | 32 | 13.961 | D | 123 | 1,490 | 2,760 |
| CORAAL ROC | 19 | 13.186 | D, agrees with viewer | 98 | 3,024 | 4,679 |
| CORAAL VLD | 14 | 11.483 | D, agrees with viewer | 297 | 3,041 | 4,002 |
| **CORAAL total** | **271** | **160.94** | | | | |
| **Long-form total** | **440** | **320.09** | | | | |

**All sets: 74,882 clips, 481.74 hours.**

### Why some long-form hours are derived

The viewer does not give usable totals for these sets, and the obvious fallbacks are wrong:

- **Earnings-22:** viewer statistics are `partial: true` and cover only 45 of 125 rows, and the rows have no file id.
  - The audio is stored as 16 kHz 16-bit PCM. The audio bytes in the first 9 parquet files, divided by the viewer's duration for those 45 rows, give 32,000.02 B/s. Extending that to all 125 rows gives 119.886 h. This assumes the other 80 rows use the same encoding.
  - Rev's `metadata.csv` sums to 118.895 h, but it understates three files. In the force-aligned references, the last word ends past the stated length by 2,690 s (4483733), 709 s (4482110) and 136 s (4471606). Adding those brings the sum to 119.877 h, within 0.01 h of the byte figure.
  - The median and max in the table come from the uncorrected metadata, so they are approximate.
- **CORAAL:** statistics fail outright for ATL, DCB, DTA and PRV, and are partial for DCA.
  - The `CORAAL.Length.of.Transcript` column sums to only 153.41 h because 11 rows are `NA` or null.
  - Bytes ÷ 88,200 (44.1 kHz, 16-bit, mono) matches the viewer for all three configs with full statistics, and the User Guide (pp. 28-35) for all eight components, within 0.1 h each (160.7 h total).
  - The length columns in the table come from the metadata and exclude the 11 missing rows.
- **Earnings-21:** the `hf-audio` card says 35 h. The measured 39.26 h matches Rev's README ("39-hour") and metadata exactly, so the card is wrong.
- **Earnings22-Cleaned-AA:** the parent `duration` field is rounded and sums to 6,895 s, and the chunks sum to 6,900.9 s. For 4483046 the parent says 1,187 s, while its chunks run to 1,193 s and the last aligned word ends at 1,191 s. Use the chunk figure.

**Sampling trap:** the `hf-audio/open-asr-leaderboard` card says each split is sorted by audio length, longest first. A "first N rows" subset would therefore take only the longest clips. That matters for #13.

## Timestamps

| Set | On the Hub | Upstream word timing | Status for word latency |
| --- | --- | --- | --- |
| ami_cleaned | Segment span in `id` | Yes: AMI manual annotations | **Usable** |
| librispeech | None | Third-party MFA | Usable, unofficial |
| Earnings-22 | None | Rev force-aligned, partial | Partial |
| Earnings22-Cleaned-AA-chunked | Chunk start and end, no per-chunk text | Rev force-aligned, partial and for different references | Weak: 2 of 6 calls are at 1% and 21% of tokens |
| CORAAL | None | Utterance level, plus MFA for DCA, DCB, PRV, ROC | Partial |
| Earnings-21 | None | None | Needs alignment |
| spgispeech | None | None released | Needs alignment |
| gigaspeech_cleaned | None | Segment level only, gated | Needs alignment |
| voxpopuli_cleaned_aa | None | None | Needs alignment |
| urgent2024, urgent2024_clean | None | None found | Needs alignment; provenance unverified |

Details by set:

- **AMI.**
  - Clip ids encode the segment span in centiseconds, e.g. `AMI_IS1009b_H03_FIO089_0023485_0026102`. Across 63 rows, the id-derived length matched `audio_length_s` within 0.1 ms (M).
  - The annotation archive has 687 word files covering 171 meetings. Every `<w>` had `starttime` and `endtime` in the 16 files checked, from 4 meetings (M).
  - I joined that one clip by hand: `H03` is channel 3, agent D, speaker FIO089 in `meetings.xml`, and agent D's words in that span match the clip text. This was one clip, so a full join still needs testing.
  - These timings are automatic: AMI states they "were achieved through forced alignment".
- **LibriSpeech.** The official release has no word timing. `CorentinJ/librispeech-alignments` lists `test-clean` and `test-other`, made with MFA. Across the whole corpus it reports 46 unaligned utterances in the TextGrids and 127 in the simple format. Clip ids such as `121-127105-0007` look like native LibriSpeech ids, so joining should be direct, but I did not test it.
- **Earnings-22.**
  - Plain `nlp_references` have no times at all: 0 of 1,008,545 tokens (M).
  - `force_aligned_nlp_references` has 740,632 of 1,029,132 tokens timed, or 72% (M). Coverage per file has a median of 90%, with 61 of 125 files at 90% or above and 30 below 50%.
  - The Hub rows have no file id to join on, so applying these times needs a match on text or audio. I did not compare Rev's references with the text in the leaderboard's long-form set.
- **Earnings22-Cleaned-AA.** The table below shows why upstream times do not transfer cleanly. AA also corrected the transcripts by hand, so tokens differ and would need re-mapping. Chunks tile each call with zero gaps (M), and a chunk end is not an end of speech.

| AA call | AA length s | Rev length s | Rev tokens timed |
| --- | ---: | ---: | ---: |
| 4483046 | 1,187 | 1,187 | 99.8% |
| 4474955 | 875 | 875 | 99.5% |
| 4471586 | 1,259 | 1,259 | 97.2% |
| 4468919_trimmed | 985 | 1,214 | 94.4% |
| 4475604 | 1,245 | 1,245 | 21.3% |
| 4482968 | 1,344 | 1,344 | 1.1% |

  For the trimmed call, the AA and Rev transcripts begin the same way, so the cut appears to be at the end. That is my inference, not something AA documents.

- **CORAAL.**
  - Upstream transcripts have utterance-level time alignment in TextGrid, ELAN and tab-delimited text. `bezzam/coraal` concatenates them into a single `text` field with no times, per its preparation scripts, and has no timing column.
  - The User Guide (p. 27) describes phone- and word-level MFA alignments from v2018.10.06. The aligned-data page lists DCA, DCB, PRV and ROC and calls them phone-level. I did not open the archives to confirm the word tier.
  - ATL, DTA, LES and VLD have no alignment. In ROC, two speakers added in 2020.05 are not included.
- **Earnings-21.** All 362,351 tokens in `nlp_references` have empty `ts` (M), and there is no force-aligned folder. The repo's `rttms` folder is by format speaker turns, and `output` holds system hypotheses, so neither gives reference word times.
- **SPGISpeech.** Kensho says the clips were cut with a forced aligner and VAD, but the fields are only filename, size and transcript.
- **GigaSpeech.** `GigaSpeech.json` has segment `begin_time` and `end_time`. The README does not mention word alignments, and the raw release requires a form. The Hub ids (`YOU1000000134_S0000258`) look like they point into it, which I did not test.
- **VoxPopuli.** The source card describes force-aligning paragraphs and splitting them into utterances, but neither its fields nor the AA cleaned card include timing.
- **URGENT2024.** I could not confirm how the `hf-audio` configs relate to the URGENT 2024 challenge. The challenge paper describes a blind test of 1,000 clips, half with no clean reference. Here all 1,000 have a clean counterpart with matching ids and lengths, so the two descriptions do not fully match.

## Forced alignment options

| Tool | Evidence | Fit |
| --- | --- | --- |
| **MFA** | Most accurate in Rousso et al. 2024 (below). Already used by CORAAL and for LibriSpeech. | Default for utterance-length clips. Needs a pronunciation dictionary, so out-of-vocabulary words such as company names need handling. |
| **NFA** | NVIDIA docs: word, token and segment CTM output, and it aligns recordings of an hour or more. CTC or hybrid models only. | Best documented fit for Earnings and CORAAL files. No independent accuracy figure found. |
| **WhisperX** | Scored below MFA. Its README says words with characters such as "2014." get no timing. It does not document aligning an external transcript. | Weak fit: earnings text is full of digits, and references must be aligned verbatim. |
| **torchaudio `forced_align`** | Its tutorial warns the API is "deprecated in 2.8 and will be removed in 2.9". | Avoid for a new harness. |

Word boundary accuracy from [Rousso et al., Interspeech 2024](https://ar5iv.labs.arxiv.org/html/2406.19363), percent within tolerance:

| System | TIMIT 25 ms | TIMIT 50 ms | TIMIT 100 ms | Buckeye 25 ms | Buckeye 50 ms | Buckeye 100 ms |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| MFA | 72.8 | 89.4 | 97.4 | 69.9 | 84.9 | 91.8 |
| MMS | 43.5 | 75.7 | 94.7 | 52.7 | 75.0 | 87.9 |
| WhisperX | 52.7 | 82.4 | 94.2 | 43.1 | 67.4 | 77.4 |

Two limits on using these figures:

- The comparison covers only words that WhisperX and MMS recognized correctly, so it does not cover all reference words.
- Buckeye (conversational) is the closer match to our data, and even MFA misses 100 ms on about 8% of words there. Reported per-word latency differences below about 100 ms would sit within alignment error. No figures were found for accented earnings audio or long-file alignment.

## Trailing silence

Measured on a small sample with Silero VAD, at probability 0.5 with a run of at least 3 frames of 32 ms. Silence is the time from the end of the last speech run to the end of the audio. Short-form samples use 30 clips each, at offsets spaced evenly across the split, which also spreads them across lengths because the splits are sorted.

| Set | Sample | Median s | p90 s | Max s | Over 1 s |
| --- | ---: | ---: | ---: | ---: | ---: |
| ami_cleaned | 30 (1 no speech detected) | 0.18 | 1.21 | 1.78 | 17% |
| gigaspeech_cleaned | 30 (1 no speech detected) | 0.03 | 0.29 | 0.70 | 0% |
| voxpopuli_cleaned_aa | 30 | 0.02 | 0.06 | 0.18 | 0% |
| librispeech test.clean | 30 | 0.33 | 0.46 | 0.49 | 0% |
| librispeech test.other | 30 | 0.27 | 0.46 | 0.47 | 0% |
| spgispeech | 30 | 0.29 | 0.56 | 0.73 | 0% |
| urgent2024 | 30 | 0.06 | 0.88 | 6.35 | 10% |
| urgent2024_clean | 30 | 0.12 | 0.89 | 1.44 | 10% |
| Earnings22-Cleaned-AA-chunked | 30 chunks | 0.80 | 2.30 | 3.52 | 30% |
| Earnings22-Cleaned-AA parents | 6, last 120 s | 3.23 | 9.04 | 12.03 | 83% |
| Earnings-21 upstream MP3 | 8, last 120 s | 1.63 | 13.45 | 38.62 | 63% |
| Earnings-22 upstream MP3 | 8, last 120 s | 7.71 | 18.32 | 23.07 | 75% |
| CORAAL | 23, last 120 s | 0.38 | 2.03 | 3.81 | 30% |

How far to trust these:

- All are small-sample indications, not set-level statistics.
- **Earnings-21 and -22** were sampled from Rev's MP3s, because the viewer refuses `hf-audio` rows over its 300 MB scan limit. I assume the recordings are the same. A 30 dB energy gate gives a much lower Earnings-22 median (1.8 s vs 7.7 s), so these tails contain non-speech sound and are not plain silence.
- **CORAAL** is opportunistic: the viewer refused 20 of 43 rows tried, so large row groups are under-represented.
- **Noisy URGENT:** the 6.35 s maximum is probably VAD trouble at low SNR. The clean counterpart's maximum is 1.44 s, on the same offsets.

**Consequence for #9:** VoxPopuli and GigaSpeech clips end almost at the last word, and SPGISpeech and LibriSpeech within about 0.3 s. A model cannot show it finalized after end of speech if the stream closes right there, so the harness may need to append silence or send an explicit end signal. Earnings chunks are the opposite: chunks tile the call with no gaps, so a chunk end typically falls inside a pause and need not be an end of turn.

## H200 cost of a 1x real-clock run

Model:

- Rate is $5.00/hour (`README.md`, `h200`).
- **Ideal** cost is hours ÷ C × $5, for C concurrent streams.
- **Packed** cost is what the per-config jobs would actually take. It uses a greedy longest-first schedule within each config, one job per config as the submit scripts do. A job can't finish before its longest file, and sets with few files can't use all C streams.

Assumptions:

- C = 8 and C = 32 are illustrative, not measured capacity. Real stream capacity is unknown, and it varies by model.
- Cost is per model entry. With up to 3 latency settings per model, a model can cost up to 3×.
- Clips start back to back. There is no time for model loading, data download, job startup or retries, so these figures are lower bounds. For scale, the `README.md` example costs of $2.92-$5.58 are full batch runs over short-form English, which run far faster than real time.
- Earnings-22 and CORAAL packing use approximate per-file lengths, scaled to the derived totals.
- All clips are included. `is_target_text_in_range` drops empty or ignored references at load time, so true hours may be slightly lower. I did not count those.

| Set | Hours | 1 stream | 8 ideal | 8 packed | 32 ideal | 32 packed |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| ami_cleaned | 8.016 | $40.08 | $5.01 | $5.01 | $1.25 | $1.25 |
| gigaspeech_cleaned | 35.147 | $175.73 | $21.97 | $21.97 | $5.49 | $5.49 |
| voxpopuli_cleaned_aa | 1.978 | $9.89 | $1.24 | $1.24 | $0.31 | $0.31 |
| librispeech test.clean | 5.403 | $27.02 | $3.38 | $3.38 | $0.84 | $0.84 |
| librispeech test.other | 5.342 | $26.71 | $3.34 | $3.34 | $0.83 | $0.84 |
| spgispeech | 100.002 | $500.01 | $62.50 | $62.50 | $15.63 | $15.63 |
| urgent2024 | 1.925 | $9.62 | $1.20 | $1.20 | $0.30 | $0.30 |
| urgent2024_clean | 1.925 | $9.62 | $1.20 | $1.20 | $0.30 | $0.30 |
| Earnings22-Cleaned-AA-chunked | 1.917 | $9.58 | $1.20 | $1.20 | $0.30 | $0.30 |
| **Short-form** | **161.66** | **$808.26** | **$101.04** | **$101.04** | **$25.25** | **$25.26** |
| earnings21 | 39.263 | $196.32 | $24.54 | $25.55 | $6.13 | $7.97 |
| earnings22 | 119.886 | $599.43 | $74.93 | $75.59 | $18.73 | $19.58 |
| CORAAL (8 configs) | 160.94 | $804.69 | $100.61 | $105.39 | $25.15 | $43.97 |
| **Long-form** | **320.09** | **$1,600.44** | **$200.08** | **$206.53** | **$50.01** | **$71.52** |
| **Total** | **481.74** | **$2,408.70** | **$301.12** | **$307.57** | **$75.26** | **$96.78** |

CORAAL per config at 1 stream / 8 packed / 32 packed: ATL $43.10 / $6.52 / $4.01, DCA $170.21 / $21.73 / $6.09, DCB $230.30 / $29.00 / $8.45, DTA $125.72 / $15.87 / $5.14, LES $42.22 / $5.83 / $4.39, PRV $69.80 / $9.03 / $3.83, ROC $65.93 / $8.71 / $6.50, VLD $57.41 / $8.70 / $5.56.

Wall clock for the whole set is 481.7 h at 1 stream. At 8 and 32 packed, the jobs total 61.5 h and 19.4 h if they run one after another. Running them in parallel on separate H200s shortens the wall clock but costs the same.

Things that move these figures:

- **Added silence:** appending silence is charged per clip, and there are 74,882 clips, nearly all short-form. Appending 2 s to every clip adds 41.6 h, or **$208 at 1 stream** and about $26 at 8. Two seconds is only an example, taken from the fallback window AA uses.
- **SPGISpeech alone is 21% of the hours** (100 h, $500 at 1 stream). Its card also restricts it to academic research and internal use, and bars redistribution without consent.
- **Pilot budget:** $10 buys any one of these at 1 stream: Earnings22-Cleaned-AA-chunked ($9.58), VoxPopuli ($9.89), URGENT ($9.62). At 8 streams it also covers `ami_cleaned` ($5.01), if the model can sustain 8.

## Inputs for the blocked tickets

These are suggestions, not decisions.

- **#8 (pacing):** at 1 stream, real pacing of everything costs $2,409 per entry, so affordability depends on stream capacity or on a subset.
- **#9 (finalization latency):** [AA's methodology](https://artificialanalysis.ai/speech-to-text/methodology) streams 20 ms chunks and times the final transcript from Silero VAD end of speech. For models that support forced endpointing, it starts the timer when the audio through that point has been sent. For others it uses a 2 s fallback. That needs no word timing, and it would work on every set here. Word timing is needed only for per-word metrics, which are limited by the alignment error above.
- **#13 (v1 sets):** the cheapest way to get word timing with existing upstream data is `ami_cleaned` (8 h, conversational, timing joins) plus LibriSpeech (10.7 h). Together they cost about $94 at 1 stream. Earnings22-Cleaned-AA-chunked is already [used by AA for streaming](https://artificialanalysis.ai/speech-to-text/methodology), but word timing there means running an aligner on 2 of the 6 calls at least. Any subset must not take the first N rows.

## Not verified

- CORAAL MFA archives were not opened, so the word tier and its match to the HF text are unconfirmed. The aligned files come from v2018.10.06, while the Hub set has later version labels.
- The AMI join was tested by hand on one clip, and `ami_cleaned` was not traced to how it was cleaned.
- LibriSpeech and GigaSpeech id joins were not tested.
- How `hf-audio` Earnings-22 row text relates to Rev's references, and which row is which file.
- URGENT2024 provenance.
- Aligner accuracy on these specific sets. No aligner was run.
- Licence compatibility for a public track was not reviewed beyond the notes above.

## Sources

Hub and viewer:

- [hf-audio/open-asr-leaderboard](https://huggingface.co/datasets/hf-audio/open-asr-leaderboard)
- [hf-audio/asr-leaderboard-longform](https://huggingface.co/datasets/hf-audio/asr-leaderboard-longform)
- [bezzam/coraal](https://huggingface.co/datasets/bezzam/coraal)
- [ArtificialAnalysis/Earnings22-Cleaned-AA](https://huggingface.co/datasets/ArtificialAnalysis/Earnings22-Cleaned-AA)
- [ArtificialAnalysis/Earnings22-Cleaned-AA-chunked](https://huggingface.co/datasets/ArtificialAnalysis/Earnings22-Cleaned-AA-chunked)
- [ArtificialAnalysis/VoxPopuli-Cleaned-AA](https://huggingface.co/datasets/ArtificialAnalysis/VoxPopuli-Cleaned-AA)
- [VoiceArena/MonsoonASR-Open-ASR-leaderboard-en-IN](https://huggingface.co/datasets/VoiceArena/MonsoonASR-Open-ASR-leaderboard-en-IN)
- [Dataset viewer API](https://datasets-server.huggingface.co/) (`statistics`, `parquet`, `rows`)

Upstream data:

- [Rev Earnings-21](https://github.com/revdotcom/speech-datasets/tree/main/earnings21) and [Rev Earnings-22](https://github.com/revdotcom/speech-datasets/tree/main/earnings22)
- [AMI transcription notes](https://groups.inf.ed.ac.uk/ami/corpus/transcription.shtml) and [AMI manual annotations v1.6.2](https://groups.inf.ed.ac.uk/ami/AMICorpusAnnotations/ami_public_manual_1.6.2.zip)
- [edinburghcstr/ami](https://huggingface.co/datasets/edinburghcstr/ami)
- [CORAAL](https://oraal.github.io/coraal), [CORAAL User Guide, June 2023](http://lingtools.uoregon.edu/coraal/userguide/CORAALUserGuide_current.pdf), [MFA-aligned CORAAL](http://lingtools.uoregon.edu/coraal/aligned/) and [prepare_coraal](https://github.com/ebezzam/prepare_coraal)
- [kensho/spgispeech](https://huggingface.co/datasets/kensho/spgispeech)
- [SpeechColab/GigaSpeech](https://github.com/SpeechColab/GigaSpeech)
- [facebook/voxpopuli](https://huggingface.co/datasets/facebook/voxpopuli)
- [Lessons Learned from the URGENT 2024 Speech Enhancement Challenge](https://arxiv.org/pdf/2506.01611) (from search-result text only)

Aligners and methodology:

- [Rousso et al. 2024, arXiv 2406.19363](https://ar5iv.labs.arxiv.org/html/2406.19363)
- [CorentinJ/librispeech-alignments](https://github.com/CorentinJ/librispeech-alignments)
- [Montreal Forced Aligner user guide](https://montreal-forced-aligner.readthedocs.io/en/latest/user_guide/index.html)
- [NeMo Forced Aligner](https://docs.nvidia.com/nemo-framework/user-guide/latest/nemotoolkit/tools/nemo_forced_aligner.html)
- [WhisperX](https://github.com/m-bain/whisperX)
- [torchaudio forced alignment tutorial](https://docs.pytorch.org/audio/stable/tutorials/forced_alignment_tutorial.html)
- [Silero VAD](https://github.com/snakers4/silero-vad)
- [Artificial Analysis speech-to-text methodology](https://artificialanalysis.ai/speech-to-text/methodology)

Repo: `README.md`, `transformers/submit_jobs_nemotron.sh`, `transformers/submit_jobs_whisper_longform.sh`, `normalizer/data_utils.py`, `normalizer/eval_utils.py`.
