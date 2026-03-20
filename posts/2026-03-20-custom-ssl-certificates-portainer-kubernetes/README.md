# How to Set Up Custom SSL Certificates in Portainer on Kubernetes

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: Portainer, Kubernetes, SSL, TLS, Security

Description: Configure custom TLS certificates for Portainer on Kubernetes using Kubernetes Secrets and Ingress TLS termination.

---

On Kubernetes, TLS certificates for Portainer can be managed at two levels: at the Ingress layer (terminating TLS before it reaches Portainer) or directly at the Portainer pod level using Kubernetes Secrets.

## Method 1: Ingress TLS Termination (Recommended)

The cleanest approach is to terminate TLS at the Ingress controller and proxy HTTP or HTTPS to Portainer.

### Step 1: Create a Kubernetes TLS Secret

```bash
# Create a TLS secret from your certificate and key files

kubectl create secret tls portainer-tls \
  --cert=/path/to/portainer.crt \
  --key=/path/to/portainer.key \
  -n portainer

# Verify the secret was created
kubectl get secrets -n portainer
```

### Step 2: Create an Ingress Resource with TLS

```yaml
# portainer-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: portainer-ingress
  namespace: portainer
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/ssl-passthrough: "false"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - portainer.example.com
      secretName: portainer-tls   # Reference our TLS secret
  rules:
    - host: portainer.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: portainer
                port:
                  number: 9443
```

```bash
# Apply the Ingress
kubectl apply -f portainer-ingress.yaml
```

## Method 2: Mount Certificate Directly in the Pod

Pass the certificate files directly to the Portainer container via a Kubernetes Secret:

```yaml
# portainer-deployment-custom-tls.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: portainer
  namespace: portainer
spec:
  replicas: 1
  selector:
    matchLabels:
      app: portainer
  template:
    metadata:
      labels:
        app: portainer
    spec:
      containers:
        - name: portainer
          image: portainer/portainer-ce:latest
          args:
            - "--ssl"
            - "--sslcert"
            - "/certs/tls.crt"   # Kubernetes mounts secrets as tls.crt / tls.key
            - "--sslkey"
            - "/certs/tls.key"
          ports:
            - containerPort: 9443
          volumeMounts:
            - name: portainer-tls
              mountPath: /certs
              readOnly: true
            - name: portainer-data
              mountPath: /data
      volumes:
        - name: portainer-tls
          secret:
            secretName: portainer-tls   # Kubernetes TLS secret
        - name: portainer-data
          persistentVolumeClaim:
            claimName: portainer-pvc
```

## Method 3: cert-manager Integration

Use cert-manager to automate certificate issuance and renewal:

```yaml
# portainer-certificate.yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: portainer-cert
  namespace: portainer
spec:
  secretName: portainer-tls
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - portainer.example.com
```

```bash
# Apply the certificate resource
kubectl apply -f portainer-certificate.yaml

# Watch cert-manager issue the certificate
kubectl get certificate -n portainer -w
```

## Verify the Certificate

```bash
# Get the Portainer service external IP/hostname
kubectl get svc -n portainer

# Verify the certificate
openssl s_client -connect portainer.example.com:443 </dev/null 2>/dev/null | \
  openssl x509 -noout -subject -issuer -enddate
```

---

*Monitor SSL certificate expiry and Portainer on Kubernetes with [OneUptime](https://oneuptime.com).*
