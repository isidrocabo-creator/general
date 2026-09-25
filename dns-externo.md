# Enlazar tu dominio propio (IP dinámica) a tu clúster MicroK8s

Tu situación: dominio propio registrado + IP pública dinámica + acceso al router.
Como tu IP cambia, necesitas un servicio de **DNS dinámico (DDNS)** que la mantenga actualizada, y luego apuntar tu dominio real a ese DDNS.

---

## Resumen del flujo

```
Internet → tu-dominio.com → CNAME → xxxx.duckdns.org → IP pública actual de tu casa
→ Router (port forwarding 80/443) → PC con MicroK8s → Traefik → Gateway → HTTPRoute → nginx-demo
```

---

## Paso 1: Elegir y configurar un proveedor de DDNS

Como tu IP es dinámica, tu router (o un script en tu PC) tiene que avisar constantemente a un servicio de la IP actual.

Opciones gratuitas más comunes:
- **DuckDNS** (muy simple, recomendado para empezar): https://www.duckdns.org
- **No-IP**: https://www.noip.com
- **Cloudflare + script propio** (si tu dominio ya usa Cloudflare como DNS, puedes automatizar la actualización tú misma con la API de Cloudflare en vez de depender de un DDNS de terceros)

### Con DuckDNS (ejemplo):
1. Entra en https://www.duckdns.org y accede con GitHub/Google.
2. Crea un subdominio, por ejemplo: `palomicius.duckdns.org`.
3. Copia el **token** que te da la web.
4. Comprueba si tu **router** soporta DuckDNS/No-IP nativamente (Configuración → DDNS, en muchos routers domésticos). Si lo soporta, actívalo ahí directamente — es la opción más fiable porque el router siempre conoce su IP pública real.
5. Si tu router NO lo soporta, puedes actualizar la IP con un cron job en tu PC:
   ```bash
   crontab -e
   ```
   Añade (sustituyendo `TUDOMINIO` y `TUTOKEN`):
   ```
   */5 * * * * curl -s "https://www.duckdns.org/update?domains=TUDOMINIO&token=TUTOKEN&ip=" >/dev/null 2>&1
   ```
   Esto actualiza la IP cada 5 minutos.

---

## Paso 2: Apuntar tu dominio real al DDNS

En el panel de tu proveedor de dominio (donde lo compraste: Namecheap, GoDaddy, Cloudflare, IONOS, etc.), añade un registro:

```
Tipo:   CNAME
Nombre: @ (o "www", o un subdominio como "home")
Valor:  palomicius.duckdns.org
```

> ⚠️ Nota: algunos proveedores no permiten CNAME en el registro raíz (`@`). En ese caso usa un subdominio, p. ej. `casa.tudominio.com → CNAME → palomicius.duckdns.org`, o mira si tu proveedor soporta "ALIAS"/"ANAME" para el raíz.

Espera la propagación DNS (puede tardar de minutos a un par de horas). Compruébalo con:
```bash
nslookup casa.tudominio.com
```
Debería devolver tu IP pública actual.

---

## Paso 3: Port forwarding en el router

Entra en la administración de tu router (normalmente `192.168.1.1` o `192.168.0.1`) y busca la sección **Port Forwarding / Reenvío de puertos / Virtual Server**.

Necesitas reenviar hacia la IP local de tu PC (la que corre MicroK8s):

| Puerto externo | Puerto interno | Protocolo | IP destino (tu PC) |
|---|---|---|---|
| 80  | 80  | TCP | IP local de tu PC, ej. `192.168.1.50` |
| 443 | 443 | TCP | IP local de tu PC |

Para saber la IP local de tu PC:
```bash
ip addr show | grep "inet " | grep -v 127.0.0.1
```

> 💡 Recomendable: asigna una **IP fija/reservada** a tu PC en el router (DHCP reservation), para que el port forwarding no se rompa si tu PC recibe otra IP local tras un reinicio.

### Notas específicas para router de Telefónica (HGU)

