# Instalación de Gateway API + Traefik en MicroK8s

Guía corregida para tener tu clúster MicroK8s accesible desde fuera con el dominio `palomicius.local`, usando la Gateway API oficial y Traefik como controlador.

---

## Paso 1: Instalar los CRDs oficiales de la Gateway API

⚠️ **No uses CRDs escritos a mano.** Los oficiales son mantenidos por Kubernetes SIG Network y son los únicos que Traefik reconoce correctamente.

Si ya aplicaste un archivo `gateway-api-crds.yaml` casero, bórralo primero:

```bash
microk8s kubectl delete -f gateway-api-crds.yaml
```

Instala los CRDs oficiales directamente desde el repositorio de Kubernetes:

```bash
microk8s kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml
```

Verifica que se crearon:

```bash
microk8s kubectl get crds | grep gateway
```

Deberías ver `gatewayclasses.gateway.networking.k8s.io`, `gateways.gateway.networking.k8s.io`, `httproutes.gateway.networking.k8s.io`, entre otros.

---

## Paso 2: Añadir el repositorio de Helm de Traefik

```bash
microk8s helm repo add traefik https://traefik.github.io/charts
microk8s helm repo update
```

---

## Paso 3: Instalar Traefik con soporte de Gateway API

```bash
microk8s helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set providers.kubernetesGateway.enabled=true \
  --set ingressClass.enabled=false
```

---

## Paso 4: Crear el archivo de red (`mi-red.yaml`)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mi-gateway-traefik
  namespace: traefik
spec:
  gatewayClassName: traefik
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: All
---
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ruta-mi-app
  namespace: default # Cambia esto si tu aplicación corre en otro namespace
spec:
  parentRefs:
  - name: mi-gateway-traefik
    namespace: traefik
  hostnames:
  - "palomicius.local" # <-- Tu dominio local
  rules:
  - backendRefs:
    - name: tu-servicio-actual # <-- REEMPLAZA con el nombre real de tu Service
      port: 80 # Puerto en el que escucha tu Service
```

Aplícalo:

```bash
microk8s kubectl apply -f mi-red.yaml
```

---

## Paso 5: Validación final

Comprueba que MetalLB le asignó IP externa a Traefik:

```bash
microk8s kubectl get svc -n traefik
```

En la columna `EXTERNAL-IP` de la línea de Traefik debería aparecer la IP asignada por MetalLB (por ejemplo `127.0.0.1` si así lo configuraste).

Si es así, añade la entrada correspondiente en tu `/etc/hosts` (si no usas DNS local) y abre en el navegador:

```
http://palomicius.local
```

---

## Notas

- Si `EXTERNAL-IP` se queda en `<pending>`, revisa que MetalLB esté correctamente instalado y con un rango de IPs configurado (`microk8s kubectl get ipaddresspools -n metallb-system`).
- Revisa los logs de Traefik si la ruta no responde:
  ```bash
  microk8s kubectl logs -n traefik deploy/traefik
  ```