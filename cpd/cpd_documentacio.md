# Proposta de CPD - InnovateTech

## 1. Ubicació Física

### Situació física de la sala
La sala del CPD està situada a la planta baixa de l'edifici, en una zona interior sense finestres per dificultar la seva identificació des de l'exterior. No hi ha senyalització visible que indiqui que es tracta d'un CPD. L'accés és restringit i només el personal autoritzat pot accedir-hi mitjançant control d'accés.

### Sistemes de climatització
- Temperatura mantinguda entre **18°C i 27°C** mitjançant sistemes d'aire condicionat de precisió amb redundància (si un falla, l'altre manté la temperatura).
- Humitat relativa entre **40% i 60%**.
- Filtres de partícules per mantenir l'aire net i lliure de pols.
- Flux d'aire fred per sota (terra tècnic) i retorn d'aire calent per dalt (sostre tècnic).

### Mesures per dificultar la identificació de la sala
- Sense senyalització exterior visible.
- Porta de la sala igual que les altres portes de l'edifici, sense distintius.
- Situada en una zona de pas intern, no accessible al públic general.

### Distribució i gestió del cablejat
- Cableado estructurat sota terra tècnic per separar el cablejat elèctric del de dades.
- Safates de cables etiquetades i organitzades per colors:
  - **Vermell**: Elèctric
  - **Blau**: Xarxa de dades
  - **Groc**: Fibra òptica
- Tots els cables etiquetats als dos extrems per facilitar el manteniment.

### Terra tècnic i sostre tècnic
- **Terra tècnic**: Elevat 30cm per al pas de cables i refrigeració per terra.
- **Sostre tècnic**: Per al retorn d'aire calent i pas de cablejat elèctric.

### Estructuració dels racks (mínim 2 racks)

**Rack 1 - Servidors:**
| Posició (U) | Element |
|-------------|---------|
| 1-2 | Patch panel 24 ports |
| 3-4 | Switch gestionable 24 ports |
| 5-10 | Servidor 1 (Web + SFTP) |
| 11-16 | Servidor 2 (LDAP) |
| 17-22 | Servidor 3 (Logs centralitzats) |
| 23-28 | Servidor 4 (Àudio) |
| 29-34 | Servidor 5 (Vídeo + Jitsi) |
| 35-40 | Servidor 6 (Base de dades) |
| 41-42 | Gestió de cables |

**Rack 2 - Xarxa i seguretat:**
| Posició (U) | Element |
|-------------|---------|
| 1-2 | Patch panel 24 ports |
| 3-4 | Switch gestionable 24 ports (redundància) |
| 5-6 | Firewall perimetral |
| 7-8 | Router |
| 9-42 | SAI (APC Smart-UPS 3000VA) |

---

## 2. Infraestructura IT

### Servidors
- **Quantitat**: 6 servidors físics (equivalents als 6 EC2 del projecte)
- **Model**: Dell PowerEdge R740
- **Especificacions per servidor**:
  - CPU: Intel Xeon Silver 4210 (8 cores, 2.2GHz)
  - RAM: 32GB DDR4 ECC
  - Disc: 2x 1TB SSD en RAID 1
  - Xarxa: 2x ports Gigabit Ethernet (redundància)

### Patch panels
- 2 patch panels de 24 ports (un per rack)
- Etiquetats i documentats per facilitar el manteniment
- Connexió directa als switches mitjançant latiguillos de categoria 6A

### Switches
- 2 switches gestionables de 24 ports (Cisco Catalyst 2960)
- Configurats en VLAN per separar el tràfic de xarxa:
  - VLAN 10: Servidors
  - VLAN 20: Administració
  - VLAN 30: Gestió

---

## 3. Infraestructura Elèctrica - SAI

### Sistemes d'alimentació redundant
- Doble línia elèctrica independent des de dos quadres elèctrics diferents.
- Cada servidor amb dues fonts d'alimentació redundants.

### Càlcul del SAI

| Element | Consum estimat |
|---------|---------------|
| 6 servidors × 500W | 3.000W |
| 2 switches × 50W | 100W |
| 2 patch panels × 10W | 20W |
| **Total** | **3.120W** |

- **Temps desitjat sense corrent**: 30 minuts (temps suficient per a apagada controlada)
- **Energia necessària**: 3.120W × 0,5h = **1.560 Wh**
- **SAI seleccionat**: APC Smart-UPS 3000VA (amb marge de seguretat del 30%)
- **Model**: APC Smart-UPS SRT 3000VA
- El SAI inclou gestió remota per monitoritzar l'estat de la bateria en temps real.

---

## 4. Seguretat Física i Lògica

### Seguretat Física

#### Control d'accés
- Portes amb lector de targeta RFID + PIN per doble factor d'autenticació.
- Registre automàtic de totes les entrades i sortides amb data, hora i usuari.
- Accés restringit únicament al personal autoritzat.
- Procediment de revocació immediata d'accés en cas de baixa del personal.

#### Videovigilància
- Càmeres IP d'alta resolució a l'entrada, interior i passadissos.
- Gravació contínua 24/7 amb retenció mínima de 30 dies.
- Monitorització en temps real des del centre de seguretat.
- Sistema d'alertes automàtiques davant moviment fora d'horari.

#### Sistemes de prevenció, detecció i extinció d'incendis
- Detectors de fum i calor distribuïts per tota la sala.
- Sistema d'extinció per gas FM-200 (no danya els equips electrònics).
- Alarma sonora i visual en cas de detecció d'incendi.
- Revisió semestral de tots els sistemes contra incendis.

