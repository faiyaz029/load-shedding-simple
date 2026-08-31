
---


# ⚡ ESP32 Smart Load-Shedding Monitor

An ultra-efficient, event-driven IoT power grid monitor built with an ESP32 and Firebase Realtime Database. 

Unlike traditional power monitors that require complex high-voltage sensing circuitry (optocouplers, relays, voltage dividers) or spam databases with constant "power is on" pings, this project utilizes a **"Dead Man's Switch" (Absence of Signal) combined with a Smart-Boot Calculation algorithm**. 

The system provides a real-time web dashboard, calculates daily/weekly outage durations, and exports clean CSV datasets suitable for Machine Learning and predictive analysis.

---

## 🧠 The Architecture: "Smart Boot" Event-Driven Logging

The ESP32 is powered directly from a standard 5V wall adapter (no battery). The logic relies on state changes rather than continuous polling:

1. **The Heartbeat (Zero Database Bloat):** While power is on, the ESP32 overwrites a single `last_ping` Unix timestamp in Firebase every 60 seconds. This uses no additional database storage over time.
2. **The Outage:** When grid power drops, the ESP32 instantly loses power and dies.
3. **The Smart Boot (The Magic):** When power returns, the ESP32 boots up, syncs the exact time via NTP, and fetches its own death timestamp (`last_ping`). It calculates the duration of the outage and logs a single, clean JSON object to the database (e.g., `{start: 1725045000, end: 1725048600, duration: 3600}`).

**Result:** A database that only grows when an actual event occurs, remaining lightweight enough to run on Firebase's free tier for decades.

---

## ✨ Features

- **Zero High-Voltage Risk:** No direct AC mains wiring required. Simply plug the ESP32 into a USB wall block.
- **Event-Driven Database:** Only logs verified power cuts.
- **Real-Time Web Dashboard:** Instantly view current grid status and historical logs from any browser.
- **Ongoing Outage Detection:** Dashboard dynamically calculates downtime before the ESP32 even boots back up.
- **Dataset Generation (CSV):** One-click export of outage datasets (Date, Time Off, Time On, Duration) for data analysis and ML predictions.

---

## 🛠️ Step 1: Firebase Configuration

