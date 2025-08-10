# Breezy AI Weather

Lightweight Svelte app that shows current weather via Open‑Meteo and generates practical tips using a local LLM through LM Studio’s OpenAI‑compatible API. No API keys required for weather.

## Features

- Search any city (Open‑Meteo geocoding)
- Current conditions: temperature, feels‑like, humidity, wind, WMO code label
- Clean, modern UI with pill search and loading states
- AI suggestion panel with model dropdown (LM Studio)
- Offline/AI fallback heuristics if the model is unavailable
- Remembers last city and selected model

## Tech stack

- Svelte 5 + Vite
- Open‑Meteo Geocoding + Forecast APIs (no key)
- LM Studio local API (/v1/chat/completions)
- marked + DOMPurify for safe Markdown rendering

## Getting started

Prereqs: Node 18+ and npm. For AI suggestions, install LM Studio (optional).

1) Install deps

```bash
npm install
```

1) Run the dev server

```bash
npm run dev
```

Vite will print a local URL; open it in your browser.

## Environment variables (optional)

Create a `.env` in the project root to customize the LM Studio endpoint:

```env
VITE_LMS_API_URL=http://localhost:6223/v1
VITE_LMS_API_KEY=
```

If not set, the app defaults to `http://localhost:1234/v1` with no API key.

## LM Studio setup (optional but recommended)

1. Install LM Studio and run a local server (OpenAI‑compatible API).
2. Download a chat model (e.g., a small instruct model) and start the server.
3. Ensure the server exposes `/v1/models` and `/v1/chat/completions`.
4. Start the app; the model dropdown will populate automatically. The app will “warm” the selected model in the background.

If no server is found or a call fails, the app falls back to heuristic tips.

## How it works

- Geocode: `GET https://geocoding-api.open-meteo.com/v1/search?name=<city>&count=1&language=en&format=json`
- Forecast (current): `GET https://api.open-meteo.com/v1/forecast?latitude=<lat>&longitude=<lon>&current=temperature_2m,apparent_temperature,relative_humidity_2m,weather_code,wind_speed_10m&timezone=auto`
- AI tips: POST to `<VITE_LMS_API_URL>/chat/completions` with the current weather summary

## Accessibility

- Keyboard accessible form controls
- Clear focus and hover states
- Reduced‑motion media query respected for hover transitions

## Troubleshooting

- Models list empty: check LM Studio is running, correct port in `.env`, and that `/v1/models` responds.
- AI errors: verify the model is loaded and try another model in the dropdown. The app will still show heuristic tips.
- CORS issues: LM Studio should run on localhost; if proxied, allow CORS for the dev origin.
- Weather not loading: verify network connectivity; Open‑Meteo requires no key.

## License

MIT

