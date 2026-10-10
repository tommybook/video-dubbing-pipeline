# Video Dubbing Pipeline

Dubs a Russian-language video into English and keeps the original speaker's voice. The pipeline transcribes the speech, translates it, synthesizes English audio with the speaker's timbre and puts it back on the original timeline.

A side project from May 2024.

## Demo

[Dubbed sample](https://github.com/tommybook/video-dubbing-pipeline/assets/55095368/54782cc0-8e15-4fbc-9488-1e71d1dd7c63). The source clip is in [`examples/`](examples).

## How it works

| Step | What happens | Tool |
|---|---|---|
| 1. Download | The video is fetched by URL (Google Drive links) | `gdown` |
| 2. Extract audio | The audio track is saved as WAV | `moviepy` |
| 3. Transcribe | Russian speech is transcribed with word-level timestamps | [WhisperX](https://github.com/m-bain/whisperX), Whisper `large` |
| 4. Translate | Each segment is translated to English | [`Helsinki-NLP/opus-mt-ru-en`](https://huggingface.co/Helsinki-NLP/opus-mt-ru-en) |
| 5. Synthesize | English speech is generated, then the original speaker's timbre is transferred onto it | [MeloTTS](https://github.com/myshell-ai/MeloTTS) + [OpenVoice v2](https://github.com/myshell-ai/OpenVoice) |
| 6. Align | The generated audio is transcribed again to get word timings, which are matched back to the translated sentences | WhisperX, `difflib` |
| 7. Mix | For each spoken segment the original track is turned down by 20 dB and the English segment is overlaid at the same position | `pydub` |
| 8. Replace | The video is written with the new audio track | `moviepy` |

## Installation

A CUDA GPU is recommended. On CPU the pipeline falls back to `int8` Whisper and runs slowly. `ffmpeg` must be installed.

1. Clone the repository:

    ```bash
    git clone https://github.com/tommybook/video-dubbing-pipeline.git
    cd video-dubbing-pipeline
    ```

2. Install the packages:

    ```bash
    pip install -r requirements.txt
    ```

3. Download the OpenVoice v2 checkpoints:

    ```bash
    wget https://myshell-public-repo-hosting.s3.amazonaws.com/openvoice/checkpoints_v2_0417.zip
    unzip checkpoints_v2_0417.zip
    ```

4. If CUDA libraries are missing, this may help:

    ```bash
    apt install libcublas11
    ```

## Usage

Set the parameters in [`main.py`](main.py) and run it:

```bash
python main.py
```

| Parameter | Meaning |
|---|---|
| `video_url` | Link to the source video (Google Drive) |
| `video_path` | Where the downloaded video is stored; if the file already exists, the download is skipped |
| `audio_output_path` | Extracted original audio |
| `new_audio_path` | Generated English speech before mixing |
| `output_video_path` | Final dubbed video |
| `checkpoints_dir` | Folder with the unpacked OpenVoice checkpoints |
| `translation_model` | Hugging Face translation model |
| `tts_model_name` | MeloTTS language/voice, `EN_NEWEST` by default |
| `transcription_model_size` | Whisper model size, `large` by default |

The mixed audio track is also saved as `synchronized_audio.wav`.

## Project structure

```
main.py                         entry point with parameters
dubbing_pipeline.py             DubbingPipeline: runs all steps in order
video_packages/
    video_download.py           VideoDownloader
    video_replacer.py           VideoReplacer
audio_packages/
    audio_extractor.py          AudioExtractor
    audio_generation.py         AudioGenerator: MeloTTS + OpenVoice tone conversion
    audio_processor_utils.py    AudioProcessorUtils: ducking and overlay
text_packages/
    transcription_model.py      TranscriptionModel: WhisperX transcription and alignment
    text_translator.py          TextTranslator
    segment_matcher.py          TextSegmenter: maps word timings to translated sentences
examples/                       source clip for the demo
```

## Limitations

- Russian to English only. Another language pair needs a different translation model and TTS voice.
- One speaker. The voice is cloned from the whole audio track.
- No source separation. Music and background noise are turned down together with the voice.
- No time-stretching. An English phrase that is longer than the original pushes the next one later.
- WhisperX, OpenVoice and MeloTTS are installed from their main branches without pinned versions. The code was last updated in May 2024 and may need version fixes to run today.
