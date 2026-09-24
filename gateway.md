# Guía de Configuración: Gateway API con Traefik en Kubernetes Local

Esta guía te permitirá configurar el estándar moderno de Kubernetes (**Gateway API**) utilizando **Traefik** como tu API Gateway receptor. Esta arquitectura te permitirá recibir tráfico desde tu DNS externo hoy, dejando el entorno preparado para añadir proxies especializados en IA (como *agentgateway*) en el futuro sin modificar tu red básica.

---

## Paso 1: Registrar las estructuras de la Gateway API de forma local

Para evitar problemas con URLs externas o recortes en la terminal, utilizaremos un manifiesto local para dar de alta las definiciones de recursos personalizados (**CRDs**) oficiales del estándar de Kubernetes.

1. Crea un archivo llamado `gateway-api-crds.yaml`.
2. Pega el siguiente contenido dentro del archivo:

```yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: gatewayclasses.gateway.networking.k8s.io
spec:
  group: gateway.networking.k8s.io
  names:
    kind: GatewayClass
    listKind: GatewayClassList
    plural: gatewayclasses
    singular: gatewayclass
  scope: Cluster
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [gatewayClassName, listeners]
            properties:
              gatewayClassName: {type: string}
              listeners:
                type: array
                items:
                  type: object
                  required: [name, protocol, port]
                  properties:
                    name: {type: string}
                    protocol: {type: string}
                    port: {type: integer}
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: gateways.gateway.networking.k8s.io
spec:
  group: gateway.networking.k8s.io
  names:
    kind: Gateway
    listKind: GatewayList
    plural: gateways
    singular: gateway
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              gatewayClassName: {type: string}
              listeners:
                type: array
                items:
                  type: object
                  properties:
                    name: {type: string}
                    protocol: {type: string}
                    port: {type: integer}
                    allowedRoutes:
                      type: object
                      properties:
                        namespaces:
                          type: object
                          properties:
                            from: {type: string}
---
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: httproutes.gateway.networking.k8s.io
spec:
  group: gateway.networking.k8s.io
  names:
    kind: HTTPRoute
    listKind: HTTPRouteList
    plural: httproutes
    singular: httproute
  scope: Namespaced
  versions:
  - name: v1
    served: true
    storage: true
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            properties:
              parentRefs:
                type: array
                items:
                  type: object
                  properties:
                    name: {type: string}
                    namespace: {type: string}
              hostnames:
                type: array
                items: {type: string}
              rules:
                type: array
                items:
                  type: object
                  properties:
                    backendRefs:
                      type: array
                      items:
                        type: object
                        properties:
                          name: {type: string}
                          port: {type: integer}
```

3. Aplica las definiciones en tu clúster ejecutando:
```bash
kubectl apply -f gateway-api-crds.yaml
```

---

## Paso 2: Instalar Traefik con soporte de Gateway API mediante Helm

Configuraremos Traefik para que actúe explícitamente bajo las reglas de la Gateway API en lugar del modo Ingress tradicional. Ejecuta secuencialmente estos comandos en tu terminal:

```bash
# 1. Registrar y actualizar el repositorio oficial de Traefik
helm repo add traefik https://github.io
helm repo update

# 2. Instalar Traefik habilitando el proveedor de la Gateway API
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --set providers.kubernetesGateway.enabled=true \
  --set ingressClass.enabled=false
```

---

## Paso 3: Configurar el punto de entrada y el enrutamiento (`mi-red.yaml`)

Crea un archivo de configuración unificado llamado `mi-red.yaml`. Este creará el recurso `Gateway` (el receptor físico del tráfico) y el recurso `HTTPRoute` (la regla lógica que vincula tu DNS externo con tu aplicación).

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: mi-gateway-traefik
  namespace: traefik
spec:
  gatewayClassName: traefik # Indica a K8s que Traefik controlará este puerto
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
  namespace: default # Reemplaza por el namespace donde corre tu aplicación actual si no es el default
spec:
  parentRefs:
  - name: mi-gateway-traefik
    namespace: traefik
  hostnames:
  - "tu-dns-externo.com" # <--- REEMPLAZA CON TU DOMINIO REAL (Ej: mi-app.local o tu-web.com)
  rules:
  - backendRefs:
    - name: tu-servicio-actual # <--- REEMPLAZA CON EL NOMBRE DEL 'SERVICE' DE TU APLICACIÓN
      port: 80 # El puerto expuesto por el Service de tu aplicación
```

Aplica esta configuración ejecutando:
```bash
kubectl apply -f mi-red.yaml
```

---

## Paso 4: Exposición de puertos según tu entorno local

Para que las peticiones del exterior entren correctamente a Traefik, debes habilitar el canal de red de tu clúster local. Ejecuta el comando correspondiente al software que utilices:

* **Si usas Minikube:** Abre un túnel para asignar IPs reales de balanceador abriendo una terminal independiente y ejecutando:
  ```bash
  minikube tunnel
  ```
* **Si usas Kind o Docker Desktop:** Traefik intentará enlazarse automáticamente al puerto 80 de tu localhost. Si necesitas forzar la conexión o redirigir de forma manual, ejecuta:
  ```bash
  kubectl port-forward deployment/traefik 8080:80 -n traefik
  ```

---

## El Futuro: ¿Cómo añadir el `agentgateway` en esta misma infraestructura?

Cuando decidas desplegar tus agentes de IA o conectar servidores MCP, no tendrás que modificar tu DNS ni tu Gateway principal. Solo deberías realizar dos pasos:

1. Instalar `agentgateway` dentro de tu clúster.
2. Añadir un nuevo recurso `HTTPRoute` (o expandir el actual) en tu archivo `mi-red.yaml` indicando que todo el tráfico dirigido a un prefijo específico se envíe al servicio del proxy de IA:

```yaml
# Ejemplo conceptual del HTTPRoute adicional en el futuro
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: ruta-agentes-ia
  namespace: default
spec:
  parentRefs:
  - name: mi-gateway-traefik
    namespace: traefik
  hostnames:
  - "tu-dns-externo.com"
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /v1/ai # Cualquier petición a ://tu-dns-externo.com ira a la IA
    backendRefs:
    - name: servicio-agentgateway # El servicio del proxy de IA
      port: 8080
```
