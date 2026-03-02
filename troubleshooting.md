# Troubleshooting Guide

This guide helps you diagnose and resolve common issues with Aether OnRamp on AWS EC2.

## General Debugging Steps

### 1. Check System Resources

```bash
# Check EC2 instance resources
top
htop

# Check memory
free -h

# Check disk space
df -h

# Check network interfaces
ip addr
ip route
```

### 2. Check Docker

```bash
# Check Docker status
sudo systemctl status docker

# Check Docker containers
docker ps -a

# Check Docker logs
docker logs <container-id>

# Check Docker resources
docker stats
```

### 3. Check Kubernetes

```bash
# Check cluster health
kubectl get nodes
kubectl get componentstatuses

# Check all pods
kubectl get pods -A

# Check all services
kubectl get svc -A

# Check events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20
```

## Common Issues

### Issue 1: Pods in CrashLoopBackOff

**Symptoms:**
```bash
$ kubectl get pods -n omec
NAME                   READY   STATUS             RESTARTS   AGE
amf-xxx                0/1     CrashLoopBackOff   5          5m
```

**Diagnosis:**

```bash
# Check pod logs
kubectl logs -n omec amf-xxx

# Check previous logs if pod restarted
kubectl logs -n omec amf-xxx --previous

# Describe pod for events
kubectl describe pod -n omec amf-xxx
```

**Common Causes:**

1. **Configuration Error**
   ```bash
   # Check ConfigMap
   kubectl get configmap -n omec
   kubectl describe configmap <config-name> -n omec
   ```

2. **Missing Dependencies**
   ```bash
   # Check if NRF is running (required for service discovery)
   kubectl get pods -n omec -l app=nrf
   ```

3. **Resource Constraints**
   ```bash
   # Check resource limits
   kubectl describe pod -n omec amf-xxx | grep -A 5 "Limits"
   
   # Check node resources
   kubectl describe nodes | grep -A 5 "Allocated resources"
   ```

**Solutions:**

```bash
# Increase resource limits
kubectl edit deployment amf -n omec
# Update resources section

# Restart pod
kubectl delete pod -n omec amf-xxx

# Check logs again
kubectl logs -n omec amf-xxx -f
```

### Issue 2: Service Discovery Failure

**Symptoms:**
- Pods can't connect to each other
- "Failed to connect to NRF" errors

**Diagnosis:**

```bash
# Check DNS resolution
kubectl run -it --rm debug --image=busybox --restart=Never -- nslookup nrf.omec.svc.cluster.local

# Check service endpoints
kubectl get endpoints -n omec

# Check NRF service
kubectl get svc nrf -n omec
```

**Solutions:**

```bash
# Restart CoreDNS
kubectl rollout restart deployment coredns -n kube-system

# Check CoreDNS logs
kubectl logs -n kube-system -l k8s-app=kube-dns

# Verify service registration
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=nrf -o jsonpath='{.items[0].metadata.name}') -- \
  curl http://localhost:29510/nnrf-nfm/v1/nf-instances
```

### Issue 3: gNB Cannot Connect to AMF

**Symptoms:**
- gNB logs show "Connection refused" or timeout
- AMF doesn't see gNB connection

**Diagnosis:**

```bash
# Check AMF service
kubectl get svc amf -n omec

# Check AMF logs
kubectl logs -n omec -l app=amf | grep -i ngap

# Test connectivity from outside cluster
telnet <node-ip> 38412

# Check security group
aws ec2 describe-security-groups --group-ids <sg-id>
```

**Solutions:**

```bash
# Expose AMF as NodePort if not already
kubectl patch svc amf -n omec -p '{"spec":{"type":"NodePort","ports":[{"port":38412,"nodePort":38412}]}}'

# Update security group
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port 38412 \
  --cidr <gnb-ip>/32

# Check AMF configuration
kubectl get configmap -n omec amf-config -o yaml

# Restart AMF
kubectl rollout restart deployment amf -n omec
```

### Issue 4: UE Registration Failure

**Symptoms:**
- UE can't register with network
- "Authentication failed" errors

**Diagnosis:**

```bash
# Check subscriber in database
kubectl exec -it simapp -n omec -- /simapp/bin/simapp subscriber list

# Check AUSF logs
kubectl logs -n omec -l app=ausf | grep -i auth

# Check UDM logs
kubectl logs -n omec -l app=udm

# Check AMF logs for registration attempts
kubectl logs -n omec -l app=amf | grep -i registration
```

