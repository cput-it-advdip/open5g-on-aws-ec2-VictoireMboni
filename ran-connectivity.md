# RAN Connectivity Guide

This guide covers connecting Radio Access Network (RAN) components to your SD-Core deployment, including both emulated and physical RANs.

## RAN Options

1. **Emulated RAN (UERANSIM)**: Software-based gNB and UE simulator
2. **Physical gNB**: Real 5G base stations
3. **Physical eNB**: 4G/LTE base stations (requires 4G core configuration)

## Option 1: Emulated RAN with UERANSIM

UERANSIM is an open-source 5G UE and RAN (gNB) simulator.

### Prerequisites

- SD-Core deployed and running
- Kubernetes cluster with network connectivity

### Step 1: Deploy UERANSIM gNB

Create gNB configuration:

```bash
cat <<EOF > /tmp/gnb-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: gnb-config
  namespace: omec
data:
  gnb.yaml: |
    mcc: '208'
    mnc: '93'
    nci: '0x000000010'
    idLength: 32
    tac: 1
    
    linkIp: 0.0.0.0
    ngapIp: 0.0.0.0
    gtpIp: 0.0.0.0
    
    # AMF Configuration
    amfConfigs:
      - address: amf.omec.svc.cluster.local
        port: 38412
    
    # Supported S-NSSAI
    slices:
      - sst: 1
        sd: 0x010203
    
    # Ignore Stream Ids
    ignoreStreamIds: true
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ueransim-gnb
  namespace: omec
  labels:
    app: ueransim-gnb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ueransim-gnb
  template:
    metadata:
      labels:
        app: ueransim-gnb
    spec:
      containers:
      - name: gnb
        image: free5gc/ueransim:latest
        command: ["/ueransim/build/nr-gnb"]
        args: ["-c", "/etc/ueransim/gnb.yaml"]
        volumeMounts:
        - name: config
          mountPath: /etc/ueransim
        securityContext:
          privileged: true
          capabilities:
            add: ["NET_ADMIN"]
      volumes:
      - name: config
        configMap:
          name: gnb-config
---
apiVersion: v1
kind: Service
metadata:
  name: ueransim-gnb
  namespace: omec
spec:
  selector:
    app: ueransim-gnb
  ports:
  - name: ngap
    protocol: TCP
    port: 38412
    targetPort: 38412
  - name: gtpu
    protocol: UDP
    port: 2152
    targetPort: 2152
EOF

kubectl apply -f /tmp/gnb-config.yaml
```

### Step 2: Verify gNB Connection

```bash
# Check gNB pod
kubectl get pods -n omec -l app=ueransim-gnb

# Check gNB logs
kubectl logs -n omec -l app=ueransim-gnb -f

# Expected output should show successful NGAP connection to AMF
```

### Step 3: Deploy UERANSIM UE

Create UE configuration:

```bash
cat <<EOF > /tmp/ue-config.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ue-config
  namespace: omec
data:
  ue.yaml: |
    # IMSI number of the UE
    supi: 'imsi-208930000000001'
    mcc: '208'
    mnc: '93'
    
    # Permanent subscription key
    key: '000102030405060708090A0B0C0D0E0F'
    # Operator code
    op: '63bfa50ee6523365ff14c1f45f88737d'
    # OP type
    opType: 'OPC'
    
    # Authentication Management Field
    amf: '8000'
    
    # IMEI number of the device
    imei: '356938035643803'
    # IMEISV number
    imeiSv: '4370816125816151'
    
    # List of gNB IP addresses for Radio Link Simulation
    gnbSearchList:
      - ueransim-gnb.omec.svc.cluster.local
    
    # Supported encryption algorithms
    ciphering: '0'
    integrity: '2'
    
    # PDU sessions to be established
    sessions:
      - type: 'IPv4'
        apn: 'internet'
        slice:
          sst: 1
          sd: 0x010203
    
    # Configured NSSAI for this UE
    configured-nssai:
      - sst: 1
        sd: 0x010203
    
    # Default Configured NSSAI for this UE
    default-nssai:
      - sst: 1
        sd: 0x010203
---
apiVersion: v1
kind: Pod
metadata:
  name: ueransim-ue
  namespace: omec
  labels:
    app: ueransim-ue
spec:
  containers:
  - name: ue
    image: free5gc/ueransim:latest
    command: ["/bin/bash", "-c"]
    args:
      - |
        /ueransim/build/nr-ue -c /etc/ueransim/ue.yaml
        tail -f /dev/null
    volumeMounts:
    - name: config
      mountPath: /etc/ueransim
    securityContext:
      privileged: true
      capabilities:
        add: ["NET_ADMIN"]
  volumes:
  - name: config
    configMap:
      name: ue-config
EOF

kubectl apply -f /tmp/ue-config.yaml
```

