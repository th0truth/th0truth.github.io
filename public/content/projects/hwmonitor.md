# hwmonitor

Lightweight hardware discovery and monitoring CLI for Linux in pure C11, reading hardware data directly from `/sys` and `/proc` instead of shelling out to tools like `lspci`, `dmidecode`, or `lshw`.

Supports human-readable terminal output, JSON export, live watch mode, and AI-assisted hardware analysis via Groq API.

## Stack
- **C11**
- **Linux `/sys` and `/proc`**
- **libcurl**
- **Groq API**
- **cJSON**
- **Make**

## Highlights
- Direct kernel filesystem parsing
- JSON-first architecture
- Live monitoring
- Low-dependency design
- Memory-safe cleanup patterns
