# K8s-GatewayAPI-FW

# Kubernetes Gateway API Migration Guide
## Ingress'ten Gateway API'ye Geçiş Rehberi

---

## 📋 İçindekiler
1. [Gateway API Nedir?](#gateway-api-nedir)
2. [Neden Gateway API?](#neden-gateway-api)
3. [Temel Kavramlar](#temel-kavramlar)
4. [Ingress vs Gateway API Karşılaştırması](#ingress-vs-gateway-api)
5. [Migration Adımları](#migration-adımları)
6. [Gateway Controller Seçenekleri](#gateway-controller-seçenekleri)
7. [Best Practices](#best-practices)

---

## 🚀 Gateway API Nedir?

**Gateway API**, Kubernetes'in yeni nesil trafik yönlendirme çözümüdür. Ingress API'nin yerine geçmek üzere tasarlanmış, daha **rol-tabanlı**, **genişletilebilir** ve **standart** bir API'dir.

### Temel Özellikler:
- ✅ **Rol Odaklı Tasarım**: Platform operatörü, cluster operatörü ve uygulama geliştiricileri için ayrı kaynaklar
- ✅ **Tip Güvenliği**: Strongly-typed API (annotation yerine native fields)
- ✅ **Genişletilebilirlik**: Policy attachment pattern ile vendor-specific özellikler
- ✅ **Gelişmiş Routing**: Header-based, weighted routing, traffic splitting
- ✅ **Multi-Protocol**: HTTP, HTTPS, gRPC, TCP, UDP, TLS passthrough
- ✅ **Portable**: Farklı Gateway implementasyonları arasında taşınabilir

---

## 🎯 Neden Gateway API?

### Ingress'in Sınırlamaları:
```yaml
# ❌ Annotation ile konfigürasyon (vendor-specific)
metadata:
  annotations:
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
```

**Sorunlar:**
- Her vendor farklı annotation kullanır (nginx, traefik, istio)
- Type-safe değil, YAML string validation yok
- Dokümantasyon dağınık
- Karmaşık senaryolar için yetersiz

### Gateway API'nin Çözümü:
```yaml
# ✅ Native Kubernetes API ile konfigürasyon
spec:
  validation:
    wellKnownCACertificates: System
  timeout:
    request: 60s
```

**Avantajlar:**
- Standart API, tüm implementasyonlarda çalışır
- Kubernetes API validation
- Auto-complete ve type checking
- Resmi dokümantasyon

---

## 📚 Temel Kavramlar

### 1. **GatewayClass** (Infrastructure Level)
Cluster yöneticisi tarafından tanımlanır. Gateway'in hangi controller tarafından yönetileceğini belirtir.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx
spec:
  controllerName: nginx.org/gateway-controller
```

**Analoji:** StorageClass gibi - hangi implementasyon kullanılacağını belirtir.

**Kim Kullanır:** Cluster Admin

---

### 2. **Gateway** (Infrastructure Level)
Load balancer/proxy'nin kendisi. Hangi portları dinleyeceği, TLS ayarları vb.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: my-tls-cert
```

**Analoji:** Ingress Controller'ın kendisi (nginx-ingress deployment)

**Kim Kullanır:** Platform Operator / Cluster Admin

**Özellikler:**
- Birden fazla listener (HTTP, HTTPS, TCP, UDP)
- TLS termination
- Namespace izolasyonu
- Cross-namespace routing (RBAC ile kontrollü)

---

### 3. **HTTPRoute** (Application Level)
Uygulama seviyesinde HTTP routing kuralları. Ingress rules'un gelişmiş versiyonu.

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app-route
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "myapp.example.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-service
      port: 8080
```

**Analoji:** Ingress rules

**Kim Kullanır:** Application Developer

**Özellikler:**
- Path, header, query parameter matching
- Traffic splitting (A/B testing, canary deployments)
- Request/response header manipulation
- Redirects ve rewrites

---

### 4. **BackendTLSPolicy** (Security)
Backend servise nasıl güvenli bağlanılacağını tanımlar.

```yaml
apiVersion: gateway.networking.k8s.io/v1alpha3
kind: BackendTLSPolicy
metadata:
  name: backend-tls
spec:
  targetRefs:
  - kind: Service
    name: secure-backend
    port: 443
  validation:
    wellKnownCACertificates: System
    hostname: backend.namespace.svc.cluster.local
```

**Analoji:** Ingress'teki `backend-protocol: HTTPS` annotation'ının native versiyonu

**Kullanım Senaryosu:** Backend serviste self-signed certificate varsa

---

### 5. **Diğer Route Tipleri**

#### **TLSRoute** - TLS Passthrough
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TLSRoute
spec:
  parentRefs:
  - name: my-gateway
    sectionName: tls-passthrough
  hostnames:
  - "secure.example.com"
  rules:
  - backendRefs:
    - name: tls-backend
      port: 8443
```

**Kullanım:** TLS termination yapmadan backend'e iletmek için (end-to-end encryption)

#### **TCPRoute** - TCP Routing
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: TCPRoute
spec:
  parentRefs:
  - name: my-gateway
    sectionName: tcp
  rules:
  - backendRefs:
    - name: database-service
      port: 5432
```

**Kullanım:** PostgreSQL, MySQL gibi TCP servisler

#### **UDPRoute** - UDP Routing
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: UDPRoute
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: dns-service
      port: 53
```

**Kullanım:** DNS, QUIC, gaming servers

#### **GRPCRoute** - gRPC Routing
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: GRPCRoute
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "grpc.example.com"
  rules:
  - matches:
    - method:
        service: my.service.v1.MyService
        method: GetUser
    backendRefs:
    - name: grpc-backend
      port: 9090
```

**Kullanım:** gRPC servisleri için method-level routing

---

## 🔄 Ingress vs Gateway API Karşılaştırması

### Liman Örneği:

#### ❌ ESKİ: Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: liman
  annotations:
    # Vendor-specific annotations
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"
    nginx.ingress.kubernetes.io/proxy-ssl-verify: "off"
    nginx.ingress.kubernetes.io/proxy-body-size: "100m"
spec:
  ingressClassName: nginx
  rules:
  - host: "liman"
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: limancore
            port:
              number: 443
```

**Sorunlar:**
- ❌ Annotation'lar portable değil (sadece nginx için)
- ❌ Backend TLS ayarları Ingress spec'inde değil
- ❌ HTTP → HTTPS redirect için ayrı annotation gerekir
- ❌ Advanced routing özellikleri yok
- ❌ TLS ayarları sınırlı

---

#### ✅ YENİ: Gateway API
```yaml
# 1. Gateway - Altyapı Seviyesi
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: liman-gateway
spec:
  gatewayClassName: nginx
  listeners:
  - name: http
    protocol: HTTP
    port: 80
  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate
      certificateRefs:
      - name: liman-tls-cert

---
# 2. HTTPRoute - Uygulama Seviyesi
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: liman-route
spec:
  parentRefs:
  - name: liman-gateway
    sectionName: https
  hostnames:
  - "liman"
  - "liman.local"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /
    backendRefs:
    - name: limancore
      port: 443
      weight: 100  # Traffic splitting için

---
# 3. BackendTLSPolicy - Backend Güvenlik
apiVersion: gateway.networking.k8s.io/v1alpha3
kind: BackendTLSPolicy
metadata:
  name: liman-backend-tls
spec:
  targetRefs:
  - kind: Service
    name: limancore
    port: 443
  validation:
    wellKnownCACertificates: System
    hostname: limancore.liman.svc.cluster.local

---
# 4. HTTP → HTTPS Redirect
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: liman-http-redirect
spec:
  parentRefs:
  - name: liman-gateway
    sectionName: http
  hostnames:
  - "liman"
  rules:
  - filters:
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301
```

**Avantajlar:**
- ✅ Rol bazlı separation of concerns
- ✅ Native Kubernetes API (type-safe)
- ✅ Portable (nginx, istio, traefik, cilium hepsi destekler)
- ✅ Her concern için ayrı resource
- ✅ Advanced routing (weights, headers, redirects)

---

## 🔧 Migration Adımları

### Adım 1: Gateway API CRD'lerini Yükle
```bash
# Standard Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# Experimental features için (TLSRoute, TCPRoute, UDPRoute)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/experimental-install.yaml
```

### Adım 2: Gateway Controller Seç ve Yükle

#### Seçenek 1: NGINX Gateway Fabric (Önerilen - NGINX'ten geçiş için)
```bash
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/latest/download/crds.yaml
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/latest/download/nginx-gateway.yaml
```

#### Seçenek 2: Istio Gateway
```bash
istioctl install --set profile=minimal
```

#### Seçenek 3: Traefik
```bash
helm repo add traefik https://traefik.github.io/charts
helm install traefik traefik/traefik \
  --set gateway.enabled=true \
  --set ingressRoute.enabled=false
```

#### Seçenek 4: Cilium (eBPF, yüksek performans)
```bash
helm install cilium cilium/cilium \
  --set gatewayAPI.enabled=true \
  --set kubeProxyReplacement=strict
```

### Adım 3: GatewayClass Doğrula
```bash
kubectl get gatewayclass

# Çıktı:
# NAME    CONTROLLER                     ACCEPTED   AGE
# nginx   nginx.org/gateway-controller   True       5m
```

### Adım 4: Gateway ve Routes Deploy Et
```bash
# Helm ile
helm upgrade --install liman . -f values-gateway.yaml -n liman

# veya manuel
kubectl apply -f liman-gateway.yaml -n liman
```

### Adım 5: Test Et
```bash
# Gateway durumu
kubectl get gateway -n liman
kubectl describe gateway liman-gateway -n liman

# Routes
kubectl get httproute -n liman
kubectl describe httproute liman-route -n liman

# Backend TLS Policy
kubectl get backendtlspolicy -n liman
```

### Adım 6: Eski Ingress'i Sil (Test Sonrası)
```bash
# Önce gateway'in çalıştığından emin ol
kubectl delete ingress liman -n liman
```

---

## 🎛️ Gateway Controller Seçenekleri

| Controller | Kullanım Senaryosu | Performans | Özellikler | Öğrenme Eğrisi |
|-----------|-------------------|------------|-----------|---------------|
| **NGINX Gateway Fabric** | NGINX Ingress'ten geçiş | ⭐⭐⭐⭐ | Basit, güvenilir | Düşük |
| **Istio Gateway** | Service mesh, mTLS, observability | ⭐⭐⭐⭐ | En zengin feature set | Yüksek |
| **Traefik** | Dinamik konfigürasyon, Let's Encrypt | ⭐⭐⭐⭐ | Auto-discovery, dashboard | Orta |
| **Cilium** | eBPF, yüksek performans, network policy | ⭐⭐⭐⭐⭐ | Kernel-level, ultra-fast | Orta-Yüksek |
| **Envoy Gateway** | Cloud-native, extensible | ⭐⭐⭐⭐ | Modern, standart-compliant | Orta |
| **Kong Gateway** | API management, plugins | ⭐⭐⭐⭐ | Enterprise features | Orta |
| **HAProxy Kubernetes Gateway** | Enterprise-grade LB | ⭐⭐⭐⭐⭐ | Proven reliability | Orta |

### Liman İçin Öneri:
**NGINX Gateway Fabric** - Mevcut NGINX Ingress bilginizi kullanabilir, migration en kolay

---

## 💡 Best Practices

### 1. Rol Ayrımı
```yaml
# Platform Operator: Gateway oluşturur
---
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: gateway-system  # Ayrı namespace
spec:
  gatewayClassName: nginx
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    allowedRoutes:
      namespaces:
        from: All  # Tüm namespace'lerden route kabul et

---
# App Developer: Sadece HTTPRoute oluşturur
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app-namespace
spec:
  parentRefs:
  - name: shared-gateway
    namespace: gateway-system  # Cross-namespace reference
  hostnames:
  - "myapp.example.com"
  rules:
  - backendRefs:
    - name: my-service
      port: 80
```

### 2. ReferenceGrant ile Güvenlik
Cross-namespace erişim için izin gerekir:

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-liman-to-gateway
  namespace: gateway-system
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: liman
  to:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: shared-gateway
```

### 3. Traffic Splitting (Canary Deployment)
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - backendRefs:
    - name: stable-backend
      port: 80
      weight: 90  # %90 stable
    - name: canary-backend
      port: 80
      weight: 10  # %10 canary
```

### 4. Header-Based Routing
```yaml
rules:
- matches:
  - headers:
    - name: X-Beta-User
      value: "true"
  backendRefs:
  - name: beta-backend
    port: 80
- backendRefs:
  - name: stable-backend
    port: 80
```

### 5. Request Header Manipulation
```yaml
rules:
- filters:
  - type: RequestHeaderModifier
    requestHeaderModifier:
      add:
      - name: X-Forwarded-By
        value: "gateway"
      remove:
      - X-Internal-Header
  backendRefs:
  - name: backend
    port: 80
```

### 6. Path Rewriting
```yaml
rules:
- matches:
  - path:
      type: PathPrefix
      value: /old-api
  filters:
  - type: URLRewrite
    urlRewrite:
      path:
        type: ReplacePrefixMatch
        replacePrefixMatch: /new-api
  backendRefs:
  - name: api-backend
    port: 80
```

### 7. Timeout ve Retry Policies
```yaml
# Gateway implementasyonuna göre değişir
# Örnek: Istio için
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: BackendPolicy
metadata:
  name: timeout-policy
spec:
  targetRefs:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: my-route
  override:
    timeout:
      request: 30s
      backendRequest: 15s
    retry:
      attempts: 3
      perTryTimeout: 5s
```

---

## 🚦 Monitoring ve Troubleshooting

### Status Kontrolü
```bash
# Gateway status
kubectl get gateway liman-gateway -n liman -o yaml

# Status altında:
# status:
#   conditions:
#   - type: Accepted
#     status: "True"
#   - type: Programmed
#     status: "True"
#   addresses:
#   - type: IPAddress
#     value: 10.0.0.100
```

### HTTPRoute Status
```bash
kubectl get httproute liman-route -n liman -o yaml

# status:
#   parents:
#   - parentRef:
#       name: liman-gateway
#     conditions:
#     - type: Accepted
#       status: "True"
#     - type: ResolvedRefs
#       status: "True"
```

### Debug
```bash
# Gateway logs
kubectl logs -n nginx-gateway -l app=nginx-gateway

# Events
kubectl get events -n liman --sort-by='.lastTimestamp'

# Describe için tüm detaylar
kubectl describe gateway liman-gateway -n liman
kubectl describe httproute liman-route -n liman
kubectl describe backendtlspolicy liman-backend-tls -n liman
```

---

## 📈 Gelişmiş Senaryolar

### Çoklu Domain Hosting
```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: multi-domain
spec:
  parentRefs:
  - name: my-gateway
  rules:
  # Domain 1
  - matches:
    - headers:
      - name: :authority  # HTTP/2 header
        value: domain1.com
    backendRefs:
    - name: domain1-service
      port: 80

  # Domain 2
  - matches:
    - headers:
      - name: :authority
        value: domain2.com
    backendRefs:
    - name: domain2-service
      port: 80
```

### Blue-Green Deployment
```yaml
# Green (yeni versiyon)
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: app-green
spec:
  parentRefs:
  - name: my-gateway
  hostnames:
  - "app.example.com"
  rules:
  - backendRefs:
    - name: app-v2  # Yeni versiyon
      port: 80

# Switch yapmak için sadece backendRef'i değiştir:
# app-v2 → app-v1 (rollback)
```

### Geo-based Routing (Custom Policy ile)
```yaml
# Implementasyon-specific (örnek: Istio)
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: BackendPolicy
metadata:
  name: geo-routing
spec:
  targetRefs:
  - kind: HTTPRoute
    name: my-route
  override:
    geoRouting:
      - region: eu-west
        backend:
          name: eu-backend
      - region: us-east
        backend:
          name: us-backend
```

---

## 🔐 Security Best Practices

### 1. mTLS (Mutual TLS)
```yaml
apiVersion: gateway.networking.k8s.io/v1alpha3
kind: BackendTLSPolicy
metadata:
  name: mtls-policy
spec:
  targetRefs:
  - kind: Service
    name: secure-backend
  validation:
    caCertificateRefs:
    - name: backend-ca
      kind: ConfigMap
    hostname: backend.svc.cluster.local
  # Client certificate
  clientCertificateRef:
    name: client-cert
    kind: Secret
```

### 2. Rate Limiting
```yaml
# Implementasyon-specific
apiVersion: gateway.networking.k8s.io/v1alpha2
kind: BackendPolicy
metadata:
  name: rate-limit
spec:
  targetRefs:
  - kind: HTTPRoute
    name: api-route
  override:
    rateLimit:
      requests: 100
      unit: minute
      perClientIP: true
```

### 3. CORS Policy
```yaml
rules:
- filters:
  - type: ResponseHeaderModifier
    responseHeaderModifier:
      add:
      - name: Access-Control-Allow-Origin
        value: "*"
      - name: Access-Control-Allow-Methods
        value: "GET, POST, OPTIONS"
```

---

## 🎓 Öğrenme Kaynakları

### Resmi Dokümantasyon
- [Gateway API Docs](https://gateway-api.sigs.k8s.io/)
- [API Reference](https://gateway-api.sigs.k8s.io/reference/spec/)
- [Implementations](https://gateway-api.sigs.k8s.io/implementations/)

### Implementation-Specific
- [NGINX Gateway Fabric](https://docs.nginx.com/nginx-gateway-fabric/)
- [Istio Gateway](https://istio.io/latest/docs/tasks/traffic-management/ingress/gateway-api/)
- [Traefik](https://doc.traefik.io/traefik/routing/providers/kubernetes-gateway/)
- [Cilium](https://docs.cilium.io/en/stable/network/servicemesh/gateway-api/)

### Topluluk
- [Kubernetes Slack #sig-network-gateway-api](https://kubernetes.slack.com/)
- [GitHub Discussions](https://github.com/kubernetes-sigs/gateway-api/discussions)

---

## ⚡ TL;DR - Hızlı Başlangıç

```bash
# 1. CRD'leri yükle
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.0.0/standard-install.yaml

# 2. NGINX Gateway Fabric yükle
kubectl apply -f https://github.com/nginxinc/nginx-gateway-fabric/releases/latest/download/nginx-gateway.yaml

# 3. Gateway deploy et
kubectl apply -f liman-gateway.yaml -n liman

# 4. Test et
kubectl get gateway,httproute -n liman

# 5. Eski Ingress'i sil
kubectl delete ingress liman -n liman
```

---

## 📊 Özet Tablo

| Özellik | Ingress | Gateway API |
|---------|---------|-------------|
| **Maturity** | GA (stable) | v1 (stable), bazı özellikler alpha/beta |
| **Portability** | Düşük (annotation-based) | Yüksek (spec-based) |
| **Type Safety** | Hayır (YAML strings) | Evet (native fields) |
| **Role Separation** | Hayır | Evet (GatewayClass, Gateway, Route) |
| **Protocol Support** | HTTP/HTTPS | HTTP/HTTPS/gRPC/TCP/UDP/TLS |
| **Traffic Splitting** | Zor (annotation) | Native (weights) |
| **Header Routing** | Hayır | Evet |
| **Request Transformation** | Vendor-specific | Standardized filters |
| **Cross-Namespace** | Hayır | Evet (ReferenceGrant ile) |
| **Future** | Maintenance mode | Aktif geliştirme |

---

## 🏁 Sonuç

Gateway API, Kubernetes'teki trafik yönetiminin geleceğidir. Ingress'e göre:

✅ **Daha güçlü**: Advanced routing, traffic splitting, multi-protocol
✅ **Daha güvenli**: Type-safe API, native policies
✅ **Daha portable**: Vendor-agnostic, standart API
✅ **Daha esnek**: Extensible policy attachment

**Migrasyon şimdi başlamalı** - 2026'da Ingress NGINX kaldırılacak!