#### Vies d'evacuació
- Senyalització lluminosa d'emergència en totes les sortides.
- Porta antipànic a la sortida principal.
- Pla d'evacuació visible i actualitzat a l'entrada de la sala.
- Simulacres d'evacuació anuals.

### Seguretat Lògica

#### Restricció d'accés per autorització
- Gestió centralitzada d'usuaris mitjançant **OpenLDAP** (EC2-2).
- Rols diferenciats: admin, vendes, administració, treballador.
- Política de contrasenyes: mínim 8 caràcters, majúscules, números i símbols.
- Revisió trimestral dels permisos d'accés.

#### Firewalls
- **AWS Security Groups**: control de tràfic entrant i sortint per a cada EC2.
- **Firewall perimetral físic**: protecció de la xarxa interna.
- Política de mínim privilegi: només s'obren els ports estrictament necessaris.

#### Monitorització
- Servidor centralitzat de logs (EC2-3) que recull els logs de tots els servidors.
- Monitorització de disponibilitat i rendiment dels servidors.
- Alertes automàtiques davant caigudes o comportaments anòmals.

#### Còpies de seguretat / Backups
- Backups diaris automatitzats mitjançant events periòdics a la base de dades.
- Còpies emmagatzemades en ubicació separada dels servidors originals.
- Verificació mensual de la integritat de les còpies de seguretat.

#### RAIDs
- Tots els servidors amb **RAID 1** (mirroring) per redundància de dades.
- En cas de fallada d'un disc, el servidor continua funcionant sense interrupció.

### Prevenció de Riscos Laborals
- Passadissos lliures d'obstacles per facilitar l'evacuació.
- Senyalització de riscos elèctrics en tots els quadres i servidors.
- EPI disponibles a l'entrada: guants aïllants, calçat de seguretat, ulleres.
- Formació obligatòria del personal en riscos elèctrics i d'incendi.
- Prohibit menjar i beure dins la sala del CPD.
- Il·luminació adequada (mínim 500 lux) per facilitar el treball segur.

---

## 5. Implementació al Núvol AWS

### Serveis desplegats

| EC2 | Servei | Configurat amb Ansible |
|-----|--------|----------------------|
| EC2-1 | Servidor Web (Apache) + SFTP | ✅ Sí |
| EC2-2 | Directori Actiu (OpenLDAP) | ✅ Sí |
| EC2-3 | Centralització de logs | ❌ (P2) |
| EC2-4 | Streaming d'àudio (Icecast) | ❌ (P2) |
| EC2-5 | Streaming de vídeo + Jitsi Meet | ❌ (P3) |
| EC2-6 | Base de dades MySQL | ❌ (P3) |

### Configuració de seguretat AWS
- Cada EC2 amb el seu propi Security Group amb el mínim de ports oberts.
- Accés SSH únicament amb clau pública/privada, sense contrasenyes.
- Usuari d'administració específic `adminitb` (no s'utilitza l'usuari per defecte).

### Ansible
- Les màquines EC2-1 i EC2-2 estan configurades completament amb Ansible.
- Playbook `setup.yml` que automatitza la instal·lació i configuració de tots els serveis.
- Inventari `inventory.ini` amb totes les màquines del projecte.

---

## 6. Bloc 1665

### RA3 - Tecnologies habilitadores digitals
AWS és la tecnologia habilitadora digital triada per a InnovateTech perquè permet desplegar una infraestructura completa al núvol sense necessitat d'inversió inicial en maquinari físic. Les seves característiques principals són l'escalabilitat (es poden afegir recursos en minuts), l'alta disponibilitat (SLA del 99,99%), la seguretat (certificacions ISO 27001, SOC 2) i la sostenibilitat (centres de dades alimentats per energies renovables).

### RA5 - Importància i protecció de les dades
Les dades gestionades per InnovateTech (dades d'empleats, clients, trucades i vídeos) són actius crítics de l'empresa. La seva pèrdua o exposició podria causar danys econòmics, legals i reputacionals.

**Mesures de protecció implementades:**
- **Seguretat física**: CPD amb accés restringit, videovigilància i extinció d'incendis.
- **Seguretat lògica**: LDAP, firewalls, backups automàtics, RAIDs i triggers d'auditoria.
- **Compliment normatiu**: Les mesures implementades s'alineen amb el **RGPD** (Reglament General de Protecció de Dades) i la normativa ISO 27001.

### RA6 - Transformació digital d'InnovateTech
InnovateTech ha passat d'una infraestructura tradicional on-premise a una solució completament al núvol AWS. Aquest canvi implica:

**Canvis tecnològics:**
- Substitució de servidors físics per instàncies EC2 al núvol.
- Gestió d'usuaris centralitzada amb LDAP en lloc de gestió local.
- Automatització de la configuració amb Ansible.
- Monitorització centralitzada de logs.

**Canvis organitzatius:**
- El personal d'IT passa de gestionar maquinari físic a gestionar infraestructura al núvol.
- Reducció del temps de desplegament de nous serveis (de setmanes a minuts).
- Possibilitat de treball remot gràcies als serveis al núvol.

**Beneficis obtinguts:**
- **Reducció de costos**: Eliminació de la inversió inicial en maquinari.
- **Escalabilitat**: Capacitat d'augmentar recursos en funció de la demanda.
- **Disponibilitat**: Serveis accessibles 24/7 des de qualsevol lloc.
- **Sostenibilitat**: Reducció de la petjada ecològica gràcies als centres de dades eficients d'AWS.
