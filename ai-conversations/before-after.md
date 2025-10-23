---
title: "Before/After Examples: 3 WeatherWise Prompts"
author: "Automated from AI conversations"
format:
  html: { toc: true }
---

# Example 1 — Fetch live weather data (wttr.in)

## Initial Prompt
```
Write a function to retrieve weather data from the wttr.in API for a given location.
```

## Initial AI-Generated Code (Before)
```python
def get_weather(location):
    import requests
    
    url = f"https://wttr.in/{location}?format=j1"
    response = requests.get(url)
    data = response.json()
    
    return data
```

## My Analysis of the Initial Code
- No error handling (network failures, bad responses).
- No input validation for `location`.
- No timeout (requests may hang).
- Returns raw API payload (no normalization).
- No documentation.

## My Follow-up Prompt
```
Improve the function to add:
1) input validation for location,
2) timeout and raise_for_status handling,
3) return a compact processed result (current + forecast),
4) support forecast_days parameter (1–5),
5) docstring and clear error handling.
```

## Improved AI-Generated Code (After)
```python
def get_weather_data(location, forecast_days=3, timeout=10):
    """
    Retrieve compact weather data from wttr.in.

    Args:
        location (str): City or location name.
        forecast_days (int): 1-5 days of forecast to return.
        timeout (int): HTTP request timeout in seconds.

    Returns:
        dict | None: {
            "location": str,
            "current": {"temperature": int, "feels_like": int, "condition": str, "humidity": int, "wind_kmph": int, "precip_mm": float},
            "forecast": [ {"date": str, "max_temp": int, "min_temp": int, "condition": str, "precip_mm": float}, ... ]
        } or None on error.
    """
    import requests

    if not location or not isinstance(location, str):
        return None

    forecast_days = max(1, min(int(forecast_days), 5))
    q = location.strip().replace(" ", "+")
    url = f"https://wttr.in/{q}?format=j1"

    try:
        resp = requests.get(url, timeout=timeout)
        resp.raise_for_status()
        data = resp.json()
    except requests.exceptions.RequestException:
        return None

    try:
        cur = data["current_condition"][0]
        result = {
            "location": location.strip(),
            "current": {
                "temperature": int(cur.get("temp_C", 0)),
                "feels_like": int(cur.get("FeelsLikeC", 0)),
                "condition": cur.get("weatherDesc", [{}])[0].get("value", ""),
                "humidity": int(cur.get("humidity", 0)),
                "wind_kmph": int(cur.get("windspeedKmph", 0)),
                "precip_mm": float(cur.get("precipMM", 0) or 0.0),
            },
            "forecast": []
        }

        for day in data.get("weather", [])[:forecast_days]:
            # pick a representative hourly slot (midday) if available
            hourly = day.get("hourly", [])
            mid = hourly[len(hourly)//2] if hourly else {}
            result["forecast"].append({
                "date": day.get("date"),
                "max_temp": int(day.get("maxtempC", 0)),
                "min_temp": int(day.get("mintempC", 0)),
                "condition": mid.get("weatherDesc", [{}])[0].get("value", ""),
                "precip_mm": float(mid.get("precipMM", 0) or 0.0),
            })

        return result
    except (KeyError, TypeError, ValueError):
        return None
```

## Why My Prompting Strategy Was Effective
- Specified concrete robustness requirements (validation, timeout, processed output).
- Requested a useful API shape (current + forecast) so AI produced structured data instead of raw JSON.
- Asked for bounds (forecast_days) which prevented unexpected outputs.


# Example 2 — Menu & input handling with PyInputPlus

## Initial Prompt
```
I need a simple menu in a Jupyter notebook to let users choose between Home, Reports, Settings. Use input() or simple prompts.
```

## Initial AI-Generated Code (Before)
```python
def menu():
    print("1) Home 2) Reports 3) Settings 0) Exit")
    choice = input("Choose: ").strip()
    return choice

while True:
    c = menu()
    if c == "1": print("Home...")
    elif c == "2": print("Reports...")
    elif c == "3": print("Settings...")
    elif c == "0": break
    else: print("Invalid")
```

## My Analysis of the Initial Code
- Uses raw input() — no validation, no retry limits, blocks indefinitely.
- Not user-friendly for repeated invalid attempts.
- Hard to reuse and test.