1. **Acceso a la interfaz**: normalmente en `http://192.168.1.1`, con las credenciales de la etiqueta del router, o vía el portal **Alejandra** / app **Smart WiFi** de Movistar.
2. **Dónde está la opción**: busca **"Puertos"** en Alejandra, o **NAT → Port Forwarding / Port Mapping** en la interfaz web clásica del HGU (el nombre exacto varía según el modelo: MitraStar, Comtrend, etc.).
3. **Elige apertura Manual**: indica la IP local de tu PC (la reservada por DHCP), el puerto (80 y 443) y protocolo **TCP**.
4. **Fija la IP local por DHCP desde el propio router** (reserva DHCP), no manualmente en el PC — es la causa más común de que el port forwarding "no funcione" aunque esté bien configurado.
5. **DDNS integrado en el router**: muchos HGU de Telefónica permiten configurar DuckDNS/No-IP directamente en su interfaz, sin depender de un cron job en tu PC. Revisa si tu modelo lo soporta — es más fiable porque el router siempre conoce su IP pública real.
6. **Descarta CG-NAT**: compara la IP pública que muestra la web de configuración del router (sección WAN/Estado) con la que te devuelve https://www.cual-es-mi-ip.net.
   - Si coinciden → tienes IP pública real, el port forwarding funcionará normalmente.
   - Si no coinciden → estás detrás de CG-NAT; tendrás que llamar a Movistar para solicitar IP pública, o usar un túnel (ej. Cloudflare Tunnel) en vez de exponer el puerto directamente.

---

## Paso 4: Ajustar el Gateway/HTTPRoute con el nuevo hostname

Edita tu `mi-red.yaml` para incluir tu dominio real junto (o en lugar de) `palomicius.local`:

```yaml
hostnames:
- "palomicius.local"        # para pruebas locales
- "casa.tudominio.com"      # tu dominio real
```

Aplica:
```bash
microk8s kubectl apply -f mi-red.yaml
```

---

## Paso 5: Verificación desde fuera de tu red

**Importante:** no puedes probar el acceso externo desde dentro de tu propia red (problema de "NAT loopback", muchos routers no lo soportan). Prueba:
- Desde datos móviles (WiFi desactivado) en tu teléfono.
- O con una herramienta online como https://www.yougetsignal.com/tools/open-ports/ para comprobar si el puerto 80 responde.

```bash
curl -v http://casa.tudominio.com
```

---

## Paso 6 (recomendado, para más adelante): HTTPS con Let's Encrypt

Una vez esto funcione en HTTP, el siguiente paso natural es añadir TLS automático con **cert-manager** + Let's Encrypt, para que `https://casa.tudominio.com` tenga certificado válido. Lo dejamos para otra sesión.

---

## Cosas a tener en cuenta / seguridad

- Abrir puertos 80/443 al exterior expone tu app a Internet real — asegúrate de que lo que sirvas ahí (de momento, nginx de prueba) no tenga datos sensibles hasta que valides bien la seguridad.
- Considera un firewall (ufw) en tu PC limitando el tráfico entrante solo a esos puertos.
- Si tu ISP bloquea el puerto 80/443 entrante (algunos lo hacen en planes residenciales), el port forwarding no funcionará aunque esté bien configurado — en ese caso tocaría usar un túnel (Cloudflare Tunnel, ngrok, etc.) en vez de exponer el puerto directamente.

---

## Checklist para retomar mañana

- [ ] Cuenta creada en DuckDNS (o proveedor DDNS elegido) y subdominio generado
- [ ] Router configurado con DDNS nativo, o cron job funcionando
- [ ] Registro CNAME creado en el panel de tu dominio
- [ ] `nslookup` confirma que el dominio resuelve a tu IP pública
- [ ] IP local de tu PC reservada/fija en el router
- [ ] Port forwarding 80 y 443 configurado en el router
- [ ] `mi-red.yaml` actualizado con el hostname real
- [ ] Prueba de acceso externo (desde datos móviles) exitosa