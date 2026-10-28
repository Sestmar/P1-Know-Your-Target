# Mapa de superficie de ataque - Airbnb

- **Objetivo:** Airbnb, programa publico de bug bounty en HackerOne (ver `alcance.md` y `scope.txt`)
- **Fecha del reconocimiento:** 2026-10-28
- **Metodo:** solo tecnicas pasivas
- **Trazabilidad:** cada dato lleva entre corchetes la hora de su entrada en `cuaderno.md`. Por ejemplo, `[11:51]` es la entrada de las 11:51.

## 1. Resumen

Airbnb, Inc. es una empresa publica de San Francisco con entre 5.001 y 10.000 empleados `[18:20]`. Registro `airbnb.com` en 2008 con MarkMonitor, que es un registrador corporativo, y tiene el dominio bloqueado (Registry Lock) `[11:44]`.
La web publica (`www`, `api`, `assets`) esta detras de la CDN y el WAF de Akamai `[14:03]` `[18:45]`. La infraestructura propia vive en AWS (region `us-east-1`) y en Google Cloud `[18:45]`.
Sus herramientas internas (`admin`, `one`, `grafana`) son visibles desde Internet, pero estan protegidas con Google Identity-Aware Proxy, que solo deja entrar a cuentas `@airbnb.com` `[16:49]`.
El correo esta en Google Workspace, con SPF estricto y DMARC `p=reject`, y los empleados inician sesion mediante Okta `[11:51]`.
Lo mas relevante encontrado:
- Servicios de pago (`callbacks`, `worldpay.callbacks`) y el entorno `staging` expuestos directamente en AWS, sin Akamai delante `[18:45]`.
- Una guia publica con metadatos de su autor `[18:02]`.
- DNSSEC sin activar `[14:16]`.

## 2. Dominios y subdominios

Se encontraron 1.715 subdominios con `subfinder` `[12:04]`, 536 nombres en crt.sh (167 de ellos nuevos) `[13:59]` `[14:00]` y 336 hosts con theHarvester `[18:15]`. Esta tabla recoge los relevantes `[18:45]`:

| Subdominio | IP | En alcance | Para que parece servir |
|---|---|---|---|
| `www.airbnb.com` | 88.221.213.50, 104.90.205.x (Akamai) | Si | Web publica, detras de CDN/WAF `[14:03]`. A las 14:00 resolvia a 23.211.15.x: Akamai cambia la IP segun el momento y el nodo `[14:00]` |
| `api.airbnb.com` | 88.221.213.x (Akamai) | Si | API publica |
| `assets.airbnb.com` | 104.90.205.66 (Akamai) | Si | Ficheros estaticos y PDF publicos `[17:46]` |
| `next.airbnb.com` | 23.204.132.196 (Akamai) | Si | Version previa de la web; tiene copia por idioma (`es.next`...) `[18:15]` |
| `preprod.airbnb.com` | 88.221.213.113 (Akamai, `edgekey`) | Si | Entorno de preproduccion |
| `callbacks.airbnb.com` | 34.227.64.120 (AWS us-east-1) | Si | Retornos (callbacks) de proveedores externos, sin CDN |
| `worldpay.callbacks.airbnb.com` | 3.215.46.20 (AWS us-east-1) | Si | Callbacks del proveedor de pagos Worldpay, sin CDN |
| `ws.airbnb.com` | 54.90.191.231 (AWS us-east-1) | Si | Servicio web (ingress "kratos"); nginx, responde 404 `[17:00]` |
| `staging.airbnb.com` | 34.236.224.3 (AWS) | Si | Entorno de staging, sin CDN |
| `m.airbnb.com` | 13.248.132.14 / 76.223.34.100 (AWS Global Accelerator) | Si | Web movil |
| `git.airbnb.com` | 13.248.132.14 / 76.223.34.100 (AWS Global Accelerator) | Si | Repositorio de codigo (probable) |
| `deployboard.airbnb.com` | 13.248.132.14 / 76.223.34.100 (AWS Global Accelerator) | Si | Panel de despliegues (probable) |
| `developer.airbnb.com` | 76.223.34.100 / 13.248.132.14 (AWS Global Accelerator) | Si | Portal para desarrolladores |
| `admin.airbnb.com` | 136.107.58.148 (Google IAP) | Si | Panel de administracion, tras login de Google `[16:49]` |
| `admin-staging.airbnb.com` | 136.67.55.64 (Google IAP) | Si | Panel de administracion de staging, tras login de Google |
| `one.airbnb.com` | 136.107.58.148 (Google IAP) | Si | Herramienta interna (`internal-tools.iap`) |
| `grafana.iac.airbnb.com` | 34.160.96.171 (Google Cloud, IAP) | Si | Monitorizacion de infraestructura `[16:49]` |
| `support-api.airbnb.com` | No resuelve | Si | API de soporte (listada en HackerOne); solo interna o retirada |
| `internal.airbnb.com` | No resuelve | Si | Interno |
| `amex-mtls.airbnb.com` | No resuelve | Si | Integracion con American Express mediante TLS mutuo `[18:15]` |
| `image.mail.airbnb.com` | 23.223.100.86 (Akamai -> Salesforce) | **No** (excluido) | Imagenes del email marketing, activo de terceros `[18:30]` |

