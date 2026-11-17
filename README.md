# PodcastCleaner

CLI tool for downloading and cleaning podcast audio. Downloads from YouTube (playlists or single videos) or takes local files, separates speech from background noise, removes residual noise, normalizes loudness, and exports in distribution formats. All processing runs locally.

## Pipeline

Audio flows through 7 stages. Each stage writes to its own subdirectory and is independently resumable via `.done` marker files.

```
download → preprocess → separate → denoise → transcribe → normalize → export
```

| Stage | What it does | Model/Tool |
|---|---|---|
| **download** | Grab audio from YouTube | yt-dlp |
| **preprocess** | Resample to 48kHz mono float32 | torchaudio, soundfile |
| **separate** | Isolate vocals from background | Demucs htdemucs_ft |
| **denoise** | Remove residual noise from vocals | DeepFilterNet3 |
| **transcribe** | Word-level transcription (optional) | WhisperX large-v3 |
| **normalize** | Loudness normalize to -16 LUFS | pyloudnorm |
| **export** | Convert to MP3/FLAC | ffmpeg |

## Quick Start

### Automated Setup

```bash
git clone https://github.com/TristanPetersDS/podcast-cleaner.git && cd podcast-cleaner
./setup.sh
```

The setup script detects Python 3.10+, creates a virtual environment, installs all dependencies, copies the example config, and runs a system check.

### Manual Setup

```bash
git clone https://github.com/TristanPetersDS/podcast-cleaner.git && cd podcast-cleaner
python -m venv .venv && source .venv/bin/activate
pip install -e '.[dev]'

# Optional: install transcription support
pip install -e '.[transcribe]'
```

### Docker (GPU-accelerated)

```bash
cp config.example.yaml config.yaml   # then edit
docker compose build
docker compose run --rm podcast-cleaner run --url "..."

# Or without Compose
docker build -t podcast-cleaner .
docker run --gpus all -v ./output:/app/output -v ./config.yaml:/app/config.yaml podcast-cleaner run --url "..."
```

The image's default command prints `--help`, so pass the subcommand you want; `docker compose up` on its own only shows the help text. The Docker image pre-downloads Demucs and DeepFilterNet models so they're baked into the image.

### Google Colab

[`colab_pipeline.ipynb`](colab_pipeline.ipynb) runs the same pipeline on a Colab GPU, keeping models, config and output on Google Drive.

### Requirements

