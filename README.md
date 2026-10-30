# Sift

Sift is a local, browser-based fake-news and claim-verification tool. It combines a responsive single-page interface with a Python Flask service, trusted-source lookups, and Google Gemma/Gemini models to produce an explainable fact-checking report.

The application is designed to help users investigate headlines, article text, social-media claims, article URLs, and news-video claims. It does not replace professional journalism, domain experts, or verification against primary sources.

## What it does

For each submission, Sift:

1. Accepts text, a web article URL, a YouTube/news-video URL, or a transcript/claim supplied with media.
2. Extracts article titles, descriptions, and paragraph text from ordinary web pages.
3. Extracts title and channel metadata from YouTube links through YouTube's oEmbed endpoint.
4. Searches several public knowledge and research repositories for related evidence:
   - PubMed / NIH / NLM
   - arXiv
   - Crossref
   - Wikipedia
5. Sends the claim and retrieved evidence to a configured Google generative model.
6. Displays a verdict, truth score, confidence level, explanation, claim-by-claim analysis, red flags, and cited evidence.
7. Lets users copy a short report or download the complete result as JSON.

## Features

- **Multiple input modes**
  - News headline, article body, or social-media statement
  - Article URL
  - YouTube or other video URL with optional spoken-claim notes
  - Audio/video file selection with a manually entered transcript or claim summary
- **Evidence grounding** using public academic and reference databases
- **Structured verdicts**
  - `VERIFIED`
  - `MOSTLY_TRUE`
  - `MISLEADING`
  - `UNSUBSTANTIATED`
  - `FABRICATED`
- **Claim-level explanations** with supporting source references
- **Rhetorical warning signals**, such as sensational language, unsupported medical claims, and missing evidence
- **Model fallback behavior** when the preferred model is unavailable
- **Local Flask API** for integrating the analyzer with other clients
- **Responsive dark glassmorphism UI** implemented without a frontend build system
- **Browser fallback mode** that can call the model directly if the local backend is unavailable
- **Sample claims** for quickly testing the interface

## Project structure

| File | Purpose |
| --- | --- |
| `FakeNewsDetector.html` | Standalone frontend. Provides the user interface, direct-browser analysis path, result rendering, report copying, and JSON export. |
| `fake_news_detector.py` | Main Python application. Hosts the Flask server, gathers evidence, extracts URL content, calls Google models, and embeds a similar frontend template. |
| `Launch_FakeNewsDetector.command` | macOS launcher script intended to start the local application and open the browser. Its hard-coded path may need to be updated for your machine. |

## Requirements

- Python 3.9 or newer
- Internet access for:
  - Google Generative Language API
  - PubMed / NIH
  - arXiv
  - Crossref
  - Wikipedia
  - Article and video URL retrieval
- A Google API key with access to the configured generative models
- Flask

The Python file imports Flask at runtime. Install it with:

```bash
python3 -m pip install flask
```

For isolated development, create a virtual environment first:

```bash
python3 -m venv .venv
source .venv/bin/activate
python3 -m pip install flask
```

## Running the application

### Start the Flask application

From the project directory:

```bash
python3 fake_news_detector.py
```

The server listens on:

```text
http://127.0.0.1:5050
```

The application attempts to open the default browser automatically. To disable that behavior:

```bash
NO_BROWSER=1 python3 fake_news_detector.py
```

To use another port:

```bash
PORT=8080 NO_BROWSER=1 python3 fake_news_detector.py
```

Then open `http://127.0.0.1:8080`.

### Use the HTML file directly

`FakeNewsDetector.html` can be opened in a browser as a static file. When the local Flask service is not available, its JavaScript attempts a direct-browser analysis path using Wikipedia and the Google Generative Language API.

The backend is recommended because it performs the full multi-source evidence search and keeps the workflow in one local service.

### Use the launcher

On macOS, make the launcher executable and run it:

```bash
chmod +x Launch_FakeNewsDetector.command
./Launch_FakeNewsDetector.command
```

Before using it, edit the `cd` path in `Launch_FakeNewsDetector.command` so it points to the directory containing this repository. The script currently references an older `app.py` filename; the repository's main Python entry point is `fake_news_detector.py`. Running the Python command directly is the most reliable option unless the launcher is updated.

## API configuration

The interface includes fields for:

- **API key**: Google Generative Language API credential
- **Model engine**: preferred model identifier

The Python application uses `gemma-4-31b-it` as its primary model and has fallback model identifiers configured in the source. If a model request fails, the service tries the configured alternatives and eventually returns an `UNSUBSTANTIATED` result with an API error explanation.

### Credential safety

Use your own API key and do not commit credentials to source control. Configure credentials through a local-only change or adapt the application to read a key from an environment variable or an untracked configuration file. If a key has ever been exposed in a public repository or browser-delivered HTML, revoke and rotate it immediately.

## HTTP API

The Flask service exposes the following routes.

### `GET /`

Returns the embedded Flask frontend.

### `GET /api/health`

Returns a basic service health response:

```json
{
  "status": "healthy",
  "primary_model": "gemma-4-31b-it"
}
```

### `POST /api/extract_url`

Extracts content and metadata from an article or YouTube URL.

Request:

```json
{
  "url": "https://example.com/news-story"
}
```

The response includes the URL, detected type, title, extracted content, author when available, and an error field when retrieval fails.

### `POST /api/trusted_sources`

Searches the configured evidence repositories.

Request:

```json
{
  "query": "claim or topic to investigate"
}
```

Response:

```json
{
  "evidence": [
    {
      "source_name": "PubMed (NIH/NLM - .gov)",
      "type": "Peer-Reviewed Medical Paper",
      "title": "Related publication",
      "url": "https://pubmed.ncbi.nlm.nih.gov/...",
      "identifier": "PMID: ...",
      "meta": "Publication metadata"
    }
  ]
}
```

### `POST /api/analyze`

Analyzes text, an article URL, or a video URL.

Example request:

```json
{
  "text": "Claim to investigate",
  "url": "",
  "video_url": "",
  "api_key": "YOUR_GOOGLE_API_KEY",
  "model": "gemma-4-31b-it"
}
```

The response is a structured fact-check report containing fields such as:

```json
{
  "verdict": "MISLEADING",
  "truth_score": 42,
  "confidence_level": "MEDIUM",
  "headline_summary": "One-sentence conclusion",
  "detailed_explanation": "Reasoned explanation",
  "claims": [
    {
      "claim": "Individual claim",
      "verdict": "FALSE",
      "reason": "Why the claim is unsupported",
      "source_ref": "Citation or source identifier"
    }
  ],
  "red_flags": [
    "Unsupported appeal to authority"
  ],
  "recommendations": [
    "Consult the original primary source"
  ],
  "model_used": "gemma-4-31b-it",
  "trusted_evidence": []
}
```

## Analysis pipeline

### 1. Input normalization

The service accepts either direct text or a URL. When a URL is provided, it attempts to retrieve and normalize the page title, meta description, and sufficiently long paragraph elements. YouTube links are handled separately through oEmbed metadata. The uploaded media control in the current UI is a transcript-assisted workflow: the file itself is not transcribed or uploaded by the backend.

### 2. Search query preparation

The application removes common stop words and keeps a limited set of high-signal terms from the submitted text. This shorter query is used to search the evidence providers.

### 3. Evidence retrieval

The backend queries PubMed, arXiv, Crossref, and Wikipedia. Each provider has a small result limit and request timeout. A failure from one provider is logged and does not prevent the remaining providers from returning results.

### 4. Model reasoning

The claim and evidence records are placed into a structured prompt. The selected Google model is asked to return JSON containing the verdict, score, confidence, explanation, individual claims, red flags, and recommendations. The service extracts JSON from a fenced code block or from a JSON object in the response.

### 5. Result presentation

The frontend renders the result as a color-coded verdict banner, truth score, explanation, claim cards, warning flags, and source cards. Results can be copied as a short report or downloaded as formatted JSON.

## Important limitations

- A model-generated verdict is not proof of truth or falsity.
- Search results are keyword-based and may be incomplete, irrelevant, delayed, or biased toward indexed sources.
- PubMed and Crossref results do not automatically establish that a claim is supported by the cited paper.
- arXiv contains preprints that may not have completed peer review.
- Wikipedia is a reference source, not a primary authority.
- Article extraction uses lightweight HTML pattern matching and may fail on JavaScript-rendered, paywalled, blocked, or unusually structured sites.
- Video URLs provide metadata; the application does not currently download or transcribe video audio.
- Selecting an audio/video file does not upload or process the file. Users must provide a transcript or the relevant claims.
- Browser-direct mode exposes the API key to the browser and should not be used for shared or production deployments.
- The default Flask server binds to `0.0.0.0`; use appropriate network controls before exposing it beyond the local machine.
- The application disables TLS certificate verification in its Python HTTP context. This is convenient for some local certificate issues but is not appropriate for a hardened production deployment.

## Development notes

The project is intentionally lightweight: there is no package manifest, frontend bundler, database, or automated test suite in the repository. The frontend is embedded in both the standalone HTML file and the Python service template, so changes to the UI may need to be applied in both places to keep their behavior consistent.

For a production-ready deployment, consider:

- Moving secrets to environment variables or a secret manager
- Removing the browser-direct API path
- Enabling normal TLS certificate verification
- Adding request validation, rate limiting, and structured logging
- Replacing HTML scraping with a robust content-extraction library
- Adding automated tests for URL extraction, provider failures, JSON parsing, and API routes
- Pinning dependencies in a requirements file
- Adding authentication before binding the service to a non-local interface

## Responsible use

Use Sift as an investigation aid. Read the linked source records, distinguish primary evidence from summaries and preprints, check publication dates, and seek qualified expert review for medical, legal, financial, or safety-critical claims.

## License

No license file is currently included in the repository. Until a license is added, users should treat the project as having no explicit open-source reuse permission.
