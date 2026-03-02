# Configuration Reference

This document provides detailed configuration options for Aether OnRamp components.

## Table of Contents

1. [Network Configuration](#network-configuration)
2. [SD-Core Configuration](#sd-core-configuration)
3. [RAN Configuration](#ran-configuration)
4. [Security Configuration](#security-configuration)
5. [Performance Tuning](#performance-tuning)
6. [Advanced Features](#advanced-features)

## Network Configuration

### PLMN Configuration

Public Land Mobile Network (PLMN) identification:

```yaml
# Format: MCC-MNC
# MCC: Mobile Country Code (3 digits)
# MNC: Mobile Network Code (2-3 digits)

# Example: 208-93 (France - Test Network)
mcc: '208'
mnc: '93'

# Common PLMN IDs:
# USA: 310-260 (T-Mobile), 310-410 (AT&T), 311-480 (Verizon)
# UK: 234-15 (Vodafone), 234-10 (O2)
# Test: 001-01 (Test PLMN)
```

### Network Slicing

Configure network slices for different service types:

```yaml
# S-NSSAI: Single Network Slice Selection Assistance Information
slices:
  # Enhanced Mobile Broadband (eMBB)
  - name: embb
    sst: 1              # Slice/Service Type
    sd: 0x010203        # Slice Differentiator (optional)
    dnn: internet       # Data Network Name
    priority: 1
    
  # Ultra-Reliable Low Latency (URLLC)
  - name: urllc
    sst: 2
    sd: 0x112233
    dnn: iot
    priority: 10
    
  # Massive IoT (mMTC)
  - name: mmtc
    sst: 3
    sd: 0x223344
    dnn: iot-massive
    priority: 5
```

### IP Address Pools

Configure IP address allocation for UEs:

```yaml
# Data Network Names (DNN) and IP pools
dnn_list:
  - dnn: internet
    cidr: 172.250.0.0/16
    dns:
      primary: 8.8.8.8
      secondary: 8.8.4.4
    mtu: 1400
    
  - dnn: iot
    cidr: 172.251.0.0/16
    dns:
      primary: 1.1.1.1
      secondary: 1.0.0.1
    mtu: 1400
```

### Interface Configuration

#### N2 Interface (AMF - gNB)

```yaml
n2:
  interface: eth0
  ip: 0.0.0.0
  port: 38412
  protocol: SCTP
```

#### N3 Interface (UPF - gNB)

```yaml
n3:
  interface: eth0
  ip: 0.0.0.0
  port: 2152
  protocol: UDP
```

#### N4 Interface (SMF - UPF)

```yaml
n4:
  interface: eth0
  pfcp:
    ip: 0.0.0.0
    port: 8805
```

## SD-Core Configuration

### AMF Configuration

Complete AMF configuration:

```yaml
amf:
  cfgFiles:
    amfcfg.conf:
      configuration:
        amfName: AMF
        
        # NGAP Interface
        ngapIpList:
          - 0.0.0.0
        ngapPort: 38412
        
        # SBI Interface
        sbi:
          scheme: http
          registerIPv4: amf
          bindingIPv4: 0.0.0.0
          port: 29518
          
        # Service Names
        serviceNameList:
          - namf-comm
          - namf-evts
          - namf-mt
          - namf-loc
          - namf-oam
        
        # GUAMI Configuration
        servedGuamiList:
          - plmnId:
              mcc: 208
              mnc: 93
            amfId: cafe00
        
        # Tracking Area Configuration
        supportTaiList:
          - plmnId:
              mcc: 208
              mnc: 93
            tac: 1
        
        # PLMN Support
        plmnSupportList:
          - plmnId:
              mcc: 208
              mnc: 93
            snssaiList:
              - sst: 1
                sd: "010203"
              - sst: 2
                sd: "112233"
        
        # Security
        security:
          integrityOrder:
            - NIA2
            - NIA1
            - NIA0
          cipheringOrder:
            - NEA2
            - NEA1
            - NEA0
        
        # Network Feature Support
        networkFeatureSupport5GS:
          enable: true
          length: 1
          imsVoPS: 0
          emc: 0
          emf: 0
          iwkN26: 0
          mpsi: 0
          emcN3: 0
          mcsi: 0
        
        # Timers (in seconds)
        t3502: 720
        t3512: 3600
        non3gppDeregistrationTimer: 3240
```

### SMF Configuration

```yaml
smf:
  cfgFiles:
    smfcfg.conf:
      configuration:
        smfName: SMF
        
        # SBI Interface
        sbi:
          scheme: http
          registerIPv4: smf
          bindingIPv4: 0.0.0.0
          port: 29502
        
        # Service Names
        serviceNameList:
          - nsmf-pdusession
          - nsmf-event-exposure
          - nsmf-oam
        
        # PFCP Interface
        pfcp:
          addr: 0.0.0.0
          port: 8805
        
        # User Plane Information
        userplane_information:
          up_nodes:
            UPF:
              type: UPF
              node_id: upf
              sNssaiUpfInfos:
                - sNssai:
                    sst: 1
                    sd: "010203"
                  dnnUpfInfoList:
                    - dnn: internet
                - sNssai:
                    sst: 2
                    sd: "112233"
                  dnnUpfInfoList:
                    - dnn: iot
          links:
            - A: gNB
              B: UPF
        
        # DNN Configuration
        dnn_list:
          - dnn: internet
            cidr: 172.250.0.0/16
            natifname: eth0
          - dnn: iot
            cidr: 172.251.0.0/16
            natifname: eth0
        
        # UE Routing
        ue_subnet: 172.250.0.0/16
```

### UPF Configuration

```yaml
upf:
  cfgFiles:
    upf.json:
      # Mode: af_packet or dpdk
      mode: af_packet
      
      # Hardware checksum offload
      hwcksum: true
      
      # GTP-U extension header support
      gtppsc: true
      
      # CP Interface
      cpiface:
        dnn: internet
        hostname: upf
        http_port: "8080"
        enable_ue_ip_alloc: false
      
      # Access Network Interface (N3)
      access:
        ifname: eth0
        ip: 0.0.0.0
      
      # Core Network Interface (N6)
      core:
        ifname: eth0
        ip: 0.0.0.0
      
      # Slice Configuration
      slice_rate_limit_config:
        n6_bps: 1000000000  # 1 Gbps
        n6_burst_bytes: 12500000  # 12.5 MB
        n3_bps: 1000000000
        n3_burst_bytes: 12500000
  
  # Resource Configuration
  resources:
    enabled: true
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "2000m"
      memory: "4Gi"
```

### UDM/UDR Configuration

```yaml
udm:
  cfgFiles:
    udmcfg.conf:
      configuration:
        sbi:
          scheme: http
          registerIPv4: udm
          bindingIPv4: 0.0.0.0
          port: 29503
        serviceNameList:
          - nudm-sdm
          - nudm-uecm
          - nudm-ueau
          - nudm-ee
          - nudm-pp

udr:
  cfgFiles:
    udrcfg.conf:
      configuration:
        sbi:
          scheme: http
          registerIPv4: udr
          bindingIPv4: 0.0.0.0
          port: 29504
        
        # MongoDB Configuration
        mongodb:
          name: free5gc
          url: mongodb://mongodb:27017
```

## RAN Configuration

### gNB Configuration

```yaml
gnb:
  # Identity
  mcc: '208'
  mnc: '93'
  nci: '0x000000010'  # NR Cell Identity
  idLength: 32
  tac: 1              # Tracking Area Code
  
  # Network Interfaces
  linkIp: 0.0.0.0     # Local IP
  ngapIp: 0.0.0.0     # N2 Interface IP
  gtpIp: 0.0.0.0      # N3 Interface IP
  
  # AMF Configuration
  amfConfigs:
    - address: 192.168.1.100
      port: 38412
  
  # Supported Slices
  slices:
    - sst: 1
      sd: 0x010203
    - sst: 2
      sd: 0x112233
  
  # Ignore Stream Ids (for UERANSIM compatibility)
  ignoreStreamIds: true
  
  # Radio Configuration
  cell:
    # Physical Cell Identity
    pci: 1
    
    # DL/UL ARFCN
    dlArfcn: 632628
    ulArfcn: 632628
    
    # Bandwidth
    dlBandwidth: 100  # MHz
    ulBandwidth: 100  # MHz
    
    # Transmission Power
    txPower: 23       # dBm
```

### UE Configuration

```yaml
ue:
  # SUPI (IMSI)
  supi: 'imsi-208930000000001'
  mcc: '208'
  mnc: '93'
  
  # Security Keys
  key: '000102030405060708090A0B0C0D0E0F'
  op: '63bfa50ee6523365ff14c1f45f88737d'
  opType: 'OPC'  # or 'OP'
  
  # AMF Value
  amf: '8000'
  
  # Identity
  imei: '356938035643803'
  imeiSv: '4370816125816151'
  
  # gNB Search List
  gnbSearchList:
    - 192.168.1.10
  
  # Security Algorithms
  ciphering: '0'    # NEA0
  integrity: '2'    # NIA2
  
  # PDU Sessions
  sessions:
    - type: 'IPv4'
      apn: 'internet'
      slice:
        sst: 1
        sd: 0x010203
    - type: 'IPv4'
      apn: 'iot'
      slice:
        sst: 2
        sd: 0x112233
  
  # NSSAI Configuration
  configured-nssai:
    - sst: 1
      sd: 0x010203
    - sst: 2
      sd: 0x112233
  
  default-nssai:
    - sst: 1
      sd: 0x010203
```

## Security Configuration

### Authentication Credentials

```yaml
subscribers:
  - imsi: '208930000000001'
    # K: Permanent Key (128-bit)
    key: '000102030405060708090A0B0C0D0E0F'
    
    # OPc: Operator Code (derived from K and OP)
    opc: '69d5c2eb2e2e624750541d3bbc692ba5'
    
    # Or use OP (Operator Variant Algorithm Configuration Field)
    # op: '63bfa50ee6523365ff14c1f45f88737d'
    
    # AMF: Authentication Management Field
    amf: '8000'
    
    # SQN: Sequence Number (for replay protection)
    sqn: '000000000000'
```

### Security Algorithms

```yaml
security:
  # Integrity Protection Algorithms (order of preference)
  integrityOrder:
    - NIA2  # 128-bit SNOW 3G
    - NIA1  # 128-bit KASUMI
    - NIA0  # Null integrity
  
  # Ciphering Algorithms (order of preference)
  cipheringOrder:
    - NEA2  # 128-bit SNOW 3G
    - NEA1  # 128-bit KASUMI
    - NEA0  # Null ciphering
```

### TLS Configuration

```yaml
tls:
  enabled: true
  
  # Certificate paths
  certPath: /etc/certs/server.crt
  keyPath: /etc/certs/server.key
  caPath: /etc/certs/ca.crt
  
  # TLS version
  minVersion: "1.2"
  maxVersion: "1.3"
  
  # Cipher suites
  cipherSuites:
    - TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
    - TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
```

## Performance Tuning

### Resource Limits

```yaml
resources:
  amf:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  
  smf:
    requests:
      cpu: "500m"
      memory: "512Mi"
    limits:
      cpu: "1000m"
      memory: "1Gi"
  
  upf:
    requests:
      cpu: "1000m"
      memory: "2Gi"
    limits:
      cpu: "4000m"
      memory: "8Gi"
  
  # Database
  mongodb:
    requests:
      cpu: "500m"
      memory: "1Gi"
    limits:
      cpu: "1000m"
      memory: "2Gi"
```

### UPF Performance

```yaml
upf:
  # DPDK Configuration
  dpdk:
    enabled: true
    cores: "0,1,2,3"
    memory: 4096  # MB
    
    # Hugepages
    hugepages:
      enabled: true
      size: 1Gi
  
  # Worker threads
  workers: 4
  
  # Buffer sizes
  bufferSize:
    rxBurst: 32
    txBurst: 32
    rxRing: 1024
    txRing: 1024
```

### Kernel Tuning

```bash
# Add to /etc/sysctl.conf

# Increase buffer sizes
net.core.rmem_max=268435456
net.core.wmem_max=268435456
net.core.rmem_default=268435456
net.core.wmem_default=268435456

# Increase connection tracking table
net.netfilter.nf_conntrack_max=1048576

# Optimize TCP
net.ipv4.tcp_congestion_control=bbr
net.ipv4.tcp_rmem=4096 87380 134217728
net.ipv4.tcp_wmem=4096 65536 134217728

# Enable IP forwarding
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

# Optimize routing cache
net.ipv4.route.gc_timeout=100
```

## Advanced Features

### QoS Configuration

```yaml
qos:
  profiles:
    # Profile 1: Conversational Voice
    - 5qi: 1
      priority: 20
      packetDelayBudget: 100  # ms
      packetErrorRate: 0.01
      bitrate:
        uplink: 150000      # 150 Kbps
        downlink: 150000
    
    # Profile 5: IMS Signaling
    - 5qi: 5
      priority: 10
      packetDelayBudget: 100
      packetErrorRate: 0.001
      bitrate:
        uplink: 1000000     # 1 Mbps
        downlink: 1000000
    
    # Profile 9: Internet (default)
    - 5qi: 9
      priority: 80
      packetDelayBudget: 300
      packetErrorRate: 0.001
      bitrate:
        uplink: 100000000   # 100 Mbps
        downlink: 200000000 # 200 Mbps
```

### Policy Control

```yaml
pcf:
  policies:
    - name: default-policy
      pccRules:
        - ruleId: 1
          flowDescriptions:
            - permit out ip from any to assigned
            - permit in ip from any to assigned
          precedence: 255
          qosRef: 9
    
    - name: video-streaming
      pccRules:
        - ruleId: 2
          flowDescriptions:
            - permit out tcp from any to any 80
            - permit out tcp from any to any 443
          precedence: 100
          qosRef: 7
```

### Monitoring Configuration

```yaml
monitoring:
  prometheus:
    enabled: true
    port: 9090
    retention: 15d
    scrapeInterval: 15s
    
  grafana:
    enabled: true
    port: 3000
    adminPassword: admin
    
  logging:
    level: info  # debug, info, warn, error
    output: stdout
    format: json
```

### Multi-Cluster Federation

```yaml
federation:
  enabled: true
  
  # Local cluster
  local:
    clusterId: cluster1
    plmn:
      mcc: 208
      mnc: 93
  
  # Remote clusters
  remote:
    - clusterId: cluster2
      endpoint: https://cluster2.example.com
      plmn:
        mcc: 208
        mnc: 94
```

## Environment-Specific Configurations

### Development

```yaml
environment: development

replicas:
  amf: 1
  smf: 1
  upf: 1

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

logging:
  level: debug

monitoring:
  enabled: false
```

### Production

```yaml
environment: production

replicas:
  amf: 3
  smf: 3
  upf: 2

resources:
  requests:
    cpu: 1000m
    memory: 2Gi
  limits:
    cpu: 4000m
    memory: 8Gi

logging:
  level: info

monitoring:
  enabled: true
  retention: 30d

backup:
  enabled: true
  schedule: "0 2 * * *"
  retention: 7

highAvailability:
  enabled: true
  antiAffinity: required
```

## Configuration Validation

```bash
# Validate YAML syntax
yamllint values.yaml

# Validate against schema
helm lint -f values.yaml

# Dry-run deployment
helm install --dry-run --debug sd-core aether/sd-core -f values.yaml

# Test configuration
helm template sd-core aether/sd-core -f values.yaml | kubectl apply --dry-run=client -f -
```

## Configuration Examples

See the `examples/` directory for complete configuration example:
- `examples/basic-5g.yaml` - Basic 5G configuration