**Solutions:**

```bash
# Add subscriber if missing
kubectl exec -it simapp -n omec -- /simapp/bin/simapp \
  subscriber add \
  imsi-208930000000001 \
  key-000102030405060708090A0B0C0D0E0F \
  opc-69d5c2eb2e2e624750541d3bbc692ba5 \
  plmn-20893

# Verify credentials match between UE and subscriber database
# Check PLMN configuration matches
kubectl get configmap -n omec amf-config -o yaml | grep -A 5 plmn
```

### Issue 5: No Data Connectivity

**Symptoms:**
- UE registered but can't access internet
- PDU session established but no data flow

**Diagnosis:**

```bash
# Check UPF logs
kubectl logs -n omec -l app=upf

# Check SMF logs for PDU session
kubectl logs -n omec -l app=smf | grep -i "pdu session"

# Check UE interface
kubectl exec -it -n omec ueransim-ue -- ip addr
kubectl exec -it -n omec ueransim-ue -- ip route

# Check UPF routing
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  ip route

# Check NAT configuration
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  iptables -t nat -L -n
```

**Solutions:**

```bash
# Enable IP forwarding in UPF
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  sysctl -w net.ipv4.ip_forward=1

# Add NAT rule if missing
kubectl exec -it -n omec $(kubectl get pods -n omec -l app=upf -o jsonpath='{.items[0].metadata.name}') -- \
  iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE

# Check DNN configuration in SMF
kubectl get configmap -n omec smf-config -o yaml | grep -A 5 dnn

# Restart UPF
kubectl rollout restart deployment upf -n omec
```

### Issue 6: Insufficient Resources

**Symptoms:**
- Pods pending or evicted
- "Insufficient CPU/memory" errors

**Diagnosis:**

```bash
# Check node resources
kubectl describe nodes | grep -A 10 "Allocated resources"

# Check pod resource requests
kubectl get pods -n omec -o custom-columns=NAME:.metadata.name,CPU_REQ:.spec.containers[*].resources.requests.cpu,MEM_REQ:.spec.containers[*].resources.requests.memory

# Check actual resource usage
kubectl top nodes
kubectl top pods -n omec
```

**Solutions:**

```bash
# Option 1: Reduce resource requests
kubectl edit deployment <deployment-name> -n omec
# Reduce resources.requests values

# Option 2: Upgrade EC2 instance
# Stop deployments
helm uninstall sd-core -n omec

# Change instance type in AWS Console or CLI
aws ec2 modify-instance-attribute \
  --instance-id <instance-id> \
  --instance-type t3.xlarge

# Restart instance and redeploy
```

### Issue 7: Network Interface Issues

**Symptoms:**
- Multus network attachment fails
- N2/N3 interfaces not created

**Diagnosis:**

```bash
# Check Multus installation
kubectl get pods -n kube-system | grep multus

# Check NetworkAttachmentDefinition
kubectl get network-attachment-definitions -n omec

# Check pod network annotations
kubectl get pod <pod-name> -n omec -o yaml | grep -A 10 annotations
```

**Solutions:**

```bash
# Reinstall Multus
kubectl delete -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml
kubectl apply -f https://raw.githubusercontent.com/k8snetworkplumbingwg/multus-cni/master/deployments/multus-daemonset.yml

# Recreate NetworkAttachmentDefinition
kubectl apply -f /path/to/network-attachment-definition.yaml

# Restart affected pods
kubectl delete pod <pod-name> -n omec
```

### Issue 8: Persistent Volume Issues

**Symptoms:**
- Pods can't mount volumes
- "FailedMount" errors

**Diagnosis:**

```bash
# Check PVs and PVCs
kubectl get pv
kubectl get pvc -A

# Describe PVC
kubectl describe pvc <pvc-name> -n omec

# Check storage class
kubectl get storageclass
```

**Solutions:**

```bash
# Delete and recreate PVC
kubectl delete pvc <pvc-name> -n omec
kubectl apply -f <pvc-definition.yaml>

# Check local-path-provisioner (for kind)
kubectl get pods -n local-path-storage

# Restart provisioner if needed
kubectl rollout restart deployment local-path-provisioner -n local-path-storage
```