## My Follow-up Prompt
```
Switch to pyinputplus for robust prompts. Add retry limits, timeouts, defaults, and exception handling so the menu is reliable. Keep handlers in functions for testability.
```

## Improved AI-Generated Code (After)
```python
import pyinputplus as pyip
from pyinputplus import RetryLimitException, TimeoutException

def home():
    print("🏠 Home content")

def reports():
    print("📊 Reports content")

def settings():
    print("⚙️ Settings content")

ROUTES = {"Home": home, "Reports": reports, "Settings": settings}

def main_menu():
    try:
        choice = pyip.inputMenu(list(ROUTES.keys()) + ["Exit"],
                                numbered=True,
                                prompt="\n=== Main Menu ===\n",
                                limit=3,
                                timeout=60,
                                default="Exit")
    except (RetryLimitException, TimeoutException):
        print("No valid input received — exiting.")
        return

    if choice == "Exit":
        print("Bye.")
        return

    ROUTES[choice]()

if __name__ == "__main__":
    while True:
        main_menu()
        cont = pyip.inputYesNo("Return to menu? (yes/no): ", default="no", limit=2)
        if cont == "no":
            break
```

## Why My Prompting Strategy Was Effective
- Requested a specific library and failure modes (limit, timeout) so AI produced a safe, reusable pattern.
- Demanded separation of handlers which improves testability and readability.
- Asked for exception handling so flows return sensible defaults instead of crashing.


# Example 3 — Parse user weather questions into JSON with Ollama

## Initial Prompt
```
Use ollama SDK to parse a user's question into JSON with keys time_period, location, weather_attribute.
```

## Initial AI-Generated Code (Before)
```python
import ollama

client = ollama.Client()

msg = [{"role":"user","content":"Will it rain tomorrow in Perth?"}]
resp = client.chat(model="llama3:8b", messages=msg)
print(resp)
# then try to parse text manually for date/location/attribute
```

## My Analysis of the Initial Code
- Naive call to chat; response is free text and not guaranteed to match required JSON schema.
- No enforcement of structured output (risk of parsing errors).
- No validation/repair loop for bad JSON.
- No normalization of relative dates ("tomorrow").

## My Follow-up Prompt
```
Have the model return JSON only (format="json"), include a short system prompt that enforces keys and allowed attributes, validate with Pydantic, and implement a repair prompt if validation fails. Normalize relative dates later.
```

## Improved AI-Generated Code (After)
```python
from pydantic import BaseModel, ValidationError
from typing import Literal
import ollama, json, time
from datetime import datetime
# Simple schema
WeatherAttr = Literal["temperature","precipitation","humidity","wind","uv","aqi","forecast","now"]

class ParsedQuery(BaseModel):
    time_period: str
    location: str
    weather_attribute: WeatherAttr

SYSTEM = (
    "Extract exactly these keys into JSON only: time_period, location, weather_attribute. "
    "time_period must be ISO date or ISO-range (YYYY-MM-DD or YYYY-MM-DD/YYYY-MM-DD). "
    "weather_attribute must be one of: temperature, precipitation, humidity, wind, uv, aqi, forecast, now."
)

def ollama_json_chat(model, messages):
    client = ollama.Client()
    resp = client.chat(model=model, messages=messages, format="json")
    content = resp["message"]["content"]
    if isinstance(content, str):
        return json.loads(content)
    return content

def parse_weather_query(user_text, model="llama3.1:8b"):
    messages = [{"role":"system","content":SYSTEM},
                {"role":"user","content":user_text}]
    raw = ollama_json_chat(model, messages)
    try:
        return ParsedQuery(**raw)
    except ValidationError:
        # repair loop: ask model to fix previously returned JSON
        repair_prompt = (
            "The previous JSON did not match the schema. Fix it and return JSON with only the three keys."
            f"\nPrevious: {json.dumps(raw, ensure_ascii=False)}"
        )
        messages.append({"role":"user","content":repair_prompt})
        fixed = ollama_json_chat(model, messages)
        return ParsedQuery(**fixed)
```

## Why My Prompting Strategy Was Effective
- Enforced structured output with Ollama's `format="json"` to minimize parsing error.
- Added a strict system prompt and a Pydantic schema for validation and deterministic downstream use.
- Included a lightweight repair loop to handle edge cases where the model returns invalid JSON.
- Left normalization (relative dates → ISO) as a deterministic post-processing step, keeping LLM responsibilities scoped.

---