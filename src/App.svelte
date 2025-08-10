<script>
  import { onMount } from "svelte";
  import { marked } from "marked";
  import DOMPurify from "dompurify";
  // Simple weather app using Open-Meteo (no API key required)
  // 1) Geocode a city name -> lat/lon via Open-Meteo Geocoding API
  // 2) Fetch current weather from Open-Meteo Forecast API

  let city = $state("");
  let error = $state("");
  let weather = $state(null);
  let loading = $state(false);
  let inputEl;

  // "AI" response (now uses LM Studio local OpenAI-compatible server)
  let aiLoading = $state(false);
  let aiError = $state("");
  let aiResponse = $state("");
  let aiHtml = $state("");
  // LM Studio config (override with VITE_LMS_API_URL, VITE_LMS_API_KEY if needed)
  const lmsUrl = import.meta.env.VITE_LMS_API_URL || "http://localhost:1234/v1";
  const lmsKey = import.meta.env.VITE_LMS_API_KEY || "";
  let models = $state([]); // list of model ids
  let modelsLoading = $state(false);
  let modelsError = $state("");
  let model = $state(""); // selected model id
  let warming = $state(false);

  // Fetch available models from LM Studio
  const loadModels = async () => {
    modelsError = "";
    modelsLoading = true;
    try {
      const res = await fetch(`${lmsUrl}/models`, {
        headers: lmsKey ? { Authorization: `Bearer ${lmsKey}` } : undefined,
      });
      const data = await res.json().catch(() => null);
      if (!res.ok)
        throw new Error(data?.error?.message || "Failed to list models");
      const ids = (data?.data || []).map((m) => m.id).filter(Boolean);
      models = ids;
      // Select from localStorage or default to first
      const saved = localStorage.getItem("lmModel");
      if (saved && ids.includes(saved)) model = saved;
      else model = ids[0] || "";
    } catch (e) {
      modelsError = e?.message || "Could not load models from LM Studio";
    } finally {
      modelsLoading = false;
    }
  };

  // Warm the selected model (causes LM Studio to load it in memory)
  const warmModel = async () => {
    if (!model) return;
    warming = true;
    try {
      await fetch(`${lmsUrl}/chat/completions`, {
        method: "POST",
        headers: {
          "Content-Type": "application/json",
          ...(lmsKey ? { Authorization: `Bearer ${lmsKey}` } : {}),
        },
        body: JSON.stringify({
          model,
          messages: [{ role: "user", content: "Reply with OK" }],
          temperature: 0.2,
          max_tokens: 5,
          stream: false,
        }),
      });
    } catch (_) {
      // ignore warm errors
    } finally {
      warming = false;
    }
  };

  const getAIResponse = async () => {
    if (!model) throw new Error("No model selected in LM Studio");
    const code = Number(weather?.current?.weather_code);
    const condition = Number.isFinite(code)
      ? (weatherCodeText[code] ?? `WMO code ${code}`)
      : "";
    const t = weather?.current?.temperature_2m;
    const feels = weather?.current?.apparent_temperature;
    const wind = weather?.current?.wind_speed_10m;
    const rh = weather?.current?.relative_humidity_2m;

    const prompt =
      `Give practical, concise tips for someone in ${weather?.place?.name}, ${weather?.place?.country} right now.\n` +
      `Weather: ${condition}. Temperature ${t}°C (feels like ${feels}°C), humidity ${rh}%, wind ${wind} m/s.\n` +
      `Focus on clothing, outdoor planning, and any safety considerations. Keep it under 80 words.`;

    const res = await fetch(`${lmsUrl}/chat/completions`, {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        ...(lmsKey ? { Authorization: `Bearer ${lmsKey}` } : {}),
      },
      body: JSON.stringify({
        model,
        messages: [
          {
            role: "system",
            content:
              "You are a helpful assistant that gives concise, practical weather tips.",
          },
          { role: "user", content: prompt },
        ],
        temperature: 0.7,
        max_tokens: 180,
        stream: false,
      }),
    });
    const data = await res.json().catch(() => null);
    if (!res.ok) throw new Error(data?.error?.message || "AI request failed");
    const content = data?.choices?.[0]?.message?.content || "";
    return typeof content === "string" ? content : "";
  };

  // Optional: map Open-Meteo weather codes (WMO) to readable text
  const weatherCodeText = {
    0: "Clear sky",
    1: "Mainly clear",
    2: "Partly cloudy",
    3: "Overcast",
    45: "Fog",
    48: "Depositing rime fog",
    51: "Light drizzle",
    53: "Moderate drizzle",
    55: "Dense drizzle",
    56: "Light freezing drizzle",
    57: "Dense freezing drizzle",
    61: "Slight rain",
    63: "Moderate rain",
    65: "Heavy rain",
    66: "Light freezing rain",
    67: "Heavy freezing rain",
    71: "Slight snow fall",
    73: "Moderate snow fall",
    75: "Heavy snow fall",
    77: "Snow grains",
    80: "Slight rain showers",
    81: "Moderate rain showers",
    82: "Violent rain showers",
    85: "Slight snow showers",
    86: "Heavy snow showers",
    95: "Thunderstorm",
    96: "Thunderstorm with slight hail",
    99: "Thunderstorm with heavy hail",
  };

  const geocode = async (name) => {
    const url = `https://geocoding-api.open-meteo.com/v1/search?name=${encodeURIComponent(name)}&count=1&language=en&format=json`;
    const res = await fetch(url);
    const data = await res.json().catch(() => null);
    if (!res.ok) throw new Error(data?.reason || "Geocoding failed");
    const first = data?.results?.[0];
    if (!first) throw new Error(`No results for "${name}"`);
    return first; // { name, country, admin1, latitude, longitude, timezone }
  };

  const fetchCurrent = async (lat, lon, timezone) => {
    const url = `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lon}&current=temperature_2m,apparent_temperature,relative_humidity_2m,weather_code,wind_speed_10m&timezone=${encodeURIComponent(timezone || "auto")}`;
    const res = await fetch(url);
    const data = await res.json().catch(() => null);
    if (!res.ok) throw new Error(data?.reason || "Weather fetch failed");
    return data; // { current: {...}, current_units: {...}, ... }
  };

  const getWeather = async () => {
    error = "";
    loading = true;
    // reset AI panel for new search
    aiError = "";
    aiResponse = "";
    aiHtml = "";
    const q = (city || "").trim();
    if (!q) {
      error = "Please enter a city name.";
      loading = false;
      return;
    }
    try {
      const place = await geocode(q);
      const data = await fetchCurrent(
        place.latitude,
        place.longitude,
        place.timezone
      );
      weather = {
        place: {
          name: place.name,
          country: place.country,
          admin1: place.admin1,
          lat: place.latitude,
          lon: place.longitude,
          timezone: place.timezone,
        },
        current: data.current,
        units: data.current_units,
      };
      localStorage.setItem("lastWeather", JSON.stringify(weather));
      localStorage.setItem("lastCity", q);
      // kick off AI suggestion automatically (non-blocking)
      generateSuggestion();
    } catch (err) {
      error = err?.message || "Failed to fetch weather.";
    } finally {
      loading = false;
    }
  };

  const clearSearch = () => {
    city = "";
    error = "";
    inputEl?.focus();
  };

  const generateSuggestion = async () => {
    aiError = "";
    aiLoading = true;
    try {
      if (!weather?.current) {
        aiResponse = "Search a city first to get a tailored tip.";
        aiHtml = DOMPurify.sanitize(await marked.parse(aiResponse));
        return;
      }
      // Try LM Studio first
      try {
        const content = await getAIResponse();
        if (content && typeof content === "string") {
          aiResponse = content.trim();
          aiHtml = DOMPurify.sanitize(await marked.parse(aiResponse));
          return;
        }
      } catch (_) {
        // fall back to heuristic below
      }

      // Fallback: quick heuristic suggestions based on current conditions
      const t = Number(weather.current.temperature_2m);
      const wind = Number(weather.current.wind_speed_10m);
      const rh = Number(weather.current.relative_humidity_2m);
      const code = Number(weather.current.weather_code);

      const advice = [];
      if (t >= 30)
        advice.push("It's hot—stay hydrated and seek shade during midday.");
      else if (t >= 22)
        advice.push("Warm and pleasant—light clothing should be fine.");
      else if (t >= 15) advice.push("Mild—bring a light layer just in case.");
      else if (t >= 8) advice.push("Cool—pack a jacket or sweater.");
      else advice.push("Chilly—dress warm and consider a coat.");

      if (wind >= 30) advice.push("It's quite windy—secure hats/umbrellas.");
      else if (wind >= 15) advice.push("A bit breezy—layers help.");

      if (rh >= 85 && t > 20)
        advice.push("Humid conditions—pace yourself outdoors.");

      const codeHints = {
        0: "Clear skies ahead—great for outdoor plans.",
        1: "Mostly clear with a few clouds.",
        2: "Partly cloudy—sun breaks likely.",
        3: "Overcast—light can feel dim.",
        45: "Fog possible—allow extra travel time.",
        48: "Rime fog—surfaces may be slick.",
        51: "Light drizzle—pack a small umbrella.",
        53: "Drizzle—consider a waterproof layer.",
        55: "Steady drizzle—rain gear recommended.",
        61: "Light rain—umbrella or hood suggested.",
        63: "Rain—wear waterproof shoes.",
        65: "Heavy rain—avoid cotton layers.",
        71: "Light snow—roads may be slick.",
        73: "Snowfall—plan for slower travel.",
        75: "Heavy snow—check transit before heading out.",
        80: "Showers—conditions can change quickly.",
        81: "Rain showers—keep gear handy.",
        82: "Strong showers—seek shelter during bursts.",
        95: "Thunderstorms—watch lightning and stay indoors.",
        96: "Storms with hail—protect vehicles if possible.",
        99: "Severe storms—delay outdoor activities.",
      };
      if (codeHints[code]) advice.unshift(codeHints[code]);

      aiResponse = advice.join(" ");
      aiHtml = DOMPurify.sanitize(await marked.parse(aiResponse));
    } catch (e) {
      aiError = e?.message || "Failed to generate suggestion.";
    } finally {
      aiLoading = false;
    }
  };

  // Restore last session
  if (localStorage.getItem("lastWeather")) {
    try {
      weather = JSON.parse(localStorage.getItem("lastWeather"));
    } catch {}
  }
  if (localStorage.getItem("lastCity")) {
    try {
      city = localStorage.getItem("lastCity") || "";
    } catch {}
  }

  // Load models at startup and warm default selection
  onMount(async () => {
    await loadModels();
    if (model) {
      localStorage.setItem("lmModel", model);
      // warm model in background
      warmModel();
    }
  });
