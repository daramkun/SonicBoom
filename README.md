# SonicBoom

A high-performance web server that generates Text-to-Speech (TTS) audio using the Supertonic 2 ONNX model and delivers it via HTTP.

## Overview

SonicBoom is a Rust-based HTTP server that:

- Loads and runs the [Supertonic 2](https://huggingface.co/hexgrad/supertonic-2) ONNX model for high-quality TTS synthesis
- Provides RESTful API endpoints for text-to-speech conversion
- Streams generated audio to clients with minimal latency
- Includes an admin panel with session-based authentication
- Manages API token-based access control

## Features

- **ONNX Runtime Inference** - Runs Supertonic 2 efficiently using the `ort` crate with CoreML acceleration on Apple Silicon
- **Streaming Audio Output** - Generates and streams audio in real-time using Opus/OGG encoding
- **Token-Based Authentication** - API access controlled via tokens stored in a JSON file
- **Admin Panel** - Web-based admin interface with login protection and rate limiting
- **Background Model Loading** - Models download and load asynchronously without blocking the server startup
- **Session Management** - Secure session handling with in-memory storage via `tower-sessions`
- **Comprehensive Logging** - Structured logging with `tracing` and `tracing-subscriber`

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                        SonicBoom                            │
├─────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Web Routes  │  │  API Routes  │  │  Admin Routes   │   │
│  │  (Frontend)  │  │  (TTS API)   │  │  (Protected)    │   │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘   │
│         │                 │                   │             │
│         └─────────────────┼───────────────────┘             │
│                           ▼                                 │
│                   ┌─────────────┐                           │
│                   │  AppState   │                           │
│                   │ - Model     │                           │
│                   │ - Tokens    │                           │
│                   │ - Config    │                           │
│                   └──────┬──────┘                           │
│                          │                                   │
│         ┌────────────────┼────────────────┐                 │
│         ▼                ▼                ▼                 │
│  ┌────────────┐   ┌──────────────┐  ┌────────────┐         │
│  │  Supertonic│   │  TokenStore  │  │   Config   │         │
│  │   Model    │   │  (JSON)      │  │  (Env)     │         │
│  └────────────┘   └──────────────┘  └────────────┘         │
└─────────────────────────────────────────────────────────────┘
```

## API Endpoints

### TTS API (Requires Token)

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/api/tts` | Generate TTS audio from text |
| `GET` | `/api/status` | Check model loading status |

### Admin Panel

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/admin` | Admin dashboard (login required) |
| `POST` | `/admin/login` | Admin login |
| `POST` | `/admin/logout` | Admin logout |
| `GET` | `/admin/tokens` | List API tokens |
| `POST` | `/admin/tokens` | Create new API token |
| `DELETE` | `/admin/tokens/:id` | Revoke API token |

### Web Routes

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/` | Home page |
| `GET` | `/health` | Health check endpoint |

## Configuration

The application is configured via environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `PORT` | Server port | `3000` |
| `ADMIN_USERNAME` | Admin panel username | `admin` |
| `ADMIN_PASSWORD` | Admin panel password | `password` |
| `TOKEN_STORE_PATH` | Path to token storage file | `tokens.json` |
| `MODEL_CACHE_DIR` | Directory for cached models | `./models` |
| `HF_TOKEN` | HuggingFace token for model download | - |

## Installation

### Prerequisites

- Rust 1.75+ (preferably latest stable)
- For Apple Silicon (M1/M2/M3): CoreML support enabled automatically
- For other platforms: ONNX Runtime libraries

### Build

```bash
# Clone the repository
git clone https://github.com/yourusername/SonicBoom.git
cd SonicBoom

# Build the project
cargo build --release
```

### Run

```bash
# Set environment variables (optional)
export PORT=3000
export ADMIN_USERNAME=admin
export ADMIN_PASSWORD=your_secure_password

# Run the server
cargo run --release
```

The server will:
1. Start listening on the configured port
2. Begin downloading the Supertonic 2 model in the background
3. Load the model once download completes
4. Be ready to serve TTS requests

### Using Docker

```bash
# Build the Docker image
docker build -t sonicboom .

# Run the container
docker run -p 3000:3000 \
  -e ADMIN_USERNAME=admin \
  -e ADMIN_PASSWORD=your_password \
  -e HF_TOKEN=your_hf_token \
  sonicboom
```

Or use docker-compose:

```bash
docker-compose up -d
```

## Usage

### Generating TTS Audio

```bash
# Using curl with a bearer token
curl -X POST https://localhost:3000/api/tts \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"text": "Hello, world!", "voice": "default"}' \
  --output audio.ogg
```

### Admin Panel

Access the admin panel at `http://localhost:3000/admin` to:
- Manage API tokens
- View server status
- Monitor model loading progress

## Technology Stack

| Component | Technology |
|-----------|------------|
| Runtime | [Tokio](https://tokio.rs/) - Asynchronous runtime for Rust |
| Web Framework | [Axum](https://github.com/tokio-rs/axum) - Ergonomic and modular web framework |
| HTTP Middleware | [Tower](https://github.com/tower-rs/tower) - Library of modular network components |
| ONNX Inference | [ORT](https://github.com/DBDi/ort) - Rust bindings for ONNX Runtime |
| Audio Encoding | [Opus](https://opus-codec.org/) - High-quality audio codec |
| Serialization | [Serde](https://serde.rs/) - Serialization framework for Rust |
| Logging | [Tracing](https://tokio.rs/blog/tracing) - Structured logging for Rust |

## Project Structure

```
SonicBoom/
├── src/
│   ├── main.rs              # Application entry point
│   ├── config.rs            # Configuration management
│   ├── error.rs             # Error types and handling
│   ├── admin/               # Admin panel handlers
│   │   ├── handlers.rs      # Admin HTTP handlers
│   │   ├── lockout.rs       # Login attempt tracking
│   │   ├── templates.rs     # Admin HTML templates
│   │   └── session.rs       # Session utilities
│   ├── api/                 # Public API handlers
│   │   └── tts.rs           # TTS generation endpoint
│   ├── auth/                # Authentication & authorization
│   │   ├── mod.rs           # Auth module exports
│   │   ├── store.rs         # Token storage management
│   │   └── token.rs         # Token types and validation
│   ├── tts/                 # TTS engine implementation
│   │   ├── mod.rs           # Module exports
│   │   ├── model.rs         # ONNX model loading & inference
│   │   ├── inference.rs     # Text-to-audio generation
│   │   ├── audio.rs         # Audio encoding (Opus/OGG)
│   │   ├── text.rs          # Text preprocessing
│   │   └── download.rs      # Model downloading from HuggingFace
│   └── web/                 # Web frontend routes
│       ├── mod.rs           # Web module exports
│       └── index.rs         # Home page and static content
├── Cargo.toml               # Rust dependencies
├── Cargo.lock               # Locked dependency versions
├── Dockerfile               # Docker image definition
├── docker-compose.yml       # Docker Compose configuration
└── README.md                # This file
```

## License

MIT License - See LICENSE file for details.

## Acknowledgments

- [Supertonic 2](https://huggingface.co/hexgrad/supertonic-2) - The TTS model used
- [ONNX Runtime](https://onnxruntime.ai/) - Cross-platform ML inference engine
- All open-source contributors