- Python 3.10+
- ffmpeg (`sudo apt install ffmpeg` or `brew install ffmpeg`)
- CUDA GPU recommended (12GB+ VRAM). CPU fallback is automatic but slow.
- [Deno](https://deno.com), the JavaScript runtime yt-dlp needs for YouTube extraction (only for `--url`; the pipeline adds `~/.deno/bin` to `PATH`)

### Check System Dependencies

```bash
podcast-cleaner check
```

Reports Python version, ffmpeg, CUDA/GPU availability, yt-dlp, and all ML library status.

## Usage

### Run on a YouTube video

```bash
podcast-cleaner run --url "https://www.youtube.com/watch?v=VIDEO_ID"
```

### Run on a YouTube playlist

```bash
podcast-cleaner run --url "https://www.youtube.com/playlist?list=PLAYLIST_ID"
```

### Run on local audio files

```bash
# Single file
podcast-cleaner run --input /path/to/episode.wav

# Directory of files (preserves existing NN_ prefixes in filenames)
podcast-cleaner run --input-dir /path/to/episodes/
```

### Resume a failed run

```bash
podcast-cleaner run --url "..." --resume
```

Without `--resume`, stages re-run from scratch. With `--resume`, completed stages are skipped.

### Skip stages

```bash
podcast-cleaner run --url "..." --skip transcribe --skip denoise
```

### Verbose / Quiet output

```bash
# Show detailed logging (DEBUG level)
podcast-cleaner run --input-dir ./episodes --verbose

# Only show progress and errors
podcast-cleaner run --input-dir ./episodes --quiet
```

### Cleanup intermediate files

```bash
# Delete preprocessed/separated/denoised/normalized dirs after success
podcast-cleaner run --input-dir ./episodes --cleanup-intermediates
```

This keeps only `raw/`, `final/`, and `analysis/` — useful for saving disk space. Cannot be combined with `--resume`.

### Copy input files (Docker / cross-filesystem)

```bash
podcast-cleaner run --input-dir ./episodes --copy-input
```

By default, input files are symlinked into the episode directory. Use `--copy-input` when working across filesystems (e.g., Docker volumes). Cross-filesystem situations are auto-detected even without this flag.

### Run a single stage

```bash
podcast-cleaner stage normalize output/01_My-Episode
```

### Analyze quality metrics

```bash
podcast-cleaner analyze output/01_My-Episode
```

Example output for one episode:
```
raw              LUFS= -30.1  SNR= 31.4dB  Peak= -5.3dBTP  Duration=990.0s
preprocessed     LUFS= -30.1  SNR= 31.4dB  Peak= -5.3dBTP  Duration=990.0s
separated        LUFS= -30.1  SNR= 34.6dB  Peak= -5.4dBTP  Duration=990.0s
denoised         LUFS= -30.3  SNR= 54.2dB  Peak= -5.6dBTP  Duration=990.0s
normalized       LUFS= -26.2  SNR= 54.2dB  Peak= -1.5dBTP  Duration=990.0s
```

The SNR column is a heuristic, not a measured signal-to-noise ratio: with no clean reference to compare against, it takes the mean level of the loudest half of 20 ms frames relative to the quietest 10%. Denoising pushes the quiet frames towards silence, so most of the jump after `denoise` reflects that rather than how the speech sounds. Use it to spot a stage that went wrong, not to rank outputs.

### All CLI Flags (run command)

| Flag | Description |
|---|---|
| `--url` | YouTube playlist or video URL |
| `--input-dir` | Directory of local audio files |
| `--input` | Single local audio file |
| `--config` | Path to config.yaml (default: `config.yaml`) |
| `--skip` | Skip a stage (repeatable) |
| `--resume` | Resume from last completed stage |
| `--cleanup-intermediates` | Delete intermediate dirs after success |
| `--copy-input` | Copy files instead of symlinking |
| `--verbose` / `-v` | Show detailed logging output |
| `--quiet` / `-q` | Only show progress and errors |

## Configuration

Copy `config.example.yaml` to `config.yaml` and edit it to tune the pipeline (`setup.sh` does the copy for you). `config.yaml` is ignored by git, so your settings stay local. Unknown keys trigger a warning. Numeric values are validated (e.g., `target_lufs` must be negative, `sample_rate` must be positive).

```yaml
output_dir: "./output"

download:
  format: "wav"
  quality: "best"
  # cookies_from_browser: "chrome"   # browser yt-dlp reads YouTube cookies from
  # remote_components: true          # let yt-dlp fetch its YouTube challenge solver

preprocess:
  sample_rate: 48000
  channels: 1

separation:
  model: "htdemucs_ft"   # Demucs model variant
  device: "auto"          # auto, cuda, cpu
  max_segment_minutes: 10

denoise:
  model: "DeepFilterNet3"

transcription:
  enabled: false          # Set true and install whisperx to enable
  model: "large-v3"
  language: null          # not read yet: WhisperX always detects the language
  device: "auto"

normalization:
  target_lufs: -16.0      # Podcast standard loudness
  true_peak_dbtp: -1.5    # Maximum true peak (prevents clipping)

export:
  formats: ["mp3", "flac"]
  mp3_bitrate: "320k"
  sample_rate: 48000
```

## Output Structure

```
output/
  01_Episode-Title/
    raw/                    # Downloaded audio
    preprocessed/           # 48kHz mono float32
    separated/              # vocals.wav + background.wav
    denoised/               # Noise-removed vocals
    normalized/             # Loudness-normalized audio
    final/                  # MP3, FLAC exports
      transcript/           # JSON + SRT (if transcription enabled)
    analysis/
      audio_report.json     # Quality metrics per stage
    processing.log          # Full pipeline log
    .download.done          # Stage completion markers
    .preprocess.done
    ...
```

## Testing

```bash
# Run all tests (fast, no GPU needed)
pytest -v -m "not slow"

# Run a specific test file
pytest tests/test_normalize.py -v

# Run quality regression tests against real processed audio
EPISODE_DIR=output/01_My-Episode pytest tests/test_audio_data.py -m slow -v
```

Unit and integration tests cover every stage with the ML models mocked, so they run without a GPU. Because the models are mocked, they test the pipeline's plumbing, not the quality of Demucs or DeepFilterNet output; the tests marked `slow` check quality thresholds against an episode you have processed yourself.

## Architecture

```
podcast_cleaner/
  cli.py                    # Click CLI: run, analyze, stage, check commands
  config.py                 # YAML config loading with defaults + validation
  display.py                # Shared Rich Console with TTY auto-detection
  tracker.py                # PipelineTracker for episode/stage progress
  utils.py                  # Audio I/O, done markers, device detection
  stages/
    __init__.py             # STAGE_ORDER definition
    download.py             # yt-dlp wrapper
    preprocess.py           # Resample + mono conversion
    separate.py             # Demucs vocal isolation (model cached per run)
    denoise.py              # DeepFilterNet3 noise removal (model cached, chunked)
    transcribe.py           # WhisperX transcription + SRT
    normalize.py            # pyloudnorm + true peak limiting
    export.py               # ffmpeg format conversion
  analysis/
    audio_stats.py          # LUFS, SNR, RMS, spectral centroid, true peak
```

Each stage follows the same pattern:
1. Check `is_done()` marker — skip if already completed
2. Find input files (with fallback to previous stage)
3. Process audio
4. Compute and save analysis metrics
5. Write `mark_done()` marker

ML models are lazy-imported to keep CLI startup fast. Demucs and DeepFilterNet models are loaded once per stage run and reused across files. GPU memory is released between stages via `gc.collect()` + `torch.cuda.empty_cache()`.

## CI/CD

- **GitHub Actions**: Runs tests on Python 3.10, 3.11, and 3.12 on every push and PR
- **Pre-commit hooks**: ruff (lint + format), trailing whitespace, YAML validation, large file check

```bash
# Install pre-commit hooks
pre-commit install
```

## Known Behavior

- **Normalization may not reach -16 LUFS** if the source audio has loud transients. The true peak limiter (-1.5 dBTP) prevents clipping, which can cap the overall loudness below target. This is intentional — clean audio over loud audio.
- **DeepFilterNet runs with cuDNN disabled** due to a compatibility issue with some torch/cuDNN versions. Performance impact is minimal.
- **Long audio is processed in 60s chunks** by DeepFilterNet to avoid GPU OOM on files >5 minutes.
- **Demucs outputs at 44100Hz** internally. The pipeline automatically resamples back to 48kHz after separation.
- **Demucs processing time** is roughly 2x real-time on a 12GB GPU. Episodes longer than ~35 minutes may fall back to CPU, which is significantly slower.
- **Very noisy recordings** (outdoor, ambient) may produce near-silence after denoising. The pipeline works best on studio/indoor podcast audio with SNR above ~20 dB.
- **yt-dlp requires a JavaScript runtime** (Deno) and browser cookies for YouTube authentication. Configure `cookies_from_browser` in `config.yaml`.
- **Only download audio you have the right to use.** YouTube's terms of service and the creators' copyright apply; this tool ships no content.
