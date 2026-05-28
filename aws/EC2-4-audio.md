4. EC2-4 — Servidor d'Àudio (Icecast2 + ffmpeg)
4.1 Instal·lació
bashsudo apt install icecast2 -y
sudo apt install ffmpeg -y
4.2 Configuració d'Icecast2
Fitxer: /etc/icecast2/icecast.xml
xml<source-password>hackme</source-password>
<relay-password>hackme</relay-password>
<admin-user>admin</admin-user>
<admin-password>ITB2026admin</admin-password>
<hostname>localhost</hostname>

⚠️ Important: La contrasenya original @ITB2026 causava errors 401/403 a ffmpeg perquè el caràcter @ és especial en URLs. Es va canviar a hackme.

bashsudo systemctl restart icecast2
sudo systemctl enable icecast2
4.3 Generació del Fitxer d'Àudio
bashsudo ffmpeg -y -f lavfi -i sine=frequency=440:duration=3600 /home/adminitb/audio.mp3
ParàmetreValorFormatMP3Freqüència de mostreig44100 HzBitrate64 kbpsCanalMonoDurada1 hora (3600 s)Ubicació/home/adminitb/audio.mp3

⚠️ Important: El fitxer s'ha ubicat a /home/adminitb/ i no a /tmp/ perquè /tmp es buida en reiniciar la instància.

4.4 Servei Systemd: radio.service
Fitxer: /etc/systemd/system/radio.service
ini[Unit]
Description=Radio Icecast Stream
After=icecast2.service

[Service]
ExecStart=/usr/bin/ffmpeg -re -i /home/adminitb/audio.mp3 -f mp3 icecast://source:hackme@localhost:8000/radio
Restart=always
User=root

[Install]
WantedBy=multi-user.target

⚠️ Important: User=root és necessari perquè l'usuari ubuntu no té permisos sobre /home/adminitb.

bashsudo systemctl daemon-reload
sudo systemctl enable radio.service
sudo systemctl start radio.service
sudo systemctl status radio.service
4.5 Explicació del Comandament ffmpeg
bashsudo ffmpeg -re -i /home/adminitb/audio.mp3 -f mp3 icecast://source:hackme@localhost:8000/radio
PartSignificatsudoExecuta amb permisos d'administradorffmpegEl programa d'àudio/vídeo-reLlegeix l'arxiu a velocitat real (1x), simula un stream en directe-i /home/adminitb/audio.mp3Fitxer d'entrada (input)-f mp3Format de sortida: MP3icecast://Protocol per enviar a Icecast2sourceUsuari de la font (sempre és "source" a Icecast)hackmeContrasenya de la font@localhostAdreça del servidor Icecast (a la mateixa màquina)8000Port on escolta Icecast2/radioNom del mountpoint (la URL on s'escolta el stream)
4.6 Accés al Stream
🎵 URL del stream: http://54.205.214.70:8000/radio
📊 Pàgina d'estat Icecast2: http://54.205.214.70:8000
4.7 Enviament de Logs al EC2-3
bashecho "*.* @172.31.18.93:514" | sudo tee /etc/rsyslog.d/remote.conf
sudo systemctl restart rsyslog

5. Proves d'Ample de Banda
bashsudo apt install speedtest-cli -y
speedtest-cli
speedtest-cli --share
Resultats
ProvaDownloadUploadLatènciaProva 11012.32 Mbit/s1080.33 Mbit/s2.062 msProva 21003.15 Mbit/s1041.59 Mbit/s1.098 msMitjana~1007 Mbit/s~1060 Mbit/s~1.58 ms
✅ Resultat: ACCEPTABLE

La infraestructura supera àmpliament els requisits per suportar serveis multimèdia. Un stream d'àudio a 64 kbps representa menys del 0.01% de l'ample de banda disponible.
