1. Introducció i Objectius
Aquest document recull la configuració, desplegament i proves realitzades per la Persona 2 del Projecte Transversal ASIXc1 de l'empresa fictícia InnovateTech.
L'objectiu principal és desplegar i configurar dos servidors EC2 a AWS:
ServidorRolServeiEC2-3Centralització de logsRsyslog (port 514 TCP/UDP)EC2-4Streaming d'àudioIcecast2 + ffmpeg (port 8000)
A més, s'han realitzat proves d'ample de banda per validar que la infraestructura suporta els serveis multimèdia.
2. Infraestructura AWS

2.1 Instàncies EC2
InstànciaIDIP PúblicaIP PrivadaRolEC2-3i-0875f0b0f58d8fe0d44.223.189.225172.31.18.93Logs (Rsyslog)EC2-4i-0ef328d9b36ca419854.205.214.70172.31.31.131Àudio (Icecast2)
![Speedtest](https://github.com/user-attachments/assets/8d8eb9a3-1553-498b-b8c6-afc4b4cebbe2)
2.2 Connexions SSH
bash# EC2-3 (Logs)
ssh -i '/home/juan.rodriguez.7e9/Baixades/PROYECTO_TRANSVERSAL.pem' ubuntu@44.223.189.225
# EC2-4 (Àudio)
ssh -i '/home/juan.rodriguez.7e9/Baixades/PROYECTO_TRANSVERSAL.pem' ubuntu@54.205.214.70
2.3 Security Groups
EC2-3 — launch-wizard-1 (sg-logs)
TipusProtocolPortOrigenTot el tràficTotTot172.31.16.0/20SSHTCP220.0.0.0/0TCP personalitzatTCP5140.0.0.0/0UDP personalitzatUDP5140.0.0.0/0
EC2-4 — SGaudio (sg-audio)
TipusProtocolPortOrigenSSHTCP220.0.0.0/0TCP personalitzatTCP80000.0.0.0/0

3. EC2-3 — Servidor de Logs (Rsyslog)
3.1 Instal·lació
bashsudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
3.2 Configuració
/etc/rsyslog.conf — Activar escolta TCP i UDP
Descomenta (o afegeix) aquestes línies:
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")

⚠️ Important: Els mòduls imudp i imtcp s'han de declarar només a rsyslog.conf. Si es declaren també a remote.conf causa errors d'inici del servei.

/etc/rsyslog.d/remote.conf — Desar logs per hostname
bashsudo nano /etc/rsyslog.d/remote.conf
$template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
*.* ?RemoteLogs
Directori de logs
bashsudo mkdir -p /var/log/remote
sudo chown syslog:syslog /var/log/remote
sudo systemctl restart rsyslog

⚠️ Important: El directori ha de pertànyer a syslog:syslog, sinó els logs no s'escriuen.

3.3 Verificació
bash# Comprovar que escolta al port 514
sudo ss -tulnp | grep 514

# Veure els logs en temps real
sudo tail -f /var/log/remote/*.log

# Llistar fitxers de log rebuts
ls /var/log/remote/

# Llegir un log concret
sudo cat /var/log/remote/ip-172-31-31-131.log
Resultat esperat de ss -tulnp | grep 514:
udp   UNCONN  0.0.0.0:514    users:(("rsyslogd",...))
tcp   LISTEN  0.0.0.0:514    users:(("rsyslogd",...))
3.4 Configuració als servidors client (EC2-4 i companys)
Cada servidor que vulgui enviar logs ha d'executar:
bashecho "*.* @172.31.18.93:514" | sudo tee /etc/rsyslog.d/remote.conf
sudo systemctl restart rsyslog

# Generar un log de prova
logger "prova de log des d'aquest servidor"
3.5 Logs rebuts
FitxerServidor origenEstatip-172-31-18-93.logEC2-3 (local)✅ Actiuip-172-31-31-131.logEC2-4 (àudio)✅ Actiuip-172-31-31-131.ec2.internal.logEC2-4 (àudio)✅ Actiu
