# Betriebshandbuch Teil 2: Troubleshooting

Dieses Dokument beschreibt die spezifischen Probleme und Herausforderungen, die während des Deployments der hybriden Netzwerkarchitektur aufgetreten sind.

## 1. Blockierter IAP-Zugriff auf Test-VMs

### Fehlerbild
Der geplante administrative SSH-Zugriff auf die Test-VMs über den Identity-Aware Proxy (IAP) schlug fehl.

### Ursachenanalyse
- Die erforderlichen IAM-Rollen (z. B. *IAP-secured Tunnel User*) waren im Projektkontext nicht vollständig vergeben.
- Die zugehörigen IAP-Firewall-Regeln waren noch nicht final provisioniert.

### Workaround im Projekt
Um den Projektzeitplan einzuhalten, erhielten die Test-VMs für die Validierungsphase temporär externe IP-Adressen. Zusätzlich wurden kurzfristig SSH-Firewall-Regeln erstellt. Die eigentliche VPN-Routing-Prüfung lief davon unabhängig weiterhin über die internen Adressen.

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

## 2. Abgewiesene Traceroute-Pakete (UDP-Drop)

### Fehlerbild
Der initiale Traceroute-Test zwischen den VMs zeigte nur Timeouts und lieferte keine nutzbaren Routing-Informationen.

### Ursachenanalyse
Der Standardaufruf von `traceroute` unter Linux sendet UDP-Pakete. Das strikte *Default-Deny*-Regelwerk der Testumgebung blockierte diesen unerwarteten UDP-Traffic.

### Lösungsansatz
Der Traceroute-Aufruf wurde mit dem Parameter `-I` auf ICMP-Pakete umgestellt, die in der Firewall explizit für Diagnosezwecke freigegeben waren.

```bash
gcloud compute ssh "vm-branch" \
  --zone="${ZONE}" \
  --command="sudo apt-get update -y > /dev/null && sudo apt-get install traceroute -y > /dev/null && sudo traceroute -I -n ${CLOUD_VM_IP}"
```

## 3. Veraltete gcloud-Syntax für NCC-Spokes

### Fehlerbild
Die Erstellung des VPN-Spokes schlug mit einem Syntax-Fehler in der CLI fehl.

### Ursachenanalyse
Google Cloud hat die Syntax für das Network Connectivity Center aktualisiert. Der ältere Befehl mit einem einzelnen `--vpn-tunnel`-Parameter war nicht mehr passend zur aktuellen API.

### Lösungsansatz
Der Spoke wurde mit dem aktuellen Unterbefehl `linked-vpn-tunnels create` und den vollständigen Tunnel-URIs angelegt.

```bash
export TUNNEL0_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_0}"
export TUNNEL1_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_1}"

gcloud network-connectivity spokes linked-vpn-tunnels create "${SPOKE_NAME}" \
  --region="${REGION}" \
  --hub="${HUB_URI}" \
  --vpn-tunnels="${TUNNEL0_URI},${TUNNEL1_URI}" \
  --site-to-site-data-transfer
```

## 4. Abweichende BGP-Statusausgabe in der CLI

### Fehlerbild
Bei der Statusprüfung des Cloud Routers via CLI wurde nicht der erwartete BGP-Zustand `ESTABLISHED`, sondern `UP` angezeigt.

### Ursachenanalyse
Die Google Cloud CLI (`gcloud`) verwendet für BGP-Sitzungen eine vereinfachte, betriebliche Statusdarstellung. Ein aktives und funktionierendes Peering wird hierbei als `UP` deklariert.

### Lösungsansatz
Die BGP-Sitzung wurde über `get-status` geprüft und der Status `UP` als erfolgreiches Peering bewertet. Zur finalen Verifikation wurden zusätzlich die gelernten Routen ausgelesen.

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

## 5. Simulation des Failover-Szenarios (BGP Peer Down)

### Fehlerbild
Für den Nachweis der Ausfallsicherheit (Redundanz) musste das System zum Wechsel auf den zweiten Tunnel gezwungen werden.

### Ursachenanalyse
Da im Normalbetrieb beide IPsec-Tunnel aktiv sind (Active/Active), musste für einen kontrollierten Ausfalltest ein Peering gezielt unterbrochen werden.

### Lösungsansatz
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

## 6. Ressourcen-Abhängigkeiten beim Teardown

### Fehlerbild
Beim Löschen der Umgebung traten Abhängigkeitsfehler auf, die den Teardown-Prozess blockierten.

### Ursachenanalyse
Einzelne Cloud-Ressourcen lassen sich nur löschen, wenn abhängige Objekte bereits entfernt wurden (z. B. zwischen NCC, VPN, Routern und Netzwerken).

### Lösungsansatz
Der Rückbau erfolgte in einer festen, strikten Reihenfolge: zuerst Spoke und Hub, danach Tunnel, Gateways, Router und zuletzt Firewalls, Subnetze und VPCs.

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
