External HTTPS (443)
    ↓
Istio Gateway (terminates TLS, decrypts to HTTP)
    ↓
VirtualService (routes HTTP traffic)
    ↓
Service: nginx-service-nodeport (port 443)
    ↓
Pod with Envoy sidecar
    ↓
Nginx container (port 443, configured for HTTPS)


# Istio Traffic Routing Setup Guide

## What is Istio?

**Istio** is a service mesh that provides:
- **Traffic Management**: Route traffic between services with advanced routing rules
- **Security**: mTLS encryption, authentication, and authorization
- **Observability**: Metrics, logs, and traces for your services
- **Policy Enforcement**: Rate limiting, quotas, and access control

## Core Istio Concepts

### 1. **Sidecar Proxy (Envoy)**
- Istio injects a sidecar container (Envoy proxy) into each pod
- This proxy intercepts all network traffic to/from your application
- No code changes needed - it works transparently

### 2. **Gateway**
- Defines how traffic enters the mesh from outside
- Similar to an Ingress controller
- Handles TLS termination and routing at the edge

### 3. **VirtualService**
- Defines routing rules for traffic within the mesh
- Can route based on URI, headers, source, etc.
- Enables A/B testing, canary deployments, traffic splitting

### 4. **DestinationRule**
- Defines policies for traffic to a service
- Load balancing algorithms (round-robin, least connections, etc.)
- Circuit breakers, connection pooling, outlier detection

## Setup Steps

### Prerequisites

1. **Install Istio** in your cluster:
   ```bash
   # Download Istio
   curl -L https://istio.io/downloadIstio | sh -
   cd istio-*
   
   # Install Istio with default profile
   istioctl install --set values.defaultRevision=default
   
   # Verify installation
   kubectl get pods -n istio-system
   ```

2. **Enable Istio in your namespace** (if not using automatic injection):
   ```bash
   kubectl label namespace raushan istio-injection=enabled
   ```

### Step 1: Enable Sidecar Injection

The deployment has been updated with the annotation:
```yaml
annotations:
  sidecar.istio.io/inject: "true"
```

This tells Istio to automatically inject the Envoy sidecar into your pods.

### Step 2: Apply Istio Resources

Apply the Istio configuration files in order:

```bash
# 1. Gateway - defines how traffic enters
kubectl apply -f istio-gateway.yaml

# 2. VirtualService - defines routing rules
kubectl apply -f istio-virtualservice.yaml

# 3. DestinationRule - defines traffic policies (optional but recommended)
kubectl apply -f istio-destinationrule.yaml
```

### Step 3: Update Your Service

Your existing service (`nginx-service-nodeport`) will work, but consider:

**Option A: Keep NodePort (Current)**
- Traffic: External → NodePort → Istio Gateway → VirtualService → Service → Pods
- Good for: Direct access without Istio Gateway

**Option B: Use ClusterIP with Istio Gateway (Recommended)**
- Change service type to `ClusterIP`
- Traffic: External → Istio Ingress Gateway → VirtualService → Service → Pods
- Better for: Full Istio control and features

### Step 4: Access Your Service

#### Via Istio Ingress Gateway:

1. **Get the Istio Ingress Gateway IP:**
   ```bash
   kubectl get svc -n istio-system istio-ingressgateway
   ```

2. **Access via the gateway:**
   ```bash
   # HTTP
   curl -H "Host: nginx.local" http://<INGRESS_GATEWAY_IP>/
   
   # HTTPS (if TLS configured)
   curl -k -H "Host: nginx.local" https://<INGRESS_GATEWAY_IP>/
   ```

#### Via NodePort (Current Setup):
   ```bash
   # Access directly via NodePort
   curl -k https://<NODE_IP>:30443/
   ```

## Traffic Flow Diagram

```
External Request
    ↓
Istio Gateway (istio-gateway.yaml)
    ↓
VirtualService (istio-virtualservice.yaml)
    ↓
DestinationRule (istio-destinationrule.yaml) - applies policies
    ↓
Kubernetes Service (nginx-service-nodeport)
    ↓
Pod with Envoy Sidecar
    ↓
Nginx Container
```

## Advanced Features You Can Use

### 1. **Canary Deployments**
Split traffic between versions:
```yaml
route:
- destination:
    host: nginx-service-nodeport
    subset: v1
  weight: 90
- destination:
    host: nginx-service-nodeport
    subset: v2
  weight: 10
```

### 2. **A/B Testing**
Route based on headers:
```yaml
match:
- headers:
    user-agent:
      regex: ".*Chrome.*"
route:
- destination:
    host: nginx-service-nodeport
    subset: chrome-version
```

### 3. **Circuit Breaker**
Already configured in DestinationRule - automatically stops sending traffic to unhealthy pods.

### 4. **mTLS (Mutual TLS)**
Enable encrypted communication between services:
```bash
kubectl apply -f - <<EOF
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: raushan
spec:
  mtls:
    mode: STRICT
EOF
```

## Troubleshooting

### Check if sidecar is injected:
```bash
kubectl get pods -n raushan
# Should show 2/2 containers (nginx + istio-proxy)
```

### View Istio configuration:
```bash
# Check Gateway
kubectl get gateway -n raushan

# Check VirtualService
kubectl get virtualservice -n raushan

# Check DestinationRule
kubectl get destinationrule -n raushan
```

### View Envoy proxy logs:
```bash
kubectl logs <pod-name> -c istio-proxy -n raushan
```

### Test routing:
```bash
# From within the cluster
kubectl exec -it <pod-name> -n raushan -- curl http://nginx-service-nodeport.raushan.svc.cluster.local:443
```

## Next Steps

1. **Monitor traffic**: Use Istio's built-in Grafana dashboards
2. **Add retries**: Configure retry policies in VirtualService
3. **Rate limiting**: Add rate limit policies
4. **Security policies**: Enable mTLS and authorization policies

## Important Notes

- The TLS secret (`nginx-tls-secret`) must exist in the `raushan` namespace
- The Gateway references this secret for TLS termination
- If you want Istio to handle TLS termination, you can remove TLS from nginx config
- The service name in VirtualService and DestinationRule must match your Kubernetes service name exactly