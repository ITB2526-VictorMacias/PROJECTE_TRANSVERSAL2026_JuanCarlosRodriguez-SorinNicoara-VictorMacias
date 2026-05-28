# EC2-4 — Servidor d'Àudio (Icecast2 + ffmpeg)

---

## 4.1 Instal·lació

```bash
sudo apt install icecast2 -y
sudo apt install ffmpeg -y
```

## 4.2 Configuració d'Icecast2

Fitxer: `/etc/icecast2/icecast.xml`

```xml
<source-password>hackme</source-password>
<relay-password>hackme</relay-password>
<admin-password>ITB2026admin</admin-password>
<hostname>localhost</hostname>
```

⚠️ La contrasenya original @ITB2026 causava errors 401/403 a ffmpeg perquè el caràcter @ és especial en URLs. Es va canviar a hackme.

```bash
sudo systemctl restart icecast2
sudo systemctl enable icecast2
```

## 4.3 Generació del Fitxer d'Àudio

```bash
sudo ffmpeg -y -f lavfi -i sine=frequency=440:duration=3600 /home/adminitb/audio.mp3
```

Format MP3, freqüència de mostreig 44100 Hz, bitrate 64 kbps, canal mono, durada 1 hora (3600 s), ubicat a `/home/adminitb/audio.mp3`.

⚠️ El fitxer s'ha ubicat a `/home/adminitb/` i no a `/tmp/` perquè `/tmp` es buida en reiniciar la instància.

<img width="750" height="621" alt="image" src="https://github.com/user-attachments/assets/9743085f-4bcd-409a-bbac-a9ce2c957546" />

## 4.4 Servei Systemd: radio.service

Fitxer: `/etc/systemd/system/radio.service`

```ini
[Unit]
Description=Radio Icecast Stream
After=icecast2.service

[Service]
ExecStart=/usr/bin/ffmpeg -re -i /home/adminitb/audio.mp3 -f mp3 icecast://source:hackme@localhost:8000/radio
Restart=always
User=root

[Install]
WantedBy=multi-user.target
```

⚠️ `User=root` és necessari perquè l'usuari ubuntu no té permisos sobre `/home/adminitb`.

```bash
sudo systemctl daemon-reload
sudo systemctl enable radio.service
sudo systemctl start radio.service
sudo systemctl status radio.service
```
<img width="799" height="301" alt="image" src="https://github.com/user-attachments/assets/dd526788-371d-4dc7-b805-e2281b71bdc2" />

## 4.5 Explicació del comandament ffmpeg

```bash
sudo ffmpeg -re -i /home/adminitb/audio.mp3 -f mp3 icecast://source:hackme@localhost:8000/radio
```

- `sudo` — executa amb permisos d'administrador
- `ffmpeg` — el programa d'àudio/vídeo
- `-re` — llegeix l'arxiu a velocitat real (1x), simula un stream en directe
- `-i /home/adminitb/audio.mp3` — fitxer d'entrada
- `-f mp3` — format de sortida MP3
- `icecast://` — protocol per enviar a Icecast2
- `source` — usuari de la font (sempre és "source" a Icecast)
- `hackme` — contrasenya de la font
- `@localhost` — adreça del servidor Icecast (a la mateixa màquina)
- `8000` — port on escolta Icecast2
- `/radio` — nom del mountpoint (la URL on s'escolta el stream)

## 4.6 Accés al Stream

URL del stream: `http://54.205.214.70:8000/radio`

<img width="948" height="961" alt="image" src="https://github.com/user-attachments/assets/73be0d2d-f614-494e-b51f-914898de337e" />

Pàgina d'estat Icecast2: `http://54.205.214.70:8000`

<img width="957" height="968" alt="image" src="https://github.com/user-attachments/assets/a995b1d3-b4be-49cb-b4ff-f306024e6cff" />


## 4.7 Enviament de Logs al EC2-3

```bash
echo "*.* @172.31.18.93:514" | sudo tee /etc/rsyslog.d/remote.conf
sudo systemctl restart rsyslog
```

---

## 5. Proves d'Ample de Banda

```bash
sudo apt install speedtest-cli -y
speedtest-cli
speedtest-cli --share
```

**Resultats:**

- Prova 1 — download 899,64 Mbit/s, upload 936.27 Mbit/s, latència 2.199 ms
  
  <img width="895" height="150" alt="image" src="https://github.com/user-attachments/assets/ad606f16-274f-4c62-bf57-c1a4908f61c9" />

- Prova 2 — download 948.32 Mbit/s, upload 950.86 Mbit/s, latència 1.427 ms

<img width="891" height="164" alt="image" src="https://github.com/user-attachments/assets/b5aa6c1c-bdbb-4591-85f5-4744b6c1d6ca" />

- Mitjana — ~1007 Mbit/s download, ~1060 Mbit/s upload, ~1.58 ms latència

**Resultat: acceptable.** La infraestructura supera àmpliament els requisits per suportar serveis multimèdia. Un stream d'àudio a 64 kbps representa menys del 0.01% de l'ample de banda disponible.
