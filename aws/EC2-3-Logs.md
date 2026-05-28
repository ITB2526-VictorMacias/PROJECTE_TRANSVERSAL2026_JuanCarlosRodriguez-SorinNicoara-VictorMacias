## 1. Introducció i Objectius

Aquest document recull la configuració, desplegament i proves realitzades per la Persona 2 del Projecte Transversal ASIXc1 de l'empresa fictícia InnovateTech.

L'objectiu principal és desplegar i configurar dos servidors EC2 a AWS:

- EC2-3: Centralització de logs amb Rsyslog (port 514 TCP/UDP)
- EC2-4: Streaming d'àudio amb Icecast2 + ffmpeg (port 8000)

A més, s'han realitzat proves d'ample de banda per validar que la infraestructura suporta els serveis multimèdia.

---

## 2. Infraestructura AWS

### 2.1 Instàncies EC2

EC2-3 — `i-0875f0b0f58d8fe0d` — IP pública 44.223.189.225 — IP privada 172.31.18.93 — Logs (Rsyslog)

EC2-4 — `i-0ef328d9b36ca4198` — IP pública 54.205.214.70 — IP privada 172.31.31.131 — Àudio (Icecast2)

![Speedtest](https://github.com/user-attachments/assets/8d8eb9a3-1553-498b-b8c6-afc4b4cebbe2)

### 2.2 Connexions SSH

```bash
# EC2-3 (Logs)
ssh -i '/home/juan.rodriguez.7e9/Baixades/PROYECTO_TRANSVERSAL.pem' ubuntu@44.223.189.225

# EC2-4 (Àudio)
ssh -i '/home/juan.rodriguez.7e9/Baixades/PROYECTO_TRANSVERSAL.pem' ubuntu@54.205.214.70
```

### 2.3 Security Groups

EC2-3 — `launch-wizard-1` (sg-logs): permet tot el tràfic intern (172.31.16.0/20), SSH (22) i els ports 514 TCP/UDP oberts a 0.0.0.0/0.

EC2-4 — `SGaudio` (sg-audio): permet SSH (22) i el port 8000 TCP obert a 0.0.0.0/0.

<img width="1739" height="165" alt="image" src="https://github.com/user-attachments/assets/a749e07e-8d88-4990-9867-cf55d49ac0ab" />

---

## 3. EC2-3 — Servidor de Logs (Rsyslog)

### 3.1 Instal·lació

```bash
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog

<img width="851" height="301" alt="image" src="https://github.com/user-attachments/assets/a18f6aff-75d1-4c9f-9c41-6a6d65d74bf7" />

```

### 3.2 Configuració

`/etc/rsyslog.conf` — activar escolta TCP i UDP. Descomenta o afegeix:

```
module(load="imudp")
input(type="imudp" port="514")

module(load="imtcp")
input(type="imtcp" port="514")
```

⚠️ Els mòduls imudp i imtcp s'han de declarar només a rsyslog.conf. Si es declaren també a remote.conf causa errors d'inici del servei.

`/etc/rsyslog.d/remote.conf` — desar logs per hostname:

```bash
sudo nano /etc/rsyslog.d/remote.conf

```
```
$template RemoteLogs,"/var/log/remote/%HOSTNAME%.log"
*.* ?RemoteLogs
```
<img width="748" height="89" alt="image" src="https://github.com/user-attachments/assets/2fbc49db-e779-45f3-b099-2a6213241440" />
Directori de logs:

```bash
sudo mkdir -p /var/log/remote
sudo chown syslog:syslog /var/log/remote
sudo systemctl restart rsyslog
```

⚠️ El directori ha de pertànyer a syslog:syslog, sinó els logs no s'escriuen.

### 3.3 Verificació

```bash
# Comprovar que escolta al port 514
sudo ss -tulnp | grep 514
```
<img width="748" height="154" alt="image" src="https://github.com/user-attachments/assets/cb351f12-9bc5-426f-b126-6757eec57fc5" />
```
# Veure els logs en temps real
sudo tail -f /var/log/remote/*.log
```
<img width="946" height="537" alt="image" src="https://github.com/user-attachments/assets/632e2a6f-71ca-474b-812c-6e41b02eacb6" />

```
# Llistar fitxers de log rebuts
ls /var/log/remote/
```
<img width="566" height="45" alt="image" src="https://github.com/user-attachments/assets/ed8faaf4-c661-4dc8-9a38-1ea0d2357d5b" />

```
# Llegir un log concret
sudo cat /var/log/remote/ip-172-31-31-131.log
```
<img width="943" height="734" alt="image" src="https://github.com/user-attachments/assets/6371eb7c-e95c-4952-b336-f70d4fb9c17d" />

**Resultat esperat de `ss -tulnp | grep 514`:**

```
udp   UNCONN  0.0.0.0:514    users:(("rsyslogd",...))
tcp   LISTEN  0.0.0.0:514    users:(("rsyslogd",...))
```

### 3.4 Configuració als servidors client (EC2-4 i companys)

Cada servidor que vulgui enviar logs ha d'executar:

```bash
echo "*.* @172.31.18.93:514" | sudo tee /etc/rsyslog.d/remote.conf
sudo systemctl restart rsyslog

# Generar un log de prova
logger "prova de log des d'aquest servidor"
```
<img width="551" height="14" alt="image" src="https://github.com/user-attachments/assets/0c74463e-ac49-414b-ade8-94dc887adcb2" />

<img width="631" height="28" alt="image" src="https://github.com/user-attachments/assets/f66b9cd7-839c-4c7e-b26c-9ce8f03e3f34" />

### 3.5 Logs rebuts

- `ip-172-31-18-93.log` — EC2-3 (local) ✅
- `ip-172-31-31-131.log` — EC2-4 (àudio) ✅
- `ip-172-31-31-131.ec2.internal.log` — EC2-4 (àudio) ✅