### Step 4: Verify UE Connection

```bash
# Check UE pod
kubectl get pods -n omec ueransim-ue

# Check UE logs
kubectl logs -n omec ueransim-ue -f

# Check interfaces in UE
kubectl exec -it -n omec ueransim-ue -- ip addr

# Look for uesimtun0 interface (PDU session)
```

### Step 5: Test Data Connectivity

```bash
# Ping from UE to internet
kubectl exec -it -n omec ueransim-ue -- ping -I uesimtun0 -c 4 8.8.8.8

# Run speed test
kubectl exec -it -n omec ueransim-ue -- curl --interface uesimtun0 -o /dev/null -w '%{speed_download}\n' http://speedtest.tele2.net/100MB.zip

# HTTP test
kubectl exec -it -n omec ueransim-ue -- curl --interface uesimtun0 http://www.google.com
```

## Option 2: Physical gNB

### Prerequisites

- Physical 5G gNB hardware
- Network connectivity between gNB and Kubernetes cluster
- gNB configured for 5G SA mode

### Network Configuration

#### Option A: Direct Connection

If gNB can directly reach the Kubernetes cluster:

```bash
# Expose AMF service as NodePort
kubectl patch svc amf -n omec -p '{"spec":{"type":"NodePort","ports":[{"port":38412,"nodePort":38412,"protocol":"TCP","name":"ngap"}]}}'

# Get NodePort IP
kubectl get nodes -o wide
```

Configure gNB to point to:
- AMF IP: `<Node-IP>`
- AMF Port: `38412`

#### Option B: LoadBalancer Service

If using AWS ELB or similar:

```bash
# Change AMF service to LoadBalancer
kubectl patch svc amf -n omec -p '{"spec":{"type":"LoadBalancer"}}'

# Get LoadBalancer IP
kubectl get svc amf -n omec
```

Configure gNB to point to the LoadBalancer IP.

#### Option C: VPN Tunnel

For secure connection:

```bash
# Install WireGuard on EC2 instance
sudo apt-get install -y wireguard

# Generate keys
wg genkey | tee privatekey | wg pubkey > publickey

# Configure WireGuard
sudo cat <<EOF > /etc/wireguard/wg0.conf
[Interface]
PrivateKey = <private-key>
Address = 10.0.0.1/24
ListenPort = 51820

[Peer]
PublicKey = <gnb-public-key>
AllowedIPs = 10.0.0.2/32
Endpoint = <gnb-public-ip>:51820
PersistentKeepalive = 25
EOF

# Start WireGuard
sudo wg-quick up wg0

# Enable at boot
sudo systemctl enable wg-quick@wg0
```

### gNB Configuration Example

Generic gNB configuration (varies by vendor):

```yaml
# Example configuration structure
gnb:
  plmn:
    mcc: 208
    mnc: 93
  
  cell:
    nci: 0x000000010
    tac: 1
    pci: 1
  
  amf:
    address: <amf-ip>
    port: 38412
  
  upf:
    address: <upf-ip>
    port: 2152
  
  slices:
    - sst: 1
      sd: 0x010203
```

### Firewall Configuration

Open required ports on EC2 security group:

```bash
# Allow NGAP (N2 interface)
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol tcp \
  --port 38412 \
  --cidr <gnb-ip>/32

# Allow GTP-U (N3 interface)
aws ec2 authorize-security-group-ingress \
  --group-id <security-group-id> \
  --protocol udp \
  --port 2152 \
  --cidr <gnb-ip>/32
```

### Verification

```bash
# Check AMF logs for gNB connection
kubectl logs -n omec -l app=amf | grep -i "gnb"

# Check NGAP associations
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  grep -i "ngap" /var/log/amf.log

# Monitor N2 interface
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  tcpdump -i any port 38412 -n
```

## Option 3: Physical eNB (4G/LTE)

### Prerequisites

- SD-Core configured for 4G mode
- Physical eNB hardware
- SIM cards with matching credentials

### Reconfigure SD-Core for 4G

