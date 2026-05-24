# Betriebshandbuch Teil 1: Standardbetrieb

Dieses Dokument beschreibt die reguläre Bereitstellung und den Betrieb einer hybriden Netzwerkarchitektur mit Google Cloud Network Connectivity Center (NCC), HA VPN und Cloud Router. Es verwendet den aktuellen `gcloud`-Syntax für VPN-Spokes und setzt einen sauberen Zugriff auf Test-VMs über Identity-Aware Proxy (IAP) voraus, sodass keine öffentlichen IP-Adressen für Administrationszugriffe benötigt werden.

## Zweck und Geltungsbereich

Das Handbuch dient als Arbeitsgrundlage für die Bereitstellung, Erweiterung und Kontrolle der Netzwerkarchitektur im Regelbetrieb. Störungen, Syntaxabweichungen und projektspezifische Workarounds sind nicht Bestandteil dieses Dokuments und werden getrennt im Troubleshooting-Handbuch behandelt. 
https://github.com/Naekun/IHK-Projektarbeit-NCC/blob/main/troubleshooting.md

## Voraussetzungen

Vor der Bereitstellung müssen die erforderlichen APIs aktiviert sein und die ausführende Person benötigt passende IAM-Berechtigungen für Compute Engine, Cloud Router, VPN und Network Connectivity Center.

Für den administrativen Zugriff ohne öffentliche IP-Adressen wird IAP TCP Forwarding verwendet. Dafür müssen IAP vorbereitet, die notwendigen IAM-Rollen vergeben und Firewall-Regeln für die IAP-Quellnetze gesetzt werden.

## Architekturübersicht

Die Architektur besteht aus einer Cloud-VPC, einer angebundenen Standort-VPC, zwei HA-VPN-Tunneln, zwei Cloud Routern und einem NCC-Hub mit einem VPN-Spoke. Der dynamische Routenaustausch erfolgt über BGP auf den Cloud Routern, nicht auf dem NCC-Hub.

## 1. Umgebungsvariablen vorbereiten

Zuerst werden Projekt, Region und alle Ressourcennamen definiert. Dadurch bleibt die Bereitstellung reproduzierbar und Eingabefehler werden reduziert.

```bash
export PROJECT_ID="projekt-id"
export REGION="europe-west3"
export ZONE="europe-west3-a"

gcloud config set project "${PROJECT_ID}"
gcloud services enable compute.googleapis.com networkconnectivity.googleapis.com iap.googleapis.com

export CLOUD_VPC="vpc-europe"
export CLOUD_SUBNET="subnet-europe"
export CLOUD_CIDR="10.10.0.0/24"
export CLOUD_ROUTER="cr-europe"
export CLOUD_ASN="65001"
export CLOUD_GW="vpngw-europe"

export BRANCH_VPC="vpc-branch-a"
export BRANCH_SUBNET="subnet-branch-a"
export BRANCH_CIDR="192.168.1.0/24"
export BRANCH_ROUTER="cr-branch-a"
export BRANCH_ASN="65002"
export BRANCH_GW="vpngw-branch-a"

export HUB_NAME="ncc-hub-global"
export SPOKE_NAME="spoke-branch-a"
export SHARED_SECRET="[SICHERER_IKEV2_KEY]"

export TUN_EU_TO_BR_0="tunnel-eu-to-br-0"
export TUN_BR_TO_EU_0="tunnel-br-to-eu-0"
export EU_BGP_0_IP="169.254.1.1"
export BR_BGP_0_IP="169.254.1.2"

export TUN_EU_TO_BR_1="tunnel-eu-to-br-1"
export TUN_BR_TO_EU_1="tunnel-br-to-eu-1"
export EU_BGP_1_IP="169.254.2.1"
export BR_BGP_1_IP="169.254.2.2"
```

## 2. Basis-Netzwerk bereitstellen

Die VPCs werden im Custom-Subnet-Modus erstellt. So entstehen nur die ausdrücklich definierten Subnetze.

```bash
gcloud compute networks create "${CLOUD_VPC}" --subnet-mode=custom
gcloud compute networks create "${BRANCH_VPC}" --subnet-mode=custom

gcloud compute networks subnets create "${CLOUD_SUBNET}" \
  --network="${CLOUD_VPC}" --region="${REGION}" --range="${CLOUD_CIDR}"

gcloud compute networks subnets create "${BRANCH_SUBNET}" \
  --network="${BRANCH_VPC}" --region="${REGION}" --range="${BRANCH_CIDR}"
```

## 3. Firewall-Regeln definieren

