<div class="project-logo-wrapper" data-src="https://github.com/th0truth/hwmonitor/raw/master/.github/assets/logo.png">
  <img src="https://github.com/th0truth/hwmonitor/raw/master/.github/assets/logo.png" alt="hwmonitor logo" class="hwmonitor-logo-img" />
</div>

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

## Article
- Read post on DEV.to: [I wrote a hardware monitor in C that talks directly to the Linux kernel](https://dev.to/th0truth/i-wrote-a-hardware-monitor-in-c-that-talks-directly-to-the-linux-kernel-34k2)