Patrones de nombres `[13:53]` `[18:15]`:
- `*.aws.inst.airbnb.com`: instancias de AWS.
- `*.iac(-dev).airbnb.com`: infraestructura como codigo y telefonia (`homer`, `sip-capture`, `rtpe`, `turn`).
- `*.staging`, `*.preprod`, `*.prototype*`: entornos que no son de produccion.
- `xx`, `xxlong`, `xxrtl`: idiomas de prueba para la traduccion.

Otros dominios vistos:
- `*.musta.ch` esta **en alcance**. Es un dominio interno de Airbnb con certificados a su nombre, por ejemplo `d.internal.us-east-1.prod.musta.ch` `[16:49]`. Todavia no se ha explorado.
- `airbnb.net` y `airbnb.biz` estan **fuera** del alcance. Aparecen como nombres intermedios en las cadenas CNAME (`*.origins.airbnb.net`, `internal-tools.iap.airbnb.biz`) `[14:00]` `[18:45]`.
- Los dominios de pais (`airbnb.com.ar`...) estan **fuera**: el enlace oficial que los listaria da 404 `[18:42]`.

## 3. Infraestructura y servicios

| Area | Hallazgo | Fuente |
|---|---|---|
| Registro | MarkMonitor, creado el 2008-08-05, caduca el 2028-08-05. Titular oculto (CA, US). Bloqueo client/server Delete/Transfer/Update | `[11:44]` |
| DNS | 8 servidores de nombres: 4 de NS1 y 4 de AWS Route 53 (redundancia con dos proveedores) | `[11:44]` `[11:51]` |
| DNSSEC | **No activo**. No hay registro DS; Netcraft dice "Enabled", pero se equivoca | `[14:16]` |
| CDN / WAF | Akamai (AS20940) delante de `www`, `api`, `assets`, `next` y `preprod`. Detras, el proxy Envoy | `[14:03]` `[14:07]` |
| Nube | AWS `us-east-1` (`callbacks`, `ws`, `staging`), AWS Global Accelerator (`m`, `git`, `deployboard`, `developer`) y Google Cloud (IAP) | `[18:45]` |
| Acceso interno | Google Identity-Aware Proxy (responde 302 al login de Google con `hd=airbnb.com`) | `[16:49]` |
| Cabeceras HTTP | HSTS (`includeSubDomains` y `preload`), CSP con informes, `X-Frame-Options` SAMEORIGIN, `X-Content-Type-Options` | `[14:07]` |
| Correo | Google Workspace (MX `aspmx.l.google.com`). SPF encadenado (`spf1` -> `spf2`) que termina en `-all`. Incluye Google y Amazon SES | `[11:51]` `[12:04]` |
| Antispoofing | DMARC `p=reject; sp=reject; pct=100`, con informes `rua` y `ruf` | `[11:51]` |
| Email marketing | Salesforce Marketing Cloud (ExactTarget) | `[17:00]` `[18:30]` |
| SaaS (registros TXT) | Okta (SSO por SAML), Microsoft 365, Atlassian, Cisco Webex, DocuSign, Miro, Canva, Adobe, 1Password, Stripe, Segment, DataDome, Palo Alto Networks, OpenAI, Anthropic... | `[11:51]` |
| SaaS (Hunter) | NetSuite, SAP, Zip, Plaid, DocuSign, Retool, Google Analytics, TikTok Pixel | `[18:30]` |
| Pagos | Worldpay (`worldpay.callbacks`), American Express (`amex-mtls`), Stripe y Plaid | `[18:15]` `[11:51]` `[18:30]` |
| Shodan | `hostname:airbnb.com` da 323 resultados, casi todos nodos de Akamai compartidos (ruido). La IP de `www` en InternetDB: solo puerto 80, etiqueta `cdn`, sin vulnerabilidades | `[14:19]` `[17:00]` |
| Ficheros expuestos | `robots.txt` revela rutas como `/api/v1/trebuchet`, `/api/v2/google_place_photos` o `/fix-it`. `security.txt` remite a HackerOne y es valido hasta 2027-06 | `[14:07]` |
| Google dorks | Muchos PDF en `assets.airbnb.com`. 0 resultados para `intitle:"index of"` y para `inurl:login` | `[17:46]` |
| Historico | Wayback Machine guarda mas de 5.000 URLs bajo `/api/`: APIs antiguas con version (`/api/feeds/1.0`, `/api/language/1.0`...) y endpoints como `AirlockApi`, `Batch` o `featureStatus`. Algunas URLs archivadas contienen codigos de reserva y una clave de API (no se reproducen aqui) | `[14:17]` `[19:05]` |
| Filtraciones | Have I Been Pwned no registra ninguna filtracion del dominio `airbnb.com` | `[19:06]` |
| Presencia publica | Sherlock da 149 plataformas con el usuario `airbnb`, la mayoria falsos positivos. Parecen oficiales GitHub (codigo abierto), Docker Hub, Hugging Face, Medium y YouTube | `[19:07]` |