Die internen Firewall-Regeln erlauben nur den erforderlichen Verkehr zwischen den Netzen. Für IAP-Zugriffe auf VM-Instanzen wird zusätzlich TCP Port 22 aus dem IAP-Bereich `35.235.240.0/20` freigegeben.

```bash
gcloud compute firewall-rules create "fw-allow-internal-eu" \
  --network="${CLOUD_VPC}" \
  --direction=INGRESS \
  --allow=tcp:22,icmp \
  --source-ranges="${BRANCH_CIDR},35.235.240.0/20"

gcloud compute firewall-rules create "fw-allow-internal-branch" \
  --network="${BRANCH_VPC}" \
  --direction=INGRESS \
  --allow=tcp:22,icmp \
  --source-ranges="${CLOUD_CIDR},35.235.240.0/20"
```

## 4. Cloud Router und HA-VPN-Gateways erstellen

Cloud Router übernimmt den BGP-basierten Routenaustausch. HA VPN benötigt zwei Tunnel pro Verbindung, um Hochverfügbarkeit zu ermöglichen.

```bash
gcloud compute routers create "${CLOUD_ROUTER}" \
  --region="${REGION}" --network="${CLOUD_VPC}" --asn="${CLOUD_ASN}"

gcloud compute routers create "${BRANCH_ROUTER}" \
  --region="${REGION}" --network="${BRANCH_VPC}" --asn="${BRANCH_ASN}"

gcloud compute vpn-gateways create "${CLOUD_GW}" \
  --network="${CLOUD_VPC}" --region="${REGION}"

gcloud compute vpn-gateways create "${BRANCH_GW}" \
  --network="${BRANCH_VPC}" --region="${REGION}"
```

## 5. Tunnel und BGP konfigurieren

Zuerst wird Tunnel 0 erstellt und mit einem BGP-Peer versehen. Danach erfolgt die gleiche Konfiguration für Tunnel 1.

```bash
# Tunnel 0: Cloud -> Branch
gcloud compute vpn-tunnels create "${TUN_EU_TO_BR_0}" \
  --peer-gcp-gateway="${BRANCH_GW}" \
  --region="${REGION}" \
  --ike-version=2 \
  --shared-secret="${SHARED_SECRET}" \
  --router="${CLOUD_ROUTER}" \
  --vpn-gateway="${CLOUD_GW}" \
  --interface=0

gcloud compute routers add-interface "${CLOUD_ROUTER}" \
  --interface-name="if-eu-to-br-0" \
  --ip-address="${EU_BGP_0_IP}" \
  --mask-length=30 \
  --vpn-tunnel="${TUN_EU_TO_BR_0}" \
  --region="${REGION}"

gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-0" \
  --interface="if-eu-to-br-0" \
  --peer-ip-address="${BR_BGP_0_IP}" \
  --peer-asn="${BRANCH_ASN}" \
  --region="${REGION}"

# Tunnel 0: Branch -> Cloud
gcloud compute vpn-tunnels create "${TUN_BR_TO_EU_0}" \
  --peer-gcp-gateway="${CLOUD_GW}" \
  --region="${REGION}" \
  --ike-version=2 \
  --shared-secret="${SHARED_SECRET}" \
  --router="${BRANCH_ROUTER}" \
  --vpn-gateway="${BRANCH_GW}" \
  --interface=0

gcloud compute routers add-interface "${BRANCH_ROUTER}" \
  --interface-name="if-br-to-eu-0" \
  --ip-address="${BR_BGP_0_IP}" \
  --mask-length=30 \
  --vpn-tunnel="${TUN_BR_TO_EU_0}" \
  --region="${REGION}"

gcloud compute routers add-bgp-peer "${BRANCH_ROUTER}" \
  --peer-name="bgp-eu-0" \
  --interface="if-br-to-eu-0" \
  --peer-ip-address="${EU_BGP_0_IP}" \
  --peer-asn="${CLOUD_ASN}" \
  --region="${REGION}"

# Tunnel 1: Cloud -> Branch
gcloud compute vpn-tunnels create "${TUN_EU_TO_BR_1}" \
  --peer-gcp-gateway="${BRANCH_GW}" \
  --region="${REGION}" \
  --ike-version=2 \
  --shared-secret="${SHARED_SECRET}" \
  --router="${CLOUD_ROUTER}" \
  --vpn-gateway="${CLOUD_GW}" \
  --interface=1

gcloud compute routers add-interface "${CLOUD_ROUTER}" \
  --interface-name="if-eu-to-br-1" \
  --ip-address="${EU_BGP_1_IP}" \
  --mask-length=30 \
  --vpn-tunnel="${TUN_EU_TO_BR_1}" \
  --region="${REGION}"

gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-1" \
  --interface="if-eu-to-br-1" \
  --peer-ip-address="${BR_BGP_1_IP}" \
  --peer-asn="${BRANCH_ASN}" \
  --region="${REGION}"

# Tunnel 1: Branch -> Cloud
gcloud compute vpn-tunnels create "${TUN_BR_TO_EU_1}" \
  --peer-gcp-gateway="${CLOUD_GW}" \
  --region="${REGION}" \
  --ike-version=2 \
  --shared-secret="${SHARED_SECRET}" \
  --router="${BRANCH_ROUTER}" \
  --vpn-gateway="${BRANCH_GW}" \
  --interface=1

gcloud compute routers add-interface "${BRANCH_ROUTER}" \
  --interface-name="if-br-to-eu-1" \
  --ip-address="${BR_BGP_1_IP}" \
  --mask-length=30 \
  --vpn-tunnel="${TUN_BR_TO_EU_1}" \
  --region="${REGION}"

gcloud compute routers add-bgp-peer "${BRANCH_ROUTER}" \
  --peer-name="bgp-eu-1" \
  --interface="if-br-to-eu-1" \
  --peer-ip-address="${EU_BGP_1_IP}" \
  --peer-asn="${CLOUD_ASN}" \
  --region="${REGION}"
```