</script>

<div class="container">
  <form
    class="search"
    onsubmit={(e) => {
      e.preventDefault();
      getWeather();
    }}
    role="search"
    aria-label="Search city weather"
  >
    <div class="search-field">
      <span class="icon" aria-hidden="true">
        <!-- magnifying glass -->
        <svg
          viewBox="0 0 24 24"
          width="20"
          height="20"
          fill="none"
          stroke="currentColor"
          stroke-width="2"
          stroke-linecap="round"
          stroke-linejoin="round"
        >
          <circle cx="11" cy="11" r="8" />
          <line x1="21" y1="21" x2="16.65" y2="16.65" />
        </svg>
      </span>
      <input
        bind:this={inputEl}
        class="input"
        bind:value={city}
        placeholder="Search city (e.g. Berlin)…"
        autocomplete="off"
        autocapitalize="words"
        spellcheck="false"
        aria-invalid={error ? "true" : "false"}
        aria-describedby={error ? "search-error" : undefined}
      />
      {#if city}
        <button
          type="button"
          class="clear"
          aria-label="Clear"
          onclick={clearSearch}
        >
          <svg
            viewBox="0 0 24 24"
            width="18"
            height="18"
            fill="none"
            stroke="currentColor"
            stroke-width="2"
            stroke-linecap="round"
            stroke-linejoin="round"
          >
            <line x1="18" y1="6" x2="6" y2="18" />
            <line x1="6" y1="6" x2="18" y2="18" />
          </svg>
        </button>
      {/if}
      <button class="submit" type="submit" disabled={loading}>
        {#if loading}
          <span class="spinner" aria-hidden="true"></span>
          <span class="sr-only">Loading…</span>
        {:else}
          Search
        {/if}
      </button>
    </div>
    {#if error}
      <div id="search-error" class="error" role="alert" aria-live="polite">
        {error}
      </div>
    {/if}
  </form>

  {#if weather}
    <div class="card">
      <div class="card-header">
        <h2>
          Weather in {weather.place.name}{weather.place.country
            ? `, ${weather.place.country}`
            : ""}
        </h2>
        <p class="meta">
          {weather.place.admin1 ? `${weather.place.admin1} · ` : ""}{weather
            .place.timezone}
        </p>
      </div>
      <div class="grid">
        <div class="tile">
          <div class="label">Temperature</div>
          <div class="value">
            {weather.current.temperature_2m}{weather.units.temperature_2m}
          </div>
        </div>
        <div class="tile">
          <div class="label">Feels like</div>
          <div class="value">
            {weather.current.apparent_temperature}{weather.units
              .apparent_temperature}
          </div>
        </div>
        <div class="tile">
          <div class="label">Humidity</div>
          <div class="value">
            {weather.current.relative_humidity_2m}{weather.units
              .relative_humidity_2m}
          </div>
        </div>
        <div class="tile">
          <div class="label">Wind</div>
          <div class="value">
            {weather.current.wind_speed_10m}
            {weather.units.wind_speed_10m}
          </div>
        </div>
      </div>
      <div class="condition">
        {weatherCodeText[weather.current.weather_code] ??
          `Code ${weather.current.weather_code}`}
      </div>
      <div class="coords">
        {weather.place.lat.toFixed(2)}, {weather.place.lon.toFixed(2)}
      </div>
    </div>
  {/if}

  <section class="ai">
    <div class="ai-header">
      <h3><strong>AI suggestion</strong></h3>
      <div class="ai-actions">
        {#if modelsLoading}
          <span class="muted">Loading models…</span>
        {:else if modelsError}
          <span class="error" role="alert">{modelsError}</span>
        {:else}
          <label class="sr-only" for="model">Model</label>
          <select
            id="model"
            class="model-select"
            bind:value={model}
            onchange={() => {
              localStorage.setItem("lmModel", model);
              warmModel();
            }}
          >
            {#each models as m}
              <option value={m}>{m}</option>
            {/each}
          </select>
        {/if}
      </div>
    </div>
    {#if aiError}
      <div class="error" role="alert">{aiError}</div>
    {/if}
    <div class="ai-card" aria-live="polite">
      {#if aiLoading}
        <div class="ai-loading">
          <span class="spinner" aria-hidden="true"></span>
          <span>Generating tips…</span>
        </div>
      {:else if aiHtml}
        <div class="prose">{@html aiHtml}</div>
      {:else}
        <p class="muted">Tips about what to wear or plan will appear here.</p>
      {/if}
    </div>
  </section>
  <footer class="footer">
    <div class="footer-inner">
      <span class="muted">© {new Date().getFullYear()} Breezy Weather</span>
      <span class="sep">·</span>
      <a
        href="https://open-meteo.com/"
        target="_blank"
        rel="noreferrer"
        class="link">Data by Open‑Meteo</a
      >
      <span class="sep">·</span>
      <a
        href="https://lmstudio.ai/"
        target="_blank"
        rel="noreferrer"
        class="link">AI via LM Studio</a
      >
    </div>
  </footer>
</div>

<style>
  :root {
    --bg: linear-gradient(180deg, #0f172a, #0b1022);
    --panel: #0f172a;
    --muted: #94a3b8;
    --text: #e2e8f0;
    --ring: #60a5fa;
    --accent: #6366f1;
    --accent-700: #4f46e5;
    --accent-dark: #4f46e5;
    --accent-700-dark: #3b3b8c;
    --border: #1f2937;
  }

  .container {
    min-height: 100dvh;
    display: grid;
    place-items: start center;
    gap: 1.25rem;
    padding: 2rem 1rem 2rem;
    background: transparent;
    color: var(--text);
  }

  .search {
    width: min(720px, 92vw);
    display: grid;
    gap: 0.625rem;
  }
  .search-field {
    position: relative;
    display: flex;
    align-items: center;
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border);
    border-radius: 9999px;
    padding: 0.5rem 0.5rem 0.5rem 2.5rem;
    box-shadow:
      0 1px 0 rgba(255, 255, 255, 0.04) inset,
      0 10px 30px rgba(0, 0, 0, 0.25);
    transition:
      box-shadow 0.2s ease,
      border-color 0.2s ease;
  }
  .search-field:focus-within {
    border-color: color-mix(in srgb, var(--ring) 60%, var(--border));
    box-shadow:
      0 0 0 3px color-mix(in srgb, var(--ring) 25%, transparent),
      0 10px 30px rgba(0, 0, 0, 0.3);
  }
  .icon {
    position: absolute;
    left: 0.9rem;
    color: var(--muted);
    display: inline-flex;
    align-items: center;
    justify-content: center;
  }
  .input {
    flex: 1 1 auto;
    background: transparent;
    border: 0;
    outline: 0;
    color: var(--text);
    font-size: clamp(1rem, 1vw + 0.75rem, 1.125rem);
    line-height: 1.4;
  }
  .input::placeholder {
    color: color-mix(in srgb, var(--muted) 80%, transparent);
  }

  .clear {
    background: transparent;
    border: 0;
    color: var(--muted);
    padding: 0.5rem;
    margin-right: 0.25rem;
    border-radius: 0.5rem;
    cursor: pointer;
  }
  .clear:hover {
    color: #cbd5e1;
    background: rgba(255, 255, 255, 0.05);
  }

  .submit {
    appearance: none;
    border: 0;
    border-radius: 9999px;
    padding: 0.6rem 1rem;
    background: linear-gradient(180deg, var(--accent), var(--accent-700));
    color: white;
    font-weight: 600;
    cursor: pointer;
    min-width: 96px;
  }
  .submit[disabled] {
    opacity: 0.7;
    cursor: wait;
  }
  .submit:hover {
    background: linear-gradient(180deg, var(--accent-dark), var(--accent-700-dark));
  }

  .spinner {
    width: 16px;
    height: 16px;
    border: 2px solid rgba(255, 255, 255, 0.35);
    border-top-color: white;
    border-radius: 50%;
    display: inline-block;
    animation: spin 0.9s linear infinite;
    vertical-align: -2px;
    margin-right: 0.5rem;
  }
  @keyframes spin {
    to {
      transform: rotate(360deg);
    }
  }

  .error {
    background: #3f1f1f;
    border: 1px solid #7f1d1d;
    color: #fecaca;
    padding: 0.75rem 1rem;
    border-radius: 0.75rem;
  }

  .card {
    width: min(720px, 92vw);
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border);
    border-radius: 1rem;
    padding: 1.25rem;
    box-shadow: 0 10px 30px rgba(0, 0, 0, 0.3);
  }
  .card-header {
    margin-bottom: 0.5rem;
  }
  .card h2 {
    margin: 0 0 0.25rem 0;
    font-size: clamp(1.15rem, 1.2vw + 0.9rem, 1.5rem);
  }
  .meta {
    margin: 0;
    color: var(--muted);
    font-size: 0.95rem;
  }

  .grid {
    display: grid;
    grid-template-columns: repeat(2, minmax(0, 1fr));
    gap: 0.75rem;
    margin-top: 0.75rem;
  }
  @media (min-width: 640px) {
    .grid {
      grid-template-columns: repeat(4, minmax(0, 1fr));
    }
  }
  .tile {
    background: rgba(0, 0, 0, 0.25);
    border: 1px solid var(--border);
    border-radius: 0.75rem;
    padding: 0.85rem;
  }
  .label {
    color: var(--muted);
    font-size: 0.9rem;
  }
  .value {
    font-size: 1.1rem;
    font-weight: 600;
    margin-top: 0.25rem;
  }

  .condition {
    margin-top: 0.75rem;
    font-size: 1rem;
    color: #dbeafe;
  }
  .coords {
    margin-top: 0.25rem;
    color: var(--muted);
    font-size: 0.9rem;
  }

  .sr-only {
    position: absolute;
    width: 1px;
    height: 1px;
    padding: 0;
    margin: -1px;
    overflow: hidden;
    clip: rect(0, 0, 0, 0);
    white-space: nowrap;
    border: 0;
  }

  .ai {
    width: min(720px, 92vw);
    display: grid;
    gap: 0.5rem;
  }
  .ai-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
  }
  .ai h3 {
    margin: 0;
    font-size: 1.05rem;
    color: #dbeafe;
  }
  .ai-card {
    background: rgba(255, 255, 255, 0.04);
    border: 1px solid var(--border);
    border-radius: 1rem;
    padding: 1rem 1.25rem;
    color: var(--text);
    min-height: 68px;
  }
  .ai-loading {
    display: inline-flex;
    align-items: center;
    gap: 0.5rem;
    color: var(--muted);
  }
  /* prose helper intentionally minimal */
  .muted {
    color: var(--muted);
  }
  .model-select {
    appearance: none;
    background: rgba(0, 0, 0, 0.25);
    border: 1px solid var(--border);
    color: var(--text);
    padding: 0.5rem 0.75rem;
    border-radius: 0.5rem;
    margin-right: 0.5rem;
  }
  .footer {
    width: 100%;
    margin-top: auto; /* push to bottom when content is short */
    display: flex;
    align-items: center;
    justify-content: center;
  }
  .footer-inner {
    width: min(720px, 92vw);
    border-top: 1px solid var(--border);
    padding: 0.75rem 0.25rem 0;
    color: var(--muted);
    text-align: center;
    display: inline-flex;
    gap: 0.5rem;
    flex-wrap: wrap;
    align-items: center;
    justify-content: center;
  }
  .footer .sep {
    opacity: 0.6;
  }
  .footer .link {
    color: inherit;
    text-decoration: none;
  }
  .footer .link:hover {
    text-decoration: underline;
  }
</style>
