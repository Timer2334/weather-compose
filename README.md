![Platform](https://img.shields.io/badge/Platform-Android%209%2B%20(API%2028–35)-3DDC84)
![Kotlin](https://img.shields.io/badge/Kotlin-JVM%2017-7F52FF)
![UI](https://img.shields.io/badge/UI-Jetpack%20Compose%20%2B%20Material%203-4285F4)
![License](https://img.shields.io/badge/License-MIT-blue)

# WeatherCompose

Android weather client built with Kotlin, Jetpack Compose, and Clean Architecture layers, displaying current conditions and multi-day forecasts via the OpenWeather API.

Uses reactive `StateFlow` state management, Jetpack DataStore preference persistence, custom spring-animated Compose controls, and Lottie/Glide media rendering.

## OpenWeather API Pipeline Overview

| | Geocoding API | Current Weather API | 5-Day / 3-Hour Forecast API |
|---|---|---|---|
| **Endpoint** | `GET /geo/1.0/direct` | `GET /data/2.5/weather` | `GET /data/2.5/forecast` |
| **Purpose** | Resolves city name to latitude & longitude | Fetches live weather snapshot | Fetches 3-hour interval forecast entries |
| **Key Query Params** | `q`, `limit=1`, `appid` | `lat`, `lon`, `units`, `lang=ru`, `appid` | `lat`, `lon`, `units`, `lang=ru`, `appid` |
| **Response Model** | `List<CityWeather>` | `WeatherResponse` | `ForecastData` (`List<Forecast>`) |
| **UI Mapping** | Feeds coordinates to weather & forecast calls | Top hero panel (temperature, icon, humidity, wind) | Horizontal daily forecast strip (`LazyRow`) |

**In summary:**
* Every weather refresh executes a **three-step Retrofit pipeline** on `Dispatchers.IO`: city geocoding followed by current weather and 5-day forecast retrieval.
* Forecast payloads are grouped into daily buckets (`DailyForecasts<ForecastForUI>`) inside `WeatherRepository` before reaching the Compose UI.
* User preferences are persisted locally via **Jetpack DataStore** and drive reactive updates automatically when temperature units or theme settings change.

---

## Quick Start

### Prerequisites
* **Android Studio** (Ladybug or newer recommended for API 35)
* **JDK 17** (`sourceCompatibility` / `targetCompatibility = 17`)
* **Android SDK 35** (`compileSdk = 35`, `targetSdk = 35`, `minSdk = 28`)
* Free **OpenWeather API Key** from [openweathermap.org/api](https://openweathermap.org/api)

### Setup Steps

> **Repository layout note:** This repository contains the Android application module (`build.gradle.kts`, `proguard-rules.pro`, `src/`). To build locally, place it as the `app/` module inside a standard Gradle Kotlin DSL Android project with a version catalog (`gradle/libs.versions.toml`).

1. Clone the repository:
   ```bash
   git clone [https://github.com/Timer2334/weather-app.git](https://github.com/Timer2334/weather-app.git)
   ```
2. Configure your OpenWeather API key (see [OpenWeather API Setup](#openweather-api-setup)).
3. Sync Gradle in Android Studio and run on an emulator or physical device running **Android 9.0 (API 28)** or higher.

---

## Detailed Guide

1. [OpenWeather API Setup](#openweather-api-setup)
2. [Interface Overview](#interface-overview)
3. [Application Behavior](#application-behavior)
4. [Architecture](#architecture)
5. [Tech Stack](#tech-stack)
6. [Current Scope & Known Limitations](#current-scope--known-limitations)
7. [License](#license)

---

## OpenWeather API Setup

API credentials are intentionally excluded from version control.

1. Register at [https://home.openweathermap.org/api_keys](https://home.openweathermap.org/api_keys) and generate an API key (free tier covers `geo/1.0/direct`, `data/2.5/weather`, and `data/2.5/forecast`).
2. Pass your key to the `apiKey` parameter of the three endpoints defined in `data/WeatherResponseApi.kt` from `data/WeatherServerOperation.kt` (recommended: load it via `BuildConfig` from `local.properties` so the key is never committed):
   ```kotlin
   // data/WeatherServerOperation.kt
   val coordinates = weatherResponseApi.getCityWeather(
       cityName = cityName,
       apiKey = BuildConfig.OPENWEATHER_API_KEY
   )[0]
   ```

---

## Interface Overview

| Main Weather Screen | Settings Drawer |
|:---:|:---:|
| ![Main Screen](images/Main_Screen.png) | ![Settings Screen](images/Settings_Screen.png) |

### 1) Main Weather Screen
* **Top App Bar:** Displays the active city name with navigation triggers for the left Favorites drawer (`NavigationDrawerMenu`) and the right spring-animated Settings drawer (`NavigationDrawerSettings`).
* **Current Conditions Hero Card:** Linear gradient container rendering current temperature, `@4x` OpenWeather icon via **Glide Compose**, localized weather description, humidity with adaptive icon levels (`0–33%`, `34–66%`, `67–100%`), and wind speed. Displays an infinite **Lottie** animation (`assets/loading.json`) while network data is loading.
* **Multi-Day Forecast Strip:** Horizontally scrollable `LazyRow` of daily cards displaying the weather icon, integer temperature, and relative day label (`сегодня`, `завтра`, or weekday name).

### 2) Settings Drawer
* **Custom Sliding Navigation:** Right-side overlay drawer animated with `animateFloatAsState` (`Spring.StiffnessMediumLow`).
* **Custom `SegmentedButtons` Composable:** Pill-shaped selector using `drawBehind` layout measurement and animated indicator width/offset transitions with automatic horizontal scroll fallback when labels exceed screen width.
* **Configurable Preferences:**
  * **Temperature Units:** `°C` (`METRIC`) / `°F` (`IMPERIAL`).
  * **Wind Speed Units:** `м/с`, `км/ч`, `миль/ч`, `узлы`.
  * **Appearance Theme:** `Системная` (`FOLLOW_SYSTEM`), `Светлая` (`LIGHT`), `Тёмная` (`DARK`).
  * **Location Selector:** Opens a Material 3 `ModalBottomSheet` for switching between location modes.

---

## Application Behavior

### Reactive Preferences & Network Refresh
* `DataStore` exposes `UserSettingsData` as a cold Kotlin `Flow` backed by `androidx.datastore.preferences` (`name = "data"`).
* On startup, `WeatherViewModel.changeCheck()` collects settings into `StateFlow<UserSettingsData?>` and attaches a downstream collector using `filterNotNull()` and `distinctUntilChanged { old, new -> old.temperature == new.temperature }`.
* Switching between `METRIC` and `IMPERIAL` automatically triggers `serverRequest()` on `Dispatchers.IO` without restarting the activity; changing theme or wind speed does not cause redundant network requests.

### Forecast Grouping & Formatting
* OpenWeather returns 3-hour forecast slices across 5 days. `WeatherRepository` maps each `Forecast` item into `ForecastForUI`, partitions items into day buckets (`today` through `sixthDay`) by calendar day, and populates the summary strip using the first available slot per day.
* Timestamps are converted using `java.time.Instant` and `DateTimeFormatter` with `Locale("ru")` and `ZoneId.of("UTC+5")`.

### Gesture & Back Navigation
* The Favorites drawer search overlay (`ShowCitySearchPanel`) implements **Android Predictive Back** via `PredictiveBackHandler`, interpolating panel height smoothly with `CubicBezierEasing(0.1f, 0.1f, 0f, 1f)` and guarding concurrent gesture states with `MutatorMutex`.

### Default Values & Logging
* If no preferences are stored yet, `DataStore` initializes with `cityName = "Уфа"`, `temperature = METRIC`, `windSpeed = METERS_PER_SECOND`, and `themeConfig = FOLLOW_SYSTEM`.
* OkHttp attaches `HttpLoggingInterceptor` at `Level.BODY` for network inspection during development.

---

## Architecture

The project follows a layered Clean Architecture package layout under `src/main/java/com/example/timer2334_20`:

| Layer / Package | Key Components | Responsibility |
|---|---|---|
| `data` | `WeatherResponseApi`, `WeatherServerOperation`, `DataStore`, `data/model/*` | Retrofit service definitions, OkHttp setup, OpenWeather DTOs, and Jetpack DataStore persistence |
| `domain` | `WeatherRepository`, `domain/model/*`, `domain/util/Resource` | Data orchestration, 3-hour-to-daily forecast aggregation, enums (`Temperature`, `WindSpeed`, `ThemeConfig`), and UI models |
| `presentation` | `MainActivity`, `WeatherViewModel` | Activity lifecycle, edge-to-edge setup, splash screen integration, and `StateFlow` state holders |
| `presentation/ui/components` | `mainWindow/*`, `drawerSettings/*`, `drawerMenu/*` | Modular Jetpack Compose screens, custom `SegmentedButtons`, animated drawers, and `ModalBottomSheet` |
| `presentation/ui/theme` | `Theme.kt`, `Color.kt`, `Shape.kt`, `Type.kt` | Material 3 light/dark color schemes and dynamic theme switching via `ThemeConfig` |

---

## Tech Stack

| Category | Library / Tool |
|---|---|
| Language & Concurrency | Kotlin (JVM 17), Coroutines, `Flow` / `StateFlow` |
| UI Framework | Jetpack Compose (BOM), Material 3, Edge-to-Edge, `core-splashscreen` |
| Networking & Serialization | Retrofit 2, Gson Converter, OkHttp + `HttpLoggingInterceptor` |
| Persistence | Jetpack DataStore (Preferences) |
| Media & Animations | Glide Compose (weather icons), Lottie Compose (`loading.json`) |

---

## Current Scope & Known Limitations

* **Module-only repository:** Root Gradle wrapper and `libs.versions.toml` are not committed yet; the repository currently tracks the `app` module sources.
* **WIP UI sections:** City search backend binding (`ShowCitySearchPanel`), favorites list storage (`DrawerMenu`), pressure unit conversion, and client-side wind speed unit conversion on the main screen are scaffolded in UI/models and planned for completion.
* **Fixed locale & timezone:** Forecast date formatting currently targets `Locale("ru")` and `UTC+5` (`WeatherRepository.kt`).

---

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