## 6. NCC-Hub und VPN-Spoke anlegen

Für den aktuellen `gcloud`-Syntax wird der Spoke mit `linked-vpn-tunnels create` erstellt. Die Tunnel werden als vollständige Compute-API-URIs übergeben.
```bash
gcloud network-connectivity hubs create "${HUB_NAME}" \
  --description="Zentraler Hybrid Cloud Hub"

export HUB_URI="https://networkconnectivity.googleapis.com/v1/projects/${PROJECT_ID}/locations/global/hubs/${HUB_NAME}"
export TUNNEL0_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_0}"
export TUNNEL1_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_1}"

gcloud network-connectivity spokes linked-vpn-tunnels create "${SPOKE_NAME}" \
  --region="${REGION}" \
  --hub="${HUB_URI}" \
  --vpn-tunnels="${TUNNEL0_URI},${TUNNEL1_URI}" \
  --site-to-site-data-transfer
```

## 7. Test-VMs ohne öffentliche IP anlegen

Für den Regelbetrieb werden Test-VMs ohne externe Adresse bereitgestellt. Der Zugriff erfolgt ausschließlich über IAP TCP Forwarding.

```bash
gcloud compute instances create "vm-cloud" \
  --zone="${ZONE}" \
  --machine-type=e2-micro \
  --network-interface=subnet="${CLOUD_SUBNET}",no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud

gcloud compute instances create "vm-branch" \
  --zone="${ZONE}" \
  --machine-type=e2-micro \
  --network-interface=subnet="${BRANCH_SUBNET}",no-address \
  --image-family=debian-12 \
  --image-project=debian-cloud
```

## 8. Zugriff über IAP und Betriebsprüfung

SSH-Zugriffe erfolgen mit `--tunnel-through-iap`. Damit bleibt die Administrationsoberfläche ohne öffentliche Adresse erreichbar.

```bash
export CLOUD_VM_IP=$(gcloud compute instances describe "vm-cloud" --zone="${ZONE}" --format='get(networkInterfaces[0].networkIP)')

# SSH über IAP
gcloud compute ssh "vm-branch" --zone="${ZONE}" --tunnel-through-iap

# Konnektivität über internes Netz prüfen
gcloud compute ssh "vm-branch" \
  --zone="${ZONE}" \
  --tunnel-through-iap \
  --command="ping -c 4 ${CLOUD_VM_IP}"
```

## 9. Status und Dokumentation prüfen

Beim Cloud Router wird im Statusfeld `UP` angezeigt, wenn die BGP-Sitzung erfolgreich aufgebaut wurde. Das entspricht dem BGP-Zustand `Established`.

```bash
# BGP Peer Status
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bgpPeerStatus[].name:label=PEER_NAME, result.bgpPeerStatus[].status:label=STATUS)"

# Gelernte Routen
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bestRoutes[].destRange:label=DESTINATION_RANGE, result.bestRoutes[].nextHopIp:label=NEXT_HOP, result.bestRoutes[].priority:label=PRIORITY)"

# Firewall-Regeln
gcloud compute firewall-rules list \
  --filter="network:(${CLOUD_VPC} OR ${BRANCH_VPC})" \
  --format="table(name:label=NAME, network:label=NETWORK, direction:label=DIRECTION, priority:label=PRIORITY, sourceRanges.list():label=SRC_RANGES, allowed[].map().firewall_rule().list():label=ALLOWED_PROTOCOLS)"
```
