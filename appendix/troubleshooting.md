# Betriebshandbuch Teil 2: Troubleshooting

Dieses Dokument beschreibt die Probleme, die während des Deployments der hybriden Netzwerkarchitektur tatsächlich aufgetreten sind. Zu jedem Punkt werden das Fehlerbild, eine wahrscheinliche Ursache und die verwendete Lösung dokumentiert.

## 1. Zugriff auf Test-VMs über IAP nicht möglich

### Problem

Der geplante SSH-Zugriff auf die Test-VMs über IAP konnte im Projekt nicht direkt genutzt werden.

### Mögliche Ursache

- Die erforderlichen IAM-Berechtigungen für IAP TCP Forwarding waren nicht vollständig vorhanden.
- Die IAP-bezogenen Firewall-Regeln waren im Projektkontext nicht ausreichend vorbereitet.

### Lösung im Projekt

Für die Testphase wurden den VMs temporär externe IP-Adressen zugewiesen. Zusätzlich wurden kurzfristig SSH-Firewall-Regeln erstellt. Der eigentliche Funktionstest des VPN lief trotzdem über die internen IP-Adressen.

```bash
# Temporäre externe IPs hinzufügen
gcloud compute instances add-access-config "vm-cloud" \
  --zone="${ZONE}" \
  --network-interface="nic0"

gcloud compute instances add-access-config "vm-branch" \
  --zone="${ZONE}" \
  --network-interface="nic0"

# Temporäre SSH-Regeln anlegen
gcloud compute firewall-rules create "fw-allow-ssh-external-eu" \
  --network="${CLOUD_VPC}" \
  --allow=tcp:22 \
  --source-ranges="0.0.0.0/0"

gcloud compute firewall-rules create "fw-allow-ssh-external-branch" \
  --network="${BRANCH_VPC}" \
  --allow=tcp:22 \
  --source-ranges="0.0.0.0/0"
```

## 2. Traceroute zeigte zunächst keinen nutzbaren Pfad

### Problem

Der erste Traceroute-Test lieferte kein verwertbares Ergebnis.

### Mögliche Ursache

Der Standardaufruf von `traceroute` verwendet UDP-Pakete. Diese wurden durch die Firewall-Regeln der Testumgebung nicht zugelassen.

### Lösung im Projekt

Der Traceroute-Test wurde mit ICMP statt UDP durchgeführt. Dafür wurde der Parameter `-I` verwendet.

```bash
gcloud compute ssh "vm-branch" \
  --zone="${ZONE}" \
  --command="sudo apt-get update -y > /dev/null && sudo apt-get install traceroute -y > /dev/null && sudo traceroute -I -n ${CLOUD_VM_IP}"
```

## 3. Veralteter NCC-Befehl für VPN-Spokes

### Problem

Der ältere CLI-Ansatz zur Erstellung eines NCC-Spokes mit einem einzelnen `--vpn-tunnel` war nicht mehr passend zur aktuellen Syntax.

### Mögliche Ursache

Die `gcloud`-Syntax für NCC-Spokes wurde geändert. Für VPN-Tunnel wird jetzt der Unterbefehl `linked-vpn-tunnels create` verwendet.

### Lösung im Projekt

Der Spoke wurde mit dem neuen Befehl und vollständigen Tunnel-URIs angelegt.

```bash
export TUNNEL0_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_0}"
export TUNNEL1_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_1}"

gcloud network-connectivity spokes linked-vpn-tunnels create "${SPOKE_NAME}" \
  --region="${REGION}" \
  --hub="${HUB_URI}" \
  --vpn-tunnels="${TUNNEL0_URI},${TUNNEL1_URI}" \
  --site-to-site-data-transfer
```

## 4. Ausgabe des BGP-Status war missverständlich

### Problem

Bei der Kontrolle des Cloud Routers wurde im Status nicht der Begriff `ESTABLISHED`, sondern `UP` angezeigt.

