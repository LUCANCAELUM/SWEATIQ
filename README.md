# HydroSense AI

Web app, backend and native mobile app for sweat-based hydration monitoring. Real measurements come only from compatible external sweat-sensor hardware. A phone cannot measure sweat chemistry.

> Status: first build, written without being able to run `npm install` or a build. Expect small fixes on first run. Send build errors back and they can be fixed quickly.

## What's inside

| Folder / file | Purpose |
|---|---|
| `src/` | React + TypeScript + Vite web app (installable PWA, works offline) |
| `src/engine.ts` | Simulator, sodium-loss calculation, risk levels with explanations, hydration advice |
| `src/sensorProfile.ts` | BLE UUIDs and the 18-byte sensor frame decoder (**edit for your hardware**) |
| `src/ble.ts` | Bluetooth: Web Bluetooth in Chrome, Capacitor BLE in the Android app |
| `src/firmware.ts` | Chunked firmware upload with CRC32 |
| `src/extras.tsx` | Alerts, GPS, team monitor (live WebSocket), reports, PDF export, device panel, lab validation |
| `backend/` | FastAPI, Postgres, Redis, JWT auth, session sync, WebSocket relay, scikit-learn insights |
| `mobile/` | Native Expo / React Native app with BLE |
| `.github/workflows/` | `deploy.yml` (GitHub Pages) and `android.yml` (debug APK) |

## Deploy the web app on GitHub

1. Create a repo and upload the unzipped files to `main`.
2. Settings → Pages → Source: **GitHub Actions**.
3. Push. The workflow builds and publishes the site.
4. Open the Pages link on your phone and add it to your home screen.

Local development: `npm install && npm run dev`

## Backend

```
docker compose up --build        # API on :8000
```

In the app: Device tab → Backend URL → Register or Log in → Sync.
Set a strong `JWT_SECRET` and `CORS_ORIGINS`. Use HTTPS in production (GitHub Pages is HTTPS, so the API must be too).

Endpoints: `POST /auth/register`, `POST /auth/login`, `POST /sessions/bulk`, `GET /sessions`, `GET /insights?planned_min=60`, `WS /ws/{room}?token=…`

## Android app (Capacitor)

Actions tab → **Android debug APK** → Run workflow → download the APK artifact. The workflow adds Bluetooth permissions. iOS: run `npx cap add ios` on a Mac.

## Native React Native app

```
cd mobile
npm install
npx expo prebuild
npx expo run:android      # needs a dev build, not Expo Go
```

`mobile/src/engine.ts` and `sensorProfile.ts` are copies of the web versions. Keep them in sync.

## Connect your real sweat sensor

Edit the UUIDs in `src/sensorProfile.ts`, or make your firmware emit this frame (18 bytes, little-endian):

| Bytes | Field |
|---|---|
| 0 | version = 1 |
| 1 | flags: bit0 sensor OK, bit2 wet, bit3 low battery |
| 2–3 | sequence number |
| 4–5 | sweat rate, 0.1 mL/h |
| 6–7 | sodium, 0.1 mmol/L |
| 8–9 | potassium, 0.1 mmol/L |
| 10–11 | chloride, 0.1 mmol/L |
| 12–13 | pH, 0.01 |
| 14–15 | skin temperature, signed, 0.01 °C |
| 16 | battery % |
| 17 | CRC8 (poly 0x07) over bytes 0–16 |

`0xFFFF` means invalid. Corrupt frames are dropped, and invalid or not-ready channels show as blank. Values are never invented.

## Firmware updates

The Device tab uploads a `.bin` file. Your firmware must implement this protocol:

1. Write `[0x01, length u32, crc32 u32]` to the firmware control characteristic.
2. Write `[seq u16, up to 128 bytes]` repeatedly to the firmware data characteristic.
3. Write `[0x02]` to the control characteristic.
4. The device verifies the CRC32, then reboots.

## Lab validation

Device tab → Lab validation. Enter paired values from the same sweat samples: raw sensor reading (calibration factor 1.00) and lab reference. Use 20 or more pairs across the range. The tool reports bias, error percentage, correlation, limits of agreement and a suggested calibration factor, and checks them against the accuracy limit you set. It does not validate anything by itself. The result depends on your lab data.

## Calculations and safety

- Sodium loss (mg/h) = sweat rate (L/h) × sweat sodium (mmol/L) × 22.99. This is calculated, not measured.
- Risk levels (GREEN, YELLOW, ORANGE, RED) are heuristic and list the factors behind each score. They are not a diagnosis.
- Sweat sodium is not blood sodium.
- Advice never recommends drinking beyond estimated losses, to avoid overhydration and hyponatremia.
- Simulated mode is clearly labeled and is not real measurement. Simulated sessions are excluded from the backend's ML insights.
- Stop and seek help for confusion, fainting, vomiting, severe headache or other heat-illness symptoms.

## Not included

- Native Wear OS and watchOS apps. Only a glanceable web view at `#watch` is provided.
- A trained model on validated data and the actual lab validation.
- Production hardening: rate limiting, backups and monitoring, and a security review.
- Medical or regulatory clearance.

## Quick glossary

- **PWA:** web app that installs on a phone and works offline.
- **GATT:** the Bluetooth Low Energy structure of services and characteristics.
- **MAPE:** mean absolute percentage error between sensor and lab values.
