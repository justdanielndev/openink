# openInk

Project to display custom dashboards ("pages") on a Waveshare 7.5" E-Ink display. It uses Node.js w/ Puppeteer to capture screenshots of the dashboard and a Python script to update the E-Ink display when changes happen. It was designed at first to help me a bit with avoiding forgetting to check my tasks and notifications, and now it's just a full on every dashboard :3

When paired with an [openInk Cloud](https://openink.isitzoe.dev) instance (also OSS), you can manage multiple devices and their configurations from a web interface.

## Architecture

1.  **Screenshot Service (`capture.js`)**:
    *   Runs a Chromium browser via Puppeteer.
    *   **Dynamic Configuration**: Fetches screen rotation rules, URLs, and schedules from a `device.json` file.
    *   Logs into Home Assistant automatically.
    *   Continuously monitors the dashboard for visual changes using `pixelmatch`.
    *   Saves a new screenshot to `shared/current_view.png` when a change is detected.

2.  **Display Service (`display.py`)**:
    *   Monitors the shared directory for file updates.
    *   Updates the Waveshare E-Ink display.
    *   Switches between "Fast" and "Standard" refresh modes to avoid ghosting.

3. **openInk Cloud**:
    *   A web-based platform to manage multiple devices and their configurations.
    *   Provides an API endpoint to serve the `device.json` configuration files.
    *   Serves a health check endpoint to monitor device status.

## Hardware Requirements

*   **Device** to run the screenshot service (e.g., Raspberry Pi, PC). Note that it obviously needs to support the HAT.
*.  **Server/Hosting** (optional) if you want to run your own openInk Cloud instance. Not required if you want to use `device.json` or the free official openInk Cloud hosted server.
*   **Waveshare 7.5inch e-Paper HAT (V2)**

## Software Requirements

*   **Node.js**
*   **Python 3**
*   **Chromium installed** (for Puppeteer)
*   **Home Assistant** instance (not required, you can just use any web page, but some conditions and implementations are Home Assistant specific)

## Installation

### 1. Clone the Repository
```bash
git clone https://github.com/justdanielndev/openink/
cd openink
```

### 2. Setup Screenshot Service

```bash
npm install
```

Create a `.env` file:
```ini
JSON_URL=http://path/to/your/device.json
# Optional: Fallback URL if JSON is unreachable
CAPTURE_URL=https://google.com
HA_USERNAME=your_ha_username
HA_PASSWORD=your_ha_password
# Optional: Path to chromium if not standard Pi
# PUPPETEER_EXECUTABLE_PATH=/usr/bin/chromium
```

### 3. Configuration (`device.json`)

You need to host or provide a `device.json` file that tells the service what to display.

This can now be done via the e-Ink Platform (also OSS), or you can create your own JSON file. Example:

```json
{
    "screens": [
        {
            "url": "http://homeassistant.local:8123/dashboard-eink/0?kiosk",
            "duration": 20
        },
        {
            "url": "http://homeassistant.local:8123/dashboard-eink/1?kiosk",
            "duration": 20
        },
        {
            "url": "http://homeassistant.local:8123/dashboard-eink/2?kiosk",
            "starttime": "20:00",
            "endtime": "06:00"
        },
        {
            "url": "http://homeassistant.local:8123/dashboard-eink/alert?kiosk",
            "conditions": [
                {
                    "type": "if-user-zone",
                    "user": "person.z",
                    "zone": "home",
                    "expected_state": true
                },
                {
                    "type": "day-of-week",
                    "days": [1, 2, 3, 4, 5],
                    "expected_state": true
                }
            ],
            "force_show_if_conditions_match": true
        }
    ],
    "randomize_screens": true,
    "json_refresh_interval": 3,
    "conditions_check_interval": 5
}
```

*   **screens**: List of URLs to display.
    *   **url**: The dashboard URL to display.
    *   **duration**: (Optional) Minutes to display before rotating (default: 20).
    *   **starttime/endtime**: (Optional) Time range (HH:MM) where this screen takes priority over rotation.
    *   **conditions**: (Optional) Array of conditions that must be met to display this screen.
        *   **type**: Type of condition. Supported types:
            *   **if-user-zone**: Checks if a Home Assistant person entity is in a specific zone.
                *   **user**: Home Assistant person entity ID (e.g., `person.zoe`).
                *   **zone**: Home Assistant zone entity ID (e.g., `zone.home`).
                *   **expected_state**: true if the user should be in the zone, false otherwise.
            *   **day-of-week**: Checks if today is one of the specified days.
                *   **days**: Array of integers representing days of the week (0 = Sunday, 6 = Saturday).
                *   **expected_state**: true if today should be in the list, false otherwise.
            *   **if-calendar-event**: Checks for events in a Home Assistant calendar.
                *   **calendar**: Home Assistant calendar entity ID (e.g., `calendar.personal`).
                *   **search**: (Optional) Keyword to search for in event title/description.
                *   **offset**: (Optional) Minutes into the future to look for events (default: 0, matches current events).
                *   **expected_state**: true if a matching event should exist, false otherwise.
    *   **force_show_if_conditions_match**: (Optional) If true, shows only this screen (and others with the same check with their conditions matching too) when all conditions match, overriding normal rotation and time checks.
        
*   **randomize_screens**: If true, picks random screens instead of sequential order.
*   **first_day_of_week**: (Optional) "monday" or "sunday" (default: "monday"). Used for UI display in the cloud platform.
*   **json_refresh_interval**: Minutes between checking for config updates.
*   **conditions_check_interval**: Minutes between checking condition states (e.g., user location, day of week).

### 4. Configuration (openInk Cloud)

You can use the official hosted openInk Cloud at [openink.isitzoe.dev](https://openink.isitzoe.dev) or host your own instance. To host your own, you'll need to:

Open the `platform` directory and install the dependencies:
```bash
pnpm install
```

Create an appwrite project and database, then set up the required collections (Devices, Users, etc.) as per the schema in the `platform` directory.

Create a `.env` file in the `platform` directory with the following content:
```ini
NEXT_PUBLIC_APPWRITE_ENDPOINT=https://cloud.appwrite.io/v1
NEXT_PUBLIC_APPWRITE_PROJECT_ID=your_project_id
NEXT_PUBLIC_APPWRITE_DATABASE_ID=your_database_id
NEXT_PUBLIC_APPWRITE_COLLECTION_ID=your_collection_id
APPWRITE_API_KEY=your_api_key
```

And finally, run the development server:
```bash
pnpm dev
```

### 5. Setup Display Service
Install the required Python libraries:

```bash
pip3 install RPi.GPIO spidev Pillow numpy
```

*Note: You will also need the Waveshare e-Paper drivers. The repo already includes them in the `lib` directory.*

## Usage

### Running the Screenshot Service
```bash
node ./capture.js
```

### Running the Display Service
```bash
python3 ./display.py
```