## AWS-Specific Issues

### Issue 9: Security Group Configuration

**Problem:** Can't access services from external network

**Solution:**

```bash
# List current security group rules
aws ec2 describe-security-groups --group-ids <sg-id>

# Add missing rules
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port <port> \
  --cidr <cidr>

# For UDP ports
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol udp \
  --port 2152 \
  --cidr <cidr>
```

### Issue 10: EC2 Instance Network Limits

**Problem:** Network performance degradation

**Solution:**

```bash
# Check network metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name NetworkPacketsIn \
  --dimensions Name=InstanceId,Value=<instance-id> \
  --start-time 2023-01-01T00:00:00Z \
  --end-time 2023-01-01T23:59:59Z \
  --period 3600 \
  --statistics Average

# Consider upgrading to network-optimized instance
# t3.large -> t3.xlarge or c5n.large
```

### Issue 11: EBS Volume Performance

**Problem:** Slow I/O affecting database operations

**Solution:**

```bash
# Check volume IOPS
aws ec2 describe-volumes --volume-ids <volume-id>

# Modify volume type to gp3 or io2
aws ec2 modify-volume \
  --volume-id <volume-id> \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125
```

## Debugging Tools

### Packet Capture

```bash
# Install tcpdump in pod
kubectl exec -it -n omec <pod-name> -- \
  apt-get update && apt-get install -y tcpdump

# Capture traffic
kubectl exec -it -n omec <pod-name> -- \
  tcpdump -i any -w /tmp/capture.pcap

# Download capture
kubectl cp omec/<pod-name>:/tmp/capture.pcap ./capture.pcap

# Analyze with Wireshark locally
```

### Network Testing

```bash
# Deploy debug pod
kubectl run debug --image=nicolaka/netshoot -it --rm --restart=Never

# Inside debug pod:
# Test DNS
nslookup amf.omec.svc.cluster.local

# Test connectivity
curl http://amf.omec.svc.cluster.local:29518

# Test port
nc -zv amf.omec.svc.cluster.local 29518

# Trace route
traceroute amf.omec.svc.cluster.local
```

### Log Aggregation

```bash
# Get logs from all pods
kubectl logs -n omec --all-containers=true --tail=100 > all-logs.txt

# Follow logs from multiple pods
stern -n omec .

# Or using kubetail
kubetail -n omec
```

## Performance Monitoring

### Enable Metrics

```bash
# Install metrics-server
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Check metrics
kubectl top nodes
kubectl top pods -n omec

# Get detailed resource usage
kubectl get pods -n omec -o custom-columns=NAME:.metadata.name,CPU:.status.containerStatuses[*].resources.requests.cpu,MEM:.status.containerStatuses[*].resources.requests.memory
```

### Monitor Logs

```bash
# Real-time monitoring of specific component
kubectl logs -n omec -l app=amf -f | grep -i error

# Count errors
kubectl logs -n omec -l app=amf --tail=1000 | grep -c ERROR

# Get timestamps of errors
kubectl logs -n omec -l app=amf --timestamps | grep ERROR
```

## Getting Help

### Collect Debug Information

```bash
# Create debug bundle
mkdir -p /tmp/debug-bundle
kubectl get all -n omec > /tmp/debug-bundle/resources.txt
kubectl describe pods -n omec > /tmp/debug-bundle/pods-describe.txt
kubectl logs -n omec --all-containers=true --tail=500 > /tmp/debug-bundle/logs.txt
kubectl get events -n omec --sort-by='.lastTimestamp' > /tmp/debug-bundle/events.txt
kubectl get nodes -o yaml > /tmp/debug-bundle/nodes.yaml

# Compress
tar -czf debug-bundle.tar.gz /tmp/debug-bundle/
```

### Report Issues

When reporting issues, include:
1. Debug bundle
2. SD-Core version
3. Kubernetes version
4. EC2 instance type
5. Steps to reproduce
6. Expected vs actual behavior

## Additional Resources

- [Kubernetes Debugging Guide](https://kubernetes.io/docs/tasks/debug/)
- [UERANSIM Documentation](https://github.com/aligungr/UERANSIM)

## Next Steps

- [Configuration Reference](configuration.md) for advanced settings
- [RAN Connectivity](ran-connectivity.md) for RAN issues
- [SD-Core Deployment](sd-core-deployment.md) for core network issues
