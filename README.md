# Team Horizon — Image to Line Drawing

A complete local Streamlit demo for the PMGT5850 capstone: upload one JPG/PNG,
preview it, click **Generate Line Drawing**, compare the original and AI line art,
and download a monochrome PNG. No login, database, billing UI or image history.

## Quick start (macOS / Linux)

Install Python 3.9 or newer (Python 3.11 recommended). Open a terminal in this
project folder, then run:

```sh
python3 -m venv .venv
source .venv/bin/activate
python -m pip install -r requirements.txt
cp .env.example .env
python -m streamlit run app.py
```

Open http://localhost:8501. The app starts without a key: upload/preview works,
and a setup message explains why Generate is disabled.

To enable AI generation, open `.env` in a text editor and set:

```dotenv
OPENAI_API_KEY=your_actual_openai_api_key
OPENAI_IMAGE_MODEL=gpt-image-1.5
```

Stop the server with Ctrl+C and run `python -m streamlit run app.py` again.
An existing `OPENAI_API_KEY` environment variable takes precedence over `.env`.
Create/manage your key at https://platform.openai.com/api-keys. Keep `.env` private;
it is excluded from version control. Never paste your key into the app or commit it.
Your API project needs access to the configured image model and available quota.

## Windows (PowerShell)

From the extracted project folder:

```powershell
py -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
.\.venv\Scripts\python.exe -m streamlit run app.py
```

Edit `.env` as above and restart. No PowerShell activation-policy changes are needed.

## Classroom demo

1. Upload a clear JPG or PNG (up to 10 MB and 25 megapixels).
2. Check the preview, then click **Generate Line Drawing** once.
3. Wait for the loading indicator to finish; generation can take a few minutes.
4. Compare the source and line drawing side by side.
5. Click **Download line drawing** to save `horizon-line-drawing.png`.

Uploading another image clears the previous result. Downloading does not generate
another image. A new Generate click makes a new API request. There is no offline
AI fallback; without credentials, preview is available but generation is disabled.

## Implementation

- `app.py`: professional English-language Streamlit UI, session state, error handling.
- `line_art.py`: file validation, orientation correction, fixed prompt and API call.
- `requirements.txt`: pinned, tested direct dependencies.
- `.env.example`: environment configuration template, containing no credentials.
- `.streamlit/config.toml`: light theme, upload cap and disabled usage telemetry.
- `tests/test_app.py`: validation, mocked generation and Streamlit UI tests.

The app uses the official `OpenAI` Python SDK's `client.images.edit(...)` endpoint,
sending the uploaded image as a PNG file tuple alongside a fixed prompt, then decoding
`result.data[0].b64_json`. The default `gpt-image-1.5` follows the official Python edit
example; it is not presented as the newest model. `OPENAI_IMAGE_MODEL` can select a
compatible GPT Image editing model without changing the UI. Output uses `size="auto"`,
`quality="medium"` and PNG. Requests have a 180-second timeout with automatic retries
disabled, so a timed-out request is not silently submitted again.

Official reference checked on 20 October 2026:
https://developers.openai.com/api/reference/python/resources/images/methods/edit

The fixed prompt asks for consistent black contours, minimal background, preserved
subject/composition, no shading and no added text. AI output is not deterministic and
may alter details. Output is converted to grayscale to eliminate color casts; smooth
edge pixels may contain gray values (this is not a strict two-color bitmap).

Inputs are validated by actual image content, not just extension. Animated PNGs,
corrupt files, unsupported formats and oversized files are rejected. EXIF orientation
is corrected, transparency is placed on white, metadata is stripped, and the image is
resized to a maximum 2048-pixel edge before preview and submission. The output's pixel
dimensions may differ from the input.

Images are held in the current app session, not written to a database or image folder.
Clicking Generate sends the normalized image to OpenAI, whose API data policies apply.
This is a local classroom prototype; hosting it publicly requires additional access
and spending controls.

## Troubleshooting

- **Setup needed:** set a nonempty `OPENAI_API_KEY` in `.env` and restart the server.
- **Key not accepted:** check that the key is valid and belongs to the intended API project.
- **Access denied:** check your project's image-model permissions/access.
- **Rate limit / quota:** check the API account's quota or wait and retry.
- **Request rejected:** try a different image; verify that the configured model supports
  image editing and the supplied parameters. The request may also have been filtered.
- **Timeout:** the remote request may have completed even if the response did not arrive.
  Wait before deliberately trying again; a retry is a new request.
- **Port in use:** run `python -m streamlit run app.py --server.port 8502`.

## Tests

With the environment installed, run:

```sh
python -m unittest discover -s tests -v
```

Tests do not call OpenAI or incur API charges. They exercise real JPG/PNG decoding,
invalid and oversized uploads, the SDK request contract using a mocked client, response
handling, no-key startup, upload/preview, generation/download and clearing results.
A live image-generation acceptance check requires your own configured API key:
upload a sample image, generate, inspect correspondence and download the PNG.
