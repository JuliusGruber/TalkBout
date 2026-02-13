# Build and Run

## 1. Install dependencies

```bash
pip install -r requirements.txt
```

## 2. Set up environment variables

Copy the example environment file and fill in your credentials:

```bash
cp .env.example .env
```

Then edit `.env` with:
- **Mastodon credentials** (from wien.rocks) - use `scripts/register_app.py` to obtain these
- **Anthropic API key** - get from https://console.anthropic.com/

## 3. Run the application

```bash
python -m viennatalksbout
```

This starts the full ingestion pipeline: streaming posts from Mastodon → extracting topics via LLM → storing snapshots.

## Optional: Run tests

```bash
python -m pytest tests/ -v --tb=short --cov=viennatalksbout --cov-report=term-missing
```

## Configuration

You can customize pipeline behavior via environment variables (all optional):

| Variable | Default | Description |
|----------|---------|-------------|
| `VIENNATALKSBOUT_BUFFER_WINDOW_SECONDS` | 600 | Batch window in seconds |
| `VIENNATALKSBOUT_BUFFER_MAX_BATCH_SIZE` | 100 | Max posts per batch |
| `VIENNATALKSBOUT_SNAPSHOT_DIR` | `data/snapshots` | Where snapshots are stored |
| `VIENNATALKSBOUT_RETENTION_HOURS` | 24 | How long to keep snapshots |
| `VIENNATALKSBOUT_STALE_STREAM_SECONDS` | 1800 | Stream staleness threshold |
| `VIENNATALKSBOUT_HEALTH_LOG_INTERVAL` | 300 | Health log interval in seconds |
| `VIENNATALKSBOUT_LOG_LEVEL` | INFO | Logging verbosity |