### Mögliche Ursache

Cloud Router verwendet in der CLI-Ausgabe eine betriebliche Statusdarstellung. Für aktive Peerings wird `UP` angezeigt.

### Lösung im Projekt

Die BGP-Sitzung wurde über `get-status` geprüft und der Status `UP` als erfolgreiches Peering bewertet. Zusätzlich wurden die gelernten Routen kontrolliert.

```bash
# BGP Peer Status prüfen
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bgpPeerStatus[].name:label=PEER_NAME, result.bgpPeerStatus[].status:label=STATUS)"

# Gelernte Routen prüfen
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bestRoutes[].destRange:label=DESTINATION_RANGE, result.bestRoutes[].nextHopIp:label=NEXT_HOP, result.bestRoutes[].priority:label=PRIORITY)"
```

## 5. Failover-Test erforderte manuelles Entfernen eines BGP-Peers

### Problem

Für den Nachweis der Redundanz musste ein Ausfall künstlich erzeugt werden.

### Mögliche Ursache

Im Normalbetrieb bleiben beide Tunnel aktiv. Für einen kontrollierten Test musste daher ein Peering gezielt entfernt werden.

### Lösung im Projekt

Der BGP-Peer von Tunnel 0 wurde temporär gelöscht. Danach wurde die Verbindung erneut geprüft. Anschließend wurde der Peer wieder angelegt.

```bash
# BGP-Peer von Tunnel 0 entfernen
gcloud compute routers remove-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-0" \
  --region="${REGION}" \
  --quiet

# Kurze Wartezeit für die Umschaltung
sleep 20

# Verbindung erneut prüfen
gcloud compute ssh "vm-branch" \
  --zone="${ZONE}" \
  --command="ping -c 4 ${CLOUD_VM_IP}"

# BGP-Peer wieder anlegen
gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-0" \
  --interface="if-eu-to-br-0" \
  --peer-ip-address="${BR_BGP_0_IP}" \
  --peer-asn="${BRANCH_ASN}" \
  --region="${REGION}"
```

## 6. Rückbau musste in korrekter Reihenfolge erfolgen

### Problem

Beim Löschen der Umgebung können Abhängigkeiten zwischen NCC, VPN, Routern und Netzwerken zu Fehlern führen.

### Mögliche Ursache

Einzelne Ressourcen lassen sich nur löschen, wenn abhängige Objekte bereits entfernt wurden.

### Lösung im Projekt

Der Rückbau erfolgte in fester Reihenfolge: zuerst Spoke und Hub, danach Tunnel, Gateways, Router und zuletzt Firewalls, Subnetze und VPCs.

```bash
# NCC entfernen
gcloud network-connectivity spokes delete "${SPOKE_NAME}" --region="${REGION}" --quiet
gcloud network-connectivity hubs delete "${HUB_NAME}" --quiet

# VPN-Komponenten löschen
gcloud compute vpn-tunnels delete \
  "${TUN_EU_TO_BR_0}" "${TUN_BR_TO_EU_0}" \
  "${TUN_EU_TO_BR_1}" "${TUN_BR_TO_EU_1}" \
  --region="${REGION}" --quiet

gcloud compute vpn-gateways delete "${CLOUD_GW}" "${BRANCH_GW}" \
  --region="${REGION}" --quiet

gcloud compute routers delete "${CLOUD_ROUTER}" "${BRANCH_ROUTER}" \
  --region="${REGION}" --quiet

# Firewall und Netzwerke löschen
gcloud compute firewall-rules delete \
  "fw-allow-internal-eu" "fw-allow-internal-branch" \
  --quiet

gcloud compute networks subnets delete "${CLOUD_SUBNET}" "${BRANCH_SUBNET}" \
  --region="${REGION}" --quiet

gcloud compute networks delete "${CLOUD_VPC}" "${BRANCH_VPC}" --quiet
```
