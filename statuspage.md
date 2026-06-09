Da vi bevidst har valgt den helt skrabede "slim"-version af `spotifyd` for at skåne din Pi Zero W, har systemet ikke adgang til de tunge kommunikations-plugins (D-Bus/MPRIS), der normalt sladrer til systemet om en sang er sat på pause.

Men vi kan meget nemt bygge en webside, der overvåger systemet live og viser de to allervigtigste ting:

1. Er Bluetooth-højtaleren fysisk forbundet lige nu?
2. Kører Spotify-tjenesten og er klar til at modtage musik?

Vi bygger det i **Python** med et mikroskopisk web-framework, der hedder **Flask**. Det er utrolig let og perfekt til en Pi Zero. Siden opdaterer sig selv hvert 5. sekund, så den fungerer som en live-monitor.

Her er opskriften:

### 1. Installer Flask

Først skal vi hente web-værktøjet til Python:

```bash
sudo apt update
sudo apt install python3-flask -y

```

### 2. Opret dit dashboard-script

Vi opretter en mappe til projektet og laver selve Python-filen:

```bash
mkdir -p ~/spotify-status
nano ~/spotify-status/app.py

```

Kopier hele denne kodeblok og sæt den ind. **Vigtigt:** Husk at rette `XX:XX:XX:XX:XX:XX` til din højtalers MAC-adresse i toppen af filen!

```python
from flask import Flask, render_template_string
import subprocess

app = Flask(__name__)

# --- INDSÆT DIN HØJTALERS MAC ADRESSE HER ---
MAC = "XX:XX:XX:XX:XX:XX"

HTML = """
<!DOCTYPE html>
<html>
<head>
    <title>Pi Lyd Status</title>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <meta http-equiv="refresh" content="5">
    <style>
        body { font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif; text-align: center; margin-top: 10vh; background: #121212; color: #ffffff; }
        .kort { font-size: 20px; padding: 25px; border-radius: 15px; display: inline-block; margin: 15px; width: 250px; box-shadow: 0 4px 6px rgba(0,0,0,0.3); }
        .titel { font-size: 14px; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 10px; opacity: 0.8; }
        .status { font-weight: bold; font-size: 24px; }
        .gron { background: linear-gradient(145deg, #1db954, #1aa34a); }
        .rod { background: linear-gradient(145deg, #e22134, #c91d2e); }
        .graa { background: linear-gradient(145deg, #333333, #222222); }
    </style>
</head>
<body>
    <h2 style="opacity: 0.8; margin-bottom: 40px;">🎵 Musik Dashboard</h2>
    
    <div class="kort {{ 'gron' if bt_forbundet else 'rod' }}">
        <div class="titel">Bluetooth Højtaler</div>
        <div class="status">{{ 'Forbundet' if bt_forbundet else 'Afbrudt' }}</div>
    </div>

    <div class="kort {{ 'gron' if sp_korrer else 'graa' }}">
        <div class="titel">Spotify Connect</div>
        <div class="status">{{ 'Klar' if sp_korrer else 'Ikke Aktiv' }}</div>
    </div>
</body>
</html>
"""

@app.route('/')
def status():
    # Tjek om Bluetooth højtaleren er forbundet
    bt_check = subprocess.run(['bluetoothctl', 'info', MAC], capture_output=True, text=True)
    bt_forbundet = "Connected: yes" in bt_check.stdout

    # Tjek om Spotifyd tjenesten kører i baggrunden
    sp_check = subprocess.run(['systemctl', 'is-active', 'spotifyd'], capture_output=True, text=True)
    sp_korrer = "active" in sp_check.stdout.strip()

    return render_template_string(HTML, bt_forbundet=bt_forbundet, sp_korrer=sp_korrer)

if __name__ == '__main__':
    # Kører på port 80 (standard HTTP port)
    app.run(host='0.0.0.0', port=80)

```

*(Gem med `Ctrl+O`, `Enter`, og luk med `Ctrl+X`).*

### 3. Gør websiden til en baggrundstjeneste

Ligesom med `spotifyd` vil vi gerne have, at websiden altid kører i baggrunden, når Pi'en tændes.

Opret en ny service-fil:

```bash
sudo nano /etc/systemd/system/spotify-status.service

```

Indsæt dette (vi kører den som `root`, da port 80 kræver administratorrettigheder):

```ini
[Unit]
Description=Spotify Dashboard Web Server
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/valbjorn/spotify-status
ExecStart=/usr/bin/python3 /home/valbjorn/spotify-status/app.py
Restart=always

[Install]
WantedBy=multi-user.target

```

*(Gem med `Ctrl+O`, `Enter`, og luk med `Ctrl+X`).*

### 4. Start din nye webside!

Nu skal vi bare aktivere og starte tjenesten:

```bash
sudo systemctl daemon-reload
sudo systemctl enable spotify-status.service
sudo systemctl start spotify-status.service

```

### Sådan ser du siden

Åbn browseren på din computer eller telefon, der er på samme netværk, og indtast Raspberry Pi'ens IP-adresse i adressefeltet (f.eks. `http://10.0.0.9`).

Du vil nu blive mødt af to tydelige, farvede kort, der lader dig følge med i præcis, hvad status er! Slukker du højtaleren, vil skærmen automatisk skifte til rød efter højst 5 sekunder.
