# GCP Network Connectivity Center (NCC) & HA VPN Deployment Guide

Dieses Repository enthält die Skripte zur automatisierten Bereitstellung einer hochverfügbaren Hub-and-Spoke-Netzwerkarchitektur in der Google Cloud Platform (GCP) mittels Network Connectivity Center (NCC) und HA VPN. 

Das Setup wurde im Rahmen eines IHK-Abschlussprojekts (Fachinformatikerin für Systemintegration) entwickelt und auf Fehlerfreiheit, Ausfallsicherheit (99,99% SLA) und Cloud FinOps (Kostenoptimierung) getestet.

## Inhaltsverzeichnis
1. [Vorbereitung und Variablen](#1-vorbereitung-und-variablen)
2. [Basis-Netzwerke & Firewall](#2-basis-netzwerke--firewall)
3. [Cloud Routers & HA VPN Gateways](#3-cloud-routers--ha-vpn-gateways)
4. [IPsec-Tunnel & BGP-Routing](#4-ipsec-tunnel--bgp-routing)
5. [Network Connectivity Center (Hub & Spoke)](#5-network-connectivity-center-hub--spoke)
6. [Testing (Konnektivität, Traceroute, Failover)](#6-testing-konnektivität-traceroute-failover)
7. [Dokumentations-Export (BGP & Firewall)](#7-dokumentations-export-bgp--firewall)
8. [Teardown / Cleanup](#8-teardown--cleanup)

---

## 1. Vorbereitung von Variablen
```bash
# 1. Projekt-ID setzen 
export PROJECT_ID="projekt-id"
gcloud config set project "${PROJECT_ID}"

# 2. Benötigte APIs aktivieren (dauert ca. 1-2 Minuten)
gcloud services enable compute.googleapis.com networkconnectivity.googleapis.com

# Verifizieren, dass die APIs aktiv sind:
gcloud services list --enabled | grep -E "compute.googleapis.com|networkconnectivity.googleapis.com"

# 3. Variablen: Cloud-Umgebung (Hub)
export REGION="europe-west3"
export CLOUD_VPC="vpc-europe"
export CLOUD_SUBNET="subnet-europe"
export CLOUD_CIDR="10.10.0.0/24"
export CLOUD_ROUTER="cr-europe"
export CLOUD_ASN="65001"
export CLOUD_GW="vpngw-europe"

# 4. Variablen: Zweigstellen-Umgebung (On-Premises / Spoke)
export BRANCH_VPC="vpc-branch-a"
export BRANCH_SUBNET="subnet-branch-a"
export BRANCH_CIDR="192.168.1.0/24"
export BRANCH_ROUTER="cr-branch-a"
export BRANCH_ASN="65002"
export BRANCH_GW="vpngw-branch-a"

# 5. Variablen: NCC Hub & HA-VPN Shared Secret
export HUB_NAME="ncc-hub-global"
export SHARED_SECRET="secure-ikev2-key-2026"

# 6. Variablen: Tunnel & BGP (Tunnel 0)
export TUN_EU_TO_BR_0="tunnel-eu-to-br-0"
export TUN_BR_TO_EU_0="tunnel-br-to-eu-0"
export EU_BGP_0_IP="169.254.1.1"
export BR_BGP_0_IP="169.254.1.2"

# 7. Variablen: Tunnel & BGP (Tunnel 1 - HA)
export TUN_EU_TO_BR_1="tunnel-eu-to-br-1"
export TUN_BR_TO_EU_1="tunnel-br-to-eu-1"
export EU_BGP_1_IP="169.254.2.1"
export BR_BGP_1_IP="169.254.2.2"
```

---

## 2. Basis-Netzwerke & Firewall

Erstellung der isolierten VPCs im Custom-Mode sowie der sicheren internen Firewall-Regeln.

```bash
# VPCs anlegen
gcloud compute networks create "${CLOUD_VPC}" --subnet-mode=custom
gcloud compute networks create "${BRANCH_VPC}" --subnet-mode=custom

# Subnetze anlegen
gcloud compute networks subnets create "${CLOUD_SUBNET}" \
  --network="${CLOUD_VPC}" --region="${REGION}" --range="${CLOUD_CIDR}"

gcloud compute networks subnets create "${BRANCH_SUBNET}" \
  --network="${BRANCH_VPC}" --region="${REGION}" --range="${BRANCH_CIDR}"

# Interne Firewall-Regeln (Erlauben Ping & IAP-SSH)
gcloud compute firewall-rules create "fw-allow-internal-eu" \
  --network="${CLOUD_VPC}" --allow tcp:22,icmp \
  --source-ranges="${BRANCH_CIDR},35.235.240.0/20"

gcloud compute firewall-rules create "fw-allow-internal-branch" \
  --network="${BRANCH_VPC}" --allow tcp:22,icmp \
  --source-ranges="${CLOUD_CIDR},35.235.240.0/20"
```

---

## 3. Cloud Routers & HA VPN Gateways

Bereitstellung der Control Plane (Cloud Router) und Data Plane (VPN Gateways).

```bash
# Cloud Routers erstellen
gcloud compute routers create "${CLOUD_ROUTER}" \
  --region="${REGION}" --network="${CLOUD_VPC}" --asn="${CLOUD_ASN}"

gcloud compute routers create "${BRANCH_ROUTER}" \
  --region="${REGION}" --network="${BRANCH_VPC}" --asn="${BRANCH_ASN}"

# HA VPN Gateways erstellen
gcloud compute vpn-gateways create "${CLOUD_GW}" \
  --network="${CLOUD_VPC}" --region="${REGION}"

gcloud compute vpn-gateways create "${BRANCH_GW}" \
  --network="${BRANCH_VPC}" --region="${REGION}"
```

---

## 4. IPsec-Tunnel & BGP-Routing

Aufbau der verschlüsselten Verbindungen und Konfiguration des dynamischen Routings (BGP). Für ein SLA von 99,99% werden zwei Tunnel (Interface 0 und 1) benötigt.

```bash
# =================== TUNNEL 0 ===================
# Cloud -> Branch (Tunnel 0)
gcloud compute vpn-tunnels create "${TUN_EU_TO_BR_0}" \
  --peer-gcp-gateway="${BRANCH_GW}" --region="${REGION}" \
  --ike-version=2 --shared-secret="${SHARED_SECRET}" \
  --router="${CLOUD_ROUTER}" --vpn-gateway="${CLOUD_GW}" --interface=0

gcloud compute routers add-interface "${CLOUD_ROUTER}" \
  --interface-name="if-eu-to-br-0" --ip-address="${EU_BGP_0_IP}" --mask-length=30 --vpn-tunnel="${TUN_EU_TO_BR_0}" --region="${REGION}"

gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-0" --interface="if-eu-to-br-0" --peer-ip-address="${BR_BGP_0_IP}" --peer-asn="${BRANCH_ASN}" --region="${REGION}"

# Branch -> Cloud (Tunnel 0)
gcloud compute vpn-tunnels create "${TUN_BR_TO_EU_0}" \
  --peer-gcp-gateway="${CLOUD_GW}" --region="${REGION}" \
  --ike-version=2 --shared-secret="${SHARED_SECRET}" \
  --router="${BRANCH_ROUTER}" --vpn-gateway="${BRANCH_GW}" --interface=0

gcloud compute routers add-interface "${BRANCH_ROUTER}" \
  --interface-name="if-br-to-eu-0" --ip-address="${BR_BGP_0_IP}" --mask-length=30 --vpn-tunnel="${TUN_BR_TO_EU_0}" --region="${REGION}"

gcloud compute routers add-bgp-peer "${BRANCH_ROUTER}" \
  --peer-name="bgp-eu-0" --interface="if-br-to-eu-0" --peer-ip-address="${EU_BGP_0_IP}" --peer-asn="${CLOUD_ASN}" --region="${REGION}"


# =================== TUNNEL 1 (HA) ===================
# Cloud -> Branch (Tunnel 1)
gcloud compute vpn-tunnels create "${TUN_EU_TO_BR_1}" \
  --peer-gcp-gateway="${BRANCH_GW}" --region="${REGION}" \
  --ike-version=2 --shared-secret="${SHARED_SECRET}" \
  --router="${CLOUD_ROUTER}" --vpn-gateway="${CLOUD_GW}" --interface=1

gcloud compute routers add-interface "${CLOUD_ROUTER}" \
  --interface-name="if-eu-to-br-1" --ip-address="${EU_BGP_1_IP}" --mask-length=30 --vpn-tunnel="${TUN_EU_TO_BR_1}" --region="${REGION}"

gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" \
  --peer-name="bgp-branch-a-1" --interface="if-eu-to-br-1" --peer-ip-address="${BR_BGP_1_IP}" --peer-asn="${BRANCH_ASN}" --region="${REGION}"

# Branch -> Cloud (Tunnel 1)
gcloud compute vpn-tunnels create "${TUN_BR_TO_EU_1}" \
  --peer-gcp-gateway="${CLOUD_GW}" --region="${REGION}" \
  --ike-version=2 --shared-secret="${SHARED_SECRET}" \
  --router="${BRANCH_ROUTER}" --vpn-gateway="${BRANCH_GW}" --interface=1

gcloud compute routers add-interface "${BRANCH_ROUTER}" \
  --interface-name="if-br-to-eu-1" --ip-address="${BR_BGP_1_IP}" --mask-length=30 --vpn-tunnel="${TUN_BR_TO_EU_1}" --region="${REGION}"

gcloud compute routers add-bgp-peer "${BRANCH_ROUTER}" \
  --peer-name="bgp-eu-1" --interface="if-br-to-eu-1" --peer-ip-address="${EU_BGP_1_IP}" --peer-asn="${CLOUD_ASN}" --region="${REGION}"
```

---

## 5. Network Connectivity Center (Hub & Spoke)

Anbindung der beiden HA-VPN-Tunnel als eine redundante Spoke an den globalen NCC Hub.

```bash
# Zentralen Hub erstellen
gcloud network-connectivity hubs create "${HUB_NAME}" \
  --description="Zentraler Hybrid Cloud Hub"

# VPN-URIs generieren (neuer gcloud Syntax)
export TUNNEL0_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_0}"
export TUNNEL1_URI="https://www.googleapis.com/compute/v1/projects/${PROJECT_ID}/regions/${REGION}/vpnTunnels/${TUN_EU_TO_BR_1}"

# Beide VPN-Tunnel als linked-vpn-tunnels an das NCC anbinden
gcloud network-connectivity spokes linked-vpn-tunnels create "spoke-branch-a" \
  --region="${REGION}" \
  --hub="${HUB_NAME}" \
  --vpn-tunnels="${TUNNEL0_URI},${TUNNEL1_URI}" \
  --site-to-site-data-transfer
```

---

## 6. Testing (Konnektivität, Traceroute, Failover)

Um strikte IAM-Restriktionen für IAP zu umgehen, weisen wir den Test-VMs temporäre externe IPs zu, testen das Routing aber ausschließlich über die **internen IP-Adressen**, um die VPN-Funktionalität zu beweisen.

```bash
# 1. Test-VMs erstellen (ohne Public IP für Security-Default)
gcloud compute instances create "vm-cloud" --zone="${REGION}-a" --machine-type=e2-micro --network-interface=subnet="${CLOUD_SUBNET}",no-address
gcloud compute instances create "vm-branch" --zone="${REGION}-a" --machine-type=e2-micro --network-interface=subnet="${BRANCH_SUBNET}",no-address

# 2. Temporäre externe IPs und Firewall für SSH-Testzugang hinzufügen
gcloud compute instances add-access-config "vm-cloud" --zone="${REGION}-a" --network-interface="nic0"
gcloud compute instances add-access-config "vm-branch" --zone="${REGION}-a" --network-interface="nic0"

gcloud compute firewall-rules create "fw-allow-ssh-external-eu" --network="${CLOUD_VPC}" --allow tcp:22 --source-ranges="0.0.0.0/0"
gcloud compute firewall-rules create "fw-allow-ssh-external-branch" --network="${BRANCH_VPC}" --allow tcp:22 --source-ranges="0.0.0.0/0"

# 3. Interne IPs auslesen
export CLOUD_VM_IP=$(gcloud compute instances describe "vm-cloud" --zone="${REGION}-a" --format='get(networkInterfaces.networkIP)')
export BRANCH_VM_IP=$(gcloud compute instances describe "vm-branch" --zone="${REGION}-a" --format='get(networkInterfaces.networkIP)')

# 4. PING-TEST (Branch -> Cloud)
gcloud compute ssh "vm-branch" --zone="${REGION}-a" --command="ping -c 4 ${CLOUD_VM_IP}"

# 5. TRACEROUTE-TEST (Mit Flag -I für ICMP, da UDP blockiert wird)
gcloud compute ssh "vm-branch" --zone="${REGION}-a" --command="sudo apt-get update -y > /dev/null && sudo apt-get install traceroute -y > /dev/null && sudo traceroute -I -n ${CLOUD_VM_IP}"

# 6. FAILOVER-TEST (Tunnel 0 BGP-Peer löschen)
gcloud compute routers remove-bgp-peer "${CLOUD_ROUTER}" --peer-name="bgp-branch-a-0" --region="${REGION}" --quiet
sleep 20 # Warten auf BGP-Convergence

# Erneuter Ping (Traffic läuft jetzt über Tunnel 1)
gcloud compute ssh "vm-branch" --zone="${REGION}-a" --command="ping -c 4 ${CLOUD_VM_IP}"

# BGP-Session wiederherstellen
gcloud compute routers add-bgp-peer "${CLOUD_ROUTER}" --peer-name="bgp-branch-a-0" --interface="if-eu-to-br-0" --peer-ip-address="${BR_BGP_0_IP}" --peer-asn="${BRANCH_ASN}" --region="${REGION}"
```

---

## 7. Dokumentations-Export (BGP & Firewall)

# 2. Firewall-Tabelle exportieren
gcloud compute firewall-rules list \
  --format="table(name:label=NAME, network:label=NETWORK, direction:label=DIRECTION, priority:label=PRIORITY, sourceRanges.list():label=SRC_RANGES, allowed[].map().firewall_rule().list():label=ALLOWED_PROTOCOLS)"

# 3. BGP Learned Routes exportieren (Dynamic Routes)
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bestRoutes[].destRange:label=DESTINATION_RANGE, result.bestRoutes[].nextHopIp:label=NEXT_HOP, result.bestRoutes[].priority:label=PRIORITY)"

# 4. BGP Peer Status (Sollte für beide Tunnel ESTABLISHED sein)
gcloud compute routers get-status "${CLOUD_ROUTER}" \
  --region="${REGION}" \
  --format="table(result.bgpPeerStatus[].name:label=PEER_NAME, result.bgpPeerStatus[].status:label=STATUS)"
```

---

## 8. Teardown / Cleanup

Vollständiges und kosteneffizientes Löschen aller erstellten Ressourcen in korrekter Reihenfolge.

```bash
# 1. Virtuelle Maschinen
gcloud compute instances delete "vm-cloud" "vm-branch" --zone="${REGION}-a" --quiet

# 2. NCC Spokes & Hub
gcloud network-connectivity spokes delete "spoke-branch-a" --region="${REGION}" --quiet
gcloud network-connectivity hubs delete "${HUB_NAME}" --quiet

# 3. VPN Tunnel
gcloud compute vpn-tunnels delete "${TUN_EU_TO_BR_0}" "${TUN_BR_TO_EU_0}" "${TUN_EU_TO_BR_1}" "${TUN_BR_TO_EU_1}" --region="${REGION}" --quiet

# 4. Gateways & Router
gcloud compute vpn-gateways delete "${CLOUD_GW}" "${BRANCH_GW}" --region="${REGION}" --quiet
gcloud compute routers delete "${CLOUD_ROUTER}" "${BRANCH_ROUTER}" --region="${REGION}" --quiet

# 5. Firewall-Regeln
gcloud compute firewall-rules delete "fw-allow-internal-eu" "fw-allow-internal-branch" --quiet

# 6. Subnetze & VPCs
gcloud compute networks subnets delete "${CLOUD_SUBNET}" "${BRANCH_SUBNET}" --region="${REGION}" --quiet
gcloud compute networks delete "${CLOUD_VPC}" "${BRANCH_VPC}" --quiet

```