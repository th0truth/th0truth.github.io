# keylogger

Windows-based system monitoring utility and input analyzer designed for authorized security testing, parental control, and low-level event hooking.

> [!WARNING]
> THIS UTILITY IS INTENDED FOR AUTHORIZED TESTING, DIAGNOSTICS, AND EDUCATIONAL USE ONLY.

## Key Features
- **Global Input Hook**: Low-level keyboard hooks capturing global keystrokes, active application process names, and window titles.
- **Layout Transliteration**: Multi-language support mapping Cyrillic keyboard characters back to active system input layouts mid-session.
- **Parallel Hardware Diagnostics**: Runs concurrent background tasks via a multiprocessing pool to dump Windows system info, capture multi-monitor screenshots, record audio clips (WAV), and enumerate active hardware devices via PowerShell.

## Tech Stack
- **Python** (pynput, multiprocessing)
- **PowerShell** (Hardware enumeration)
- **Windows API**