1. Go to the [Firebase Console](https://console.firebase.google.com/) and create a new project.
2. Navigate to **Build > Realtime Database** and click **Create Database**.
3. Go to the **Rules** tab and set the database to public read/write. This allows the ESP32 to communicate via simple REST API calls without heavy OAuth token libraries:
   ```json
   {
     "rules": {
       ".read": true,
       ".write": true
     }
   }



4. Note your Database URL (e.g., `https://<YOUR-PROJECT-ID>.firebasedatabase.app`).

### Database Structure

The system will automatically generate this optimized structure:

```json
{
  "status": {
    "last_ping": 1725131563
  },
  "outages": {
    "-O1XyZabc123": {
      "start": 1725045000,
      "end": 1725048600,
      "duration_sec": 3600
    }
  }
}

```

---

## 💻 Step 2: ESP32 Firmware (C++)

1. Open the Arduino IDE with the ESP32 board manager installed.
2. Replace `YOUR_WIFI_SSID`, `YOUR_WIFI_PASSWORD`, and `<YOUR-PROJECT-ID>` in the code below.
3. Flash to your ESP32.

```cpp
#include <WiFi.h>
#include <HTTPClient.h>
#include <time.h>

// --- CONFIGURATION ---
const char* ssid = "YOUR_WIFI_SSID";
const char* password = "YOUR_WIFI_PASSWORD";

// Base URL of your database (NO .json at the end here)
String db_base = "https://<YOUR-PROJECT-ID>.firebasedatabase.app";

// NTP Server settings (Set for GMT+6 Bangladesh Time)
const char* ntpServer = "pool.ntp.org";
const long  gmtOffset_sec = 21600; 
const int   daylightOffset_sec = 0;

void setup() {
  Serial.begin(115200);
  
  // 1. Connect to WiFi
  WiFi.begin(ssid, password);
  Serial.print("Connecting to WiFi");
  while (WiFi.status() != WL_CONNECTED) { delay(500); Serial.print("."); }
  Serial.println("\nWiFi Connected!");
  
  // 2. Sync Time from the Internet
  configTime(gmtOffset_sec, daylightOffset_sec, ntpServer);
  Serial.print("Syncing time");
  while (time(nullptr) < 100000) { delay(500); Serial.print("."); }
  Serial.println("\nTime Synced!");

  // 3. THE SMART BOOT LOGIC
  HTTPClient http;
  http.begin(db_base + "/status/last_ping.json");
  int getCode = http.GET();
  
  if (getCode == 200) {
    String lastPingStr = http.getString();
    unsigned long lastPing = lastPingStr.toInt();
    unsigned long currentTime = time(nullptr);
    
    // Calculate the gap in seconds
    unsigned long gap = currentTime - lastPing;
    
    // If the gap is > 3 minutes (180s), log it as a confirmed outage
    if (lastPing > 0 && gap > 180) {
      Serial.printf("Outage Detected! Power was off for %lu seconds.\n", gap);
      
      String outageData = "{\"start\": " + String(lastPing) + ", \"end\": " + String(currentTime) + ", \"duration_sec\": " + String(gap) + "}";
      
      http.begin(db_base + "/outages.json");
      http.addHeader("Content-Type", "application/json");
      http.POST(outageData);
    }
  }
  http.end();
}

void loop() {
  if (WiFi.status() == WL_CONNECTED) {
    HTTPClient http;
    
    // Overwrite heartbeat timestamp every minute (zero DB bloat)
    http.begin(db_base + "/status/last_ping.json");
    unsigned long currentUnixTime = time(nullptr);
    http.PUT(String(currentUnixTime));
    http.end();
    
    Serial.printf("Heartbeat updated: %lu\n", currentUnixTime);
  }
  delay(60000); // 1-minute heartbeat interval
}

```

---

## 🌐 Step 3: Web Dashboard (HTML/JS)

The dashboard is a dependency-free, single-file HTML application. It calculates total outages for the day/week and detects ongoing power cuts dynamically.

1. Create an `index.html` file.
2. Paste the code below.
3. Change `DB_BASE` to your Firebase URL.
4. Host it for free on GitHub Pages, Vercel, or open it locally.

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Smart Load Shedding Analytics</title>
    <style>
        body { font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; background: #f0f2f5; margin: 0; padding: 20px; color: #333; }
        .container { max-width: 900px; margin: 0 auto; }
        .grid { display: grid; grid-template-columns: repeat(auto-fit, minmax(250px, 1fr)); gap: 20px; margin-bottom: 20px; }
        .card { background: white; padding: 25px; border-radius: 12px; box-shadow: 0 4px 6px rgba(0,0,0,0.05); text-align: center; }
        .status { font-size: 20px; font-weight: bold; padding: 15px; border-radius: 8px; color: white; margin-bottom: 10px; }
        .online { background-color: #10b981; }
        .offline { background-color: #ef4444; }
        .stat-value { font-size: 32px; font-weight: bold; color: #2563eb; margin: 10px 0; }
        .stat-label { font-size: 14px; color: #6b7280; text-transform: uppercase; letter-spacing: 1px; }
        table { width: 100%; border-collapse: collapse; background: white; border-radius: 12px; overflow: hidden; box-shadow: 0 4px 6px rgba(0,0,0,0.05); }
        th, td { padding: 15px; text-align: left; border-bottom: 1px solid #e5e7eb; }
        th { background-color: #f8fafc; font-weight: 600; color: #475569; }
        button { background: #2563eb; color: white; border: none; padding: 12px 24px; border-radius: 8px; cursor: pointer; font-weight: bold; transition: background 0.3s; }
        button:hover { background: #1d4ed8; }
        .header-flex { display: flex; justify-content: space-between; align-items: center; margin-bottom: 20px; }
    </style>
</head>
<body>

<div class="container">
    <div class="header-flex">
        <h2>⚡ Load Shedding Analytics</h2>
        <button onclick="exportCSV()">📥 Download CSV</button>
    </div>

    <div class="grid">
        <div class="card">
            <div id="liveStatus" class="status online">Loading...</div>
            <div class="stat-label" id="lastPingText">Connecting to database...</div>
        </div>
        <div class="card">
            <div class="stat-value" id="todayOutage">0 hrs 0 mins</div>
            <div class="stat-label">Total Outage Today</div>
        </div>
        <div class="card">
            <div class="stat-value" id="weekOutage">0 hrs 0 mins</div>
            <div class="stat-label">Total Outage (Last 7 Days)</div>
        </div>
    </div>

    <h3>Historical Outage Log</h3>
    <table>
        <thead>
            <tr>
                <th>Date</th>
                <th>Power Went Off</th>
                <th>Power Came Back</th>
                <th>Duration</th>
            </tr>
        </thead>
        <tbody id="outageTableBody">
            <tr><td colspan="4" style="text-align:center;">Fetching data...</td></tr>
        </tbody>
    </table>
</div>

<script>
    // REPLACE WITH YOUR ACTUAL FIREBASE URL
    const DB_BASE = "https://<YOUR-PROJECT-ID>.firebasedatabase.app";
    let globalDisplayOutages = []; 

    async function fetchAnalytics() {
        try {
            const [pingRes, outagesRes] = await Promise.all([
                fetch(`${DB_BASE}/status/last_ping.json`),
                fetch(`${DB_BASE}/outages.json`)
            ]);

            const lastPingUnix = await pingRes.json();
            const outagesData = await outagesRes.json();

            if (!lastPingUnix) return;

            processDashboard(lastPingUnix, outagesData);
        } catch (error) {
            console.error("Error fetching data:", error);
            document.getElementById('liveStatus').textContent = "CONNECTION ERROR";
            document.getElementById('liveStatus').className = "status offline";
        }
    }

    function processDashboard(lastPingUnix, outagesData) {
        const currentUnixSec = Math.floor(Date.now() / 1000);
        const gapSec = currentUnixSec - lastPingUnix;
        const isOnline = gapSec < 120; // 2 minutes grace period

        const statusEl = document.getElementById('liveStatus');
        if (isOnline) {
            statusEl.textContent = "POWER IS ON 💡";
            statusEl.className = "status online";
        } else {
            statusEl.textContent = "ACTIVE LOAD SHEDDING 🔌";
            statusEl.className = "status offline";
        }
        
        const lastPingDate = new Date(lastPingUnix * 1000);
        document.getElementById('lastPingText').textContent = "Last heartbeat: " + lastPingDate.toLocaleTimeString();

        let outagesArray = [];
        if (outagesData) {
            outagesArray = Object.values(outagesData).sort((a, b) => b.start - a.start);
        }

        if (!isOnline) {
            outagesArray.unshift({
                start: lastPingUnix,
                end: currentUnixSec,
                duration_sec: gapSec,
                ongoing: true
            });
        }

        globalDisplayOutages = outagesArray;

        let todaySec = 0;
        let weekSec = 0;
        
        const startOfTodaySec = new Date().setHours(0,0,0,0) / 1000;
        const startOfWeekSec = new Date(new Date().setDate(new Date().getDate() - 7)).getTime() / 1000;

        outagesArray.forEach(outage => {
            if (outage.start >= startOfTodaySec) todaySec += outage.duration_sec;
            if (outage.start >= startOfWeekSec) weekSec += outage.duration_sec;
        });

        document.getElementById('todayOutage').textContent = formatDuration(todaySec);
        document.getElementById('weekOutage').textContent = formatDuration(weekSec);

        const tbody = document.getElementById('outageTableBody');
        tbody.innerHTML = '';
        
        if (outagesArray.length === 0) {
            tbody.innerHTML = '<tr><td colspan="4" style="text-align:center;">No outages recorded yet!</td></tr>';
            return;
        }

        outagesArray.slice(0, 15).forEach(outage => {
            const tr = document.createElement('tr');
            const startDate = new Date(outage.start * 1000);
            const endDateText = outage.ongoing ? "<i>Ongoing...</i>" : new Date(outage.end * 1000).toLocaleTimeString();
            
            tr.innerHTML = `
                <td>${startDate.toLocaleDateString()}</td>
                <td>${startDate.toLocaleTimeString()}</td>
                <td>${endDateText}</td>
                <td style="font-weight:bold; color: #ef4444;">${formatDuration(outage.duration_sec)}</td>
            `;
            tbody.appendChild(tr);
        });
    }

    function formatDuration(totalSeconds) {
        let hours = Math.floor(totalSeconds / 3600);
        let minutes = Math.floor((totalSeconds % 3600) / 60);
        
        if (hours === 0) return `${minutes} mins`;
        return `${hours} hrs ${minutes} mins`;
    }

    function exportCSV() {
        if (globalDisplayOutages.length === 0) {
            alert("No data to export.");
            return;
        }

        let csvContent = "data:text/csv;charset=utf-8,Date,Time_Off,Time_On,Duration_Minutes\n";

        globalDisplayOutages.forEach(outage => {
            const startDate = new Date(outage.start * 1000);
            const dateStr = startDate.toLocaleDateString();
            const timeOffStr = startDate.toLocaleTimeString();
            const timeOnStr = outage.ongoing ? "Ongoing" : new Date(outage.end * 1000).toLocaleTimeString();
            const durationMins = Math.round(outage.duration_sec / 60); 
            
            csvContent += `"${dateStr}","${timeOffStr}","${timeOnStr}",${durationMins}\n`;
        });

        const encodedUri = encodeURI(csvContent);
        const link = document.createElement("a");
        link.setAttribute("href", encodedUri);
        link.setAttribute("download", "load_shedding_dataset.csv");
        document.body.appendChild(link);
        link.click();
        document.body.removeChild(link);
    }

    fetchAnalytics();
    setInterval(fetchAnalytics, 10000);
</script>
</body>
</html>

```

## ⚠️ Important Notes

* Remember to exclude any files containing your Wi-Fi credentials from your commits, or use placeholder strings before pushing to public repositories.
* Make sure your Wi-Fi router is connected to the direct mains line (not a UPS/Battery Backup) so the router's downtime aligns accurately with grid power cuts.

```

```