```bash
# Update Helm values for 4G
cat <<EOF > /tmp/sd-core-4g-values.yaml
global:
  projectName: aether-onramp

omec-control-plane:
  enable5G: false
  enable4G: true
  
  mme:
    cfgFiles:
      config.json:
        mme:
          mcc:
            dig1: 2
            dig2: 0
            dig3: 8
          mnc:
            dig1: 9
            dig2: 3
            dig3: -1
          apnlist:
            internet: "spgwc"

omec-user-plane:
  enable: true
  spgwu:
    cfgFiles:
      upf.json:
        mode: af_packet

EOF

# Upgrade to 4G configuration
helm upgrade sd-core aether/sd-core \
  --namespace omec \
  --values /tmp/sd-core-4g-values.yaml \
  --wait
```

### eNB Configuration

Configure eNB to connect to MME:
- MME IP: `<Node-IP or LoadBalancer-IP>`
- S1AP Port: `36412`

## Multi-RAN Deployment

For connecting multiple RANs:

```bash
# Scale gNB deployment
kubectl scale deployment ueransim-gnb -n omec --replicas=3

# Or deploy multiple gNB configurations
for i in {1..3}; do
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: gnb-config-$i
  namespace: omec
data:
  gnb.yaml: |
    mcc: '208'
    mnc: '93'
    nci: '0x00000001$i'
    idLength: 32
    tac: 1
    linkIp: 0.0.0.0
    ngapIp: 0.0.0.0
    gtpIp: 0.0.0.0
    amfConfigs:
      - address: amf.omec.svc.cluster.local
        port: 38412
    slices:
      - sst: 1
        sd: 0x010203
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ueransim-gnb-$i
  namespace: omec
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ueransim-gnb-$i
  template:
    metadata:
      labels:
        app: ueransim-gnb-$i
    spec:
      containers:
      - name: gnb
        image: free5gc/ueransim:latest
        command: ["/ueransim/build/nr-gnb", "-c", "/etc/ueransim/gnb.yaml"]
        volumeMounts:
        - name: config
          mountPath: /etc/ueransim
        securityContext:
          privileged: true
      volumes:
      - name: config
        configMap:
          name: gnb-config-$i
EOF
done
```

## Monitoring and Debugging

### Monitor NGAP Messages

```bash
# Install tcpdump in AMF pod
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  apt-get update && apt-get install -y tcpdump

# Capture NGAP traffic
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  tcpdump -i any -n port 38412 -w /tmp/ngap.pcap

# Download pcap file
kubectl cp omec/$(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}'):/tmp/ngap.pcap ./ngap.pcap
```

### Monitor GTP-U Tunnels

```bash
# Check GTP-U traffic on UPF
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  tcpdump -i any -n udp port 2152

# Check tunnel interfaces
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  ip link show type gtp
```

### Check UE Registration

```bash
# View registered UEs in AMF
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=amf -o jsonpath='{.items[0].metadata.name}') -- \
  cat /var/log/amf.log | grep -i "registration"

# View PDU sessions in SMF
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=smf -o jsonpath='{.items[0].metadata.name}') -- \
  cat /var/log/smf.log | grep -i "pdu session"
```

## Troubleshooting

### gNB Cannot Connect to AMF

```bash
# Check AMF service
kubectl get svc amf -n omec

# Check AMF logs
kubectl logs -n omec -l app=amf

# Test connectivity from gNB
ping <amf-ip>
telnet <amf-ip> 38412
```

### UE Registration Failed

```bash
# Check subscriber provisioning
kubectl exec -it simapp -n omec -- /simapp/bin/simapp subscriber list

# Check AUSF logs
kubectl logs -n omec -l app=ausf

# Check UDM logs
kubectl logs -n omec -l app=udm
```

### No Data Connectivity

```bash
# Check UPF status
kubectl logs -n omec -l app=upf

# Check routing in UPF
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  ip route

# Check NAT/iptables rules
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  iptables -t nat -L
```

## Performance Optimization

### UPF Tuning

```bash
# Enable hugepages for DPDK
kubectl patch deployment upf -n omec --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/resources/limits/hugepages-2Mi", "value": "1Gi"}]'

# Enable CPU pinning
kubectl patch deployment upf -n omec --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/resources/limits/cpu", "value": "2"}]'
```

### Network Optimization

```bash
# Enable jumbo frames
sudo ip link set dev eth0 mtu 9000

# Optimize kernel parameters
sudo sysctl -w net.core.rmem_max=268435456
sudo sysctl -w net.core.wmem_max=268435456
```

## Next Steps

After connecting RAN, you can:
- [Troubleshoot issues](troubleshooting.md)
- [Configure advanced features](configuration.md)
- Monitor traffic and performance
- Scale your deployment