## 4. Correos y personas

- **Dominio de correo:** `@airbnb.com`, sobre Google Workspace `[11:51]`.
- **Patron de correo:** no se pudo obtener. Hunter (plan gratuito) no lo muestra y theHarvester no encontro ningun correo con crtsh y duckduckgo; ademas, Bing ya no esta soportado en la version 4.11.1 `[18:13]` `[18:15]` `[18:20]`.
- **Volumen:** Hunter conoce 4.760 correos: 4.484 de personas, 414 de directivos y 276 genericos `[18:20]`.
- **Departamentos** (segun los correos conocidos): IT/Engineering 1.251, Operations 560, Management 341, Support 256, Sales 211, Legal 180, Marketing 175, Finance 173 `[18:30]`.
- **Identidad corporativa:** SSO con Okta (`airbnb.okta.com`, que es de terceros) y Google IAP para herramientas internas `[11:51]` `[16:49]`.
- **Metadatos:** la guia fiscal publica de EY (`assets.airbnb.com/eyguidance/es.pdf`) tiene como autor a una persona de EY (aqui solo "S. B."). Se hizo con Word 2013 en mayo de 2017 `[18:02]`.
- **Consecuencia:** el SPF estricto y el DMARC `reject` hacen muy dificil suplantar `@airbnb.com`. El riesgo de phishing se desplaza a dominios parecidos y al portal de Okta. Esto es solo una observacion: la ingenieria social esta prohibida en el programa.

## 5. Puntos de interes

Estos son los tres elementos que investigaria primero en la fase activa, siempre dentro del alcance y de las reglas del programa:

1. **`callbacks.airbnb.com` y `worldpay.callbacks.airbnb.com`** `[18:45]`
   - Reciben notificaciones de proveedores de pago y estan expuestos directamente en AWS, sin el WAF de Akamai.
   - `callbacks.airbnb.com` aparece explicitamente en la lista de activos de HackerOne y tiene 0 informes resueltos.
   - Habria que comprobar como valida el origen de las notificaciones (firmas, IP de origen).

2. **`staging.airbnb.com` y los entornos que no son de produccion** (`*.staging`, `preprod`, `admin-staging`, `dev.staging`) `[18:45]` `[13:53]`
   - `staging` resuelve directamente a AWS, sin CDN.
   - Los entornos de pruebas suelen tener controles mas debiles, datos de prueba y funciones a medio terminar.
   - Hay certificados comodin para `*.staging` y `*.dev.staging`, lo que indica que hay muchos hosts detras `[18:15]`.

3. **`git.airbnb.com` y `deployboard.airbnb.com`** `[18:45]`
   - Por el nombre, son un repositorio de codigo y un panel de despliegues: lo mas sensible de una organizacion.
   - A diferencia de `admin`, `one` o `grafana`, **no** apuntan a Google IAP, sino a AWS Global Accelerator.
   - No se sabe que autenticacion tienen. Es lo primero que habria que comprobar.

**Hallazgo para Manuel:**
- Wayback Machine conserva URLs de `/api/reservation/requested` con codigos de reserva reales y una clave de API del frontend `[19:05]`.
- No se han visitado esas URLs ni se reproducen los datos.
- Es informacion historica de un tercero (Internet Archive), pero se deja constancia segun las reglas del proyecto.

**Pendiente:**
- Explorar `*.musta.ch`, que esta en alcance y es de alto impacto.
- LinkedIn (tecnica opcional).
