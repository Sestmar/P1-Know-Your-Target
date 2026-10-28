# Cuaderno de recon - airbnb.com

Parte 1: recon MANUAL. Parte 2: mismo objetivo con el agente (aap-base).
Parte 3: comparativa.md (yo vs agente).

Progreso Parte 1 (segun enunciado "Proyecto 1 · Know Your Target"):

Preparacion
- [x] 1.2 alcance.md (enlace HackerOne + fecha, dentro/fuera, reglas) y scope.txt con formato del enunciado

Tecnicas OBLIGATORIAS (las 8)
- [x] Registro del dominio: whois
- [x] Registros DNS: dig (ns, mx, txt, _dmarc)
- [x] Subdominios: subfinder (1715) · crt.sh (536) · alcance de los relevantes en mapa.md
- [x] Huella tecnologica: Netcraft
- [x] Ficheros expuestos: robots.txt y security.txt
- [x] Servicios expuestos: Shodan (web) + InternetDB
- [x] Google dorks
- [x] Correos: theHarvester (0 correos, 336 hosts) · Hunter.io (patron no visible en plan gratuito)

Tecnicas OPCIONALES
- [x] Resolver y clasificar (dig www + whois IP) -> Akamai
- [x] Wayback Machine (14:17 caido; 19:05 OK, 5000 URLs de /api/)
- [x] exiftool (guia EY: autor, Word 2013, 2017)
- [ ] LinkedIn (departamentos y tecnologias)
- [x] Have I Been Pwned (sin filtraciones del dominio)
- [x] Sherlock (149 resultados, mayoria falsos positivos)

Entregables Parte 1
- [x] cuaderno.md (este fichero)
- [x] mapa.md (resumen, tabla subdominios/IP/alcance, infraestructura, correos y personas, 3 puntos de interes)

Formato de cada entrada:

```
AAAA-MM-DD HH:MM · objetivo: <dominio> (in scope)
$ <comando>
-> <resultado resumido>
siguiente: <que hacer con lo encontrado>
```

---

```
2026-10-28 11:44 · objetivo: airbnb.com (in scope)
$ whois airbnb.com
-> Registrador: MarkMonitor Inc. (registrador corporativo, anti-secuestro de dominios)
-> Creado: 2008-08-05 · Actualizado: 2026-07-04 · Caduca: 2028-08-05
-> Titular: datos REDACTED FOR PRIVACY (solo se ve State: CA, Country: US)
-> Estado: client/server Delete/Transfer/Update Prohibited (dominio bloqueado: Registry Lock)
-> DNS: 4x NS1 (dnsN.p08.nsone.net) + 4x AWS Route 53 (ns-*.awsdns-*) -> DNS redundante con 2 proveedores
-> DNSSEC: unsigned (no firmado)
siguiente: dig +short ns/mx/txt/_dmarc para confirmar los NS y ver correo, SPF y DMARC
```

```
2026-10-28 11:51 · objetivo: airbnb.com (in scope)
$ dig +short ns airbnb.com
-> 8 NS: 4x NS1 + 4x AWS Route 53 (coincide con whois)

$ dig +short mx airbnb.com
-> aspmx.l.google.com + alt1-4 -> correo en Google Workspace

$ dig +short txt airbnb.com
-> SPF: include:spf1.airbnb.com + rangos IPv6 de Google + 4 rangos IPv4 propios, termina en -all (estricto)
   ip4: 98.77.0.0/16 · 87.253.232.0/21 · 76.223.176.0/20 · 76.223.128.0/19
-> SSO corporativo: Okta (airbnb.okta.com, SAML)
-> Proveedores SaaS verificados (huella tecnologica):
   Google, Microsoft 365 (MS=), Atlassian, Cisco Webex, Neat Pulse (salas de reuniones),
   DocuSign, Miro, Canva, Adobe, 1Password, Stripe, Segment, DataDome (anti-bot),
   Palo Alto Networks, Apple, Facebook, Yahoo, OpenAI, Anthropic, ElevenLabs,
   Gradle, LucidLink, MASV, Razorpay, WeWork, Status page (x2), Ecostruxure IT...

$ dig +short txt _dmarc.airbnb.com
-> p=reject; sp=reject; pct=100 -> postura antispoofing maxima (correo falso se rechaza)
-> informes a dmarc.aggregate@ y dmarc.forensic@airbnb.com

siguiente: resolver spf1.airbnb.com (dig +short txt spf1.airbnb.com) y whois de los rangos IPv4;
           pasar a subdominios (subfinder / crt.sh)
```

```
2026-10-28 12:04 · objetivo: airbnb.com (in scope)
$ dig +short txt spf1.airbnb.com
-> SPF encadenado: spf1 -> include:spf2.airbnb.com (cadena airbnb.com -> spf1 -> spf2)
-> 11 rangos IPv4 de envio: 74.125.0.0/16 (Google), 54.240.0.0/18 y 54.240.64.0/18 (Amazon SES),
   resto pendiente de identificar con whois (probables plataformas de email marketing)
-> CORRECCION de 11:51: los rangos IPv4 del SPF son de proveedores que envian correo en nombre
   de Airbnb, no infraestructura propia
-> termina en -all (estricto)

$ subfinder -d airbnb.com -silent | tee subdominios.txt | wc -l
-> 1715 subdominios (fuentes pasivas, sin verificar si siguen vivos) guardados en subdominios.txt
siguiente: filtrar subdominios interesantes (dev, staging, admin, vpn, api...) y priorizar
```

```
2026-10-28 13:53 · objetivo: airbnb.com (in scope)
$ grep -iE 'dev|stag|test|qa|uat|admin|internal|corp|vpn|jenkins|git|jira|grafana|kibana|okta|sso|api' subdominios.txt | sort | head -150
-> OJO: sort+head = solo A..i del abecedario, no los mas interesantes, solo los primeros
-> Patrones de nomenclatura deducidos:
   *.aws.inst.airbnb.com   -> instancias AWS internas (con entornos qa01, use1qa...)
   *.next.airbnb.com       -> plataforma/entorno "next" (gitlab, git, api, dev...)
   *.iac.airbnb.com / iac-dev -> Infrastructure as Code (grafana, homer)
   admin-lite / admin-next / admin-staging -> familia de paneles de administracion
-> Momento IA - prioridad (solo para RESOLVER IP y clasificar, NO entrar):
   ALTA: git.airbnb.com, gitlab.next.airbnb.com, grafana.iac.airbnb.com,
         admin.airbnb.com, api-private.airbnb.com, corp.airbnb.com, developer.airbnb.com
   MEDIA: *.staging / *.preprod / *.uat (entornos no productivos)
   BAJA: hashes aleatorios *.https.dev.aws.inst (efimeros, probablemente muertos)
siguiente: crt.sh (segundo comando de la diapositiva de subdominios) y comparar con subfinder
```

```
2026-10-28 13:59 · objetivo: airbnb.com (in scope)
$ curl -s 'https://crt.sh/?q=%25.airbnb.com&output=json' | jq -r '.[].name_value' | sort -u | tee crtsh.txt | wc -l
-> 536 nombres en certificados SSL (guardados en crtsh.txt)
-> subfinder: 1715 -> crt.sh es solo UNA de las fuentes que usa subfinder
-> incluye certificados wildcard (*.algo.airbnb.com) que revelan zonas enteras
siguiente: comparar crtsh.txt con subdominios.txt (cuantos aporta crt.sh que subfinder no tenia)
```

```
2026-10-28 14:00 · objetivo: airbnb.com (in scope)
$ grep -c '^\*' crtsh.txt
-> 57 certificados wildcard (*.zona.airbnb.com)
$ sed 's/^\*\.//' crtsh.txt | sort -u | comm -13 <(sort -u subdominios.txt) - | wc -l
-> 167 nombres en crt.sh que subfinder NO tenia -> conviene combinar fuentes

$ dig +short www.airbnb.com
-> cadena CNAME:
   www.airbnb.com
   -> airbnb-web.production.global.product.origins.airbnb.net   (nombre del origen, dominio airbnb.net)
   -> san.airbnb.com.edgekey.net                                (Akamai)
   -> e118243.a.akamaiedge.net                                  (Akamai)
   -> 9 IPs 23.211.15.132-161
-> conclusion: www esta DETRAS DE CDN/WAF (Akamai): hablas con el escudo, no con el servidor
-> OJO alcance: airbnb.net NO esta en scope.txt -> no se investiga sin confirmarlo en HackerOne
siguiente: whois 23.211.15.132 | grep -iE 'orgname|netname|country' para confirmar Akamai
```

```
2026-10-28 14:03 · objetivo: airbnb.com (in scope)
$ whois 23.211.15.132 | grep -iE 'orgname|netname|country'
-> NetName: AKAMAI · OrgName: Akamai Technologies, Inc. · Country: US
-> CONFIRMADO: www.airbnb.com detras de CDN/WAF Akamai (la IP real del origen no se ve)
siguiente: paso 5 Netcraft (navegador) y paso 6 robots.txt / security.txt
```

```
2026-10-28 14:07 · objetivo: airbnb.com (in scope)
Netcraft: https://sitereport.netcraft.com/?url=https://www.airbnb.com  (capturas guardadas)
-> Visto por primera vez: marzo 2009 · Site rank: 659
-> Hosting: Akamai Technologies, IP 95.100.98.75 (EU), AS20940 (Akamai), reverse DNS *.akamaitechnologies.com
   (IP distinta a la nuestra 23.211.15.x: la CDN da a cada uno el nodo mas cercano)
-> Nameserver ns-558.awsdns-05.net · Registrar MarkMonitor · Organisation: Airbnb, Inc., US
-> DNSSEC: Netcraft dice "Enabled" pero whois dice "unsigned" -> CONTRADICCION, verificar con dig ds
-> SPF / DMARC "no presentes": Netcraft miro www.airbnb.com, no airbnb.com (alli SI existen, paso 2)
-> Tecnologia: Envoy (proxy, detras de Akamai) · CDN Akamai · Google Custom Search · HTML5, JS, gzip
-> Cabeceras de seguridad: HSTS (includeSubDomains + preload), CSP + report, X-Frame-Options SAMEORIGIN,
   X-Content-Type-Options, X-XSS-Protection -> buena postura
-> Geolocalizacion: rate limit de Netcraft (sin dato)

$ curl -s https://www.airbnb.com/robots.txt | head -60
-> Banner ASCII + mensaje de reclutamiento a ingenieros
-> Rutas interesantes: /api/v1/trebuchet, /api/v2/google_place_photos, /fix-it, /fixit,
   /alumni, /home/dashboard, /manage-listing, /embeddable, /disaster/lookup
-> Allow: /.well-known/amphtml/apikey.pub (clave publica AMP, publica por diseño)
-> El resto son rutas privadas de usuario (account, inbox, logout...) -> evitar indexacion, no secretos
-> Solo vistas 60 lineas (seccion Googlebot)

$ curl -s https://www.airbnb.com/.well-known/security.txt
-> Contact/Policy: hackerone.com/airbnb -> canal oficial de reporte = HackerOne (confirma programa)
-> Expires: 2027-06-01 (vigente) · Preferred-Languages: en
siguiente: comprobar DNSSEC (dig +short ds airbnb.com) y paso 7 Wayback Machine
```

```
2026-10-28 14:16 · objetivo: airbnb.com (in scope)
$ dig +short ds airbnb.com
-> vacio: no hay registro DS en el .com -> DNSSEC NO activo
-> whois (unsigned) tenia razon; Netcraft ("Enabled") se equivoca -> contrastar siempre fuentes
siguiente: paso 7 Wayback Machine
```

```
2026-10-28 14:17 · objetivo: airbnb.com (in scope)
$ curl -s 'https://web.archive.org/cdx/search/cdx?url=airbnb.com/api/*&collapse=urlkey&fl=original&limit=5000' | tee wayback_api.txt | wc -l
-> 14 lineas, pero NO son URLs: es la pagina HTML "Internet Archive: Temporarily Offline"
-> Wayback Machine caida en este momento -> SIN DATO, reintentar mas tarde
-> leccion: comprobar el contenido, no solo el numero de lineas (las 14 lineas eran HTML, no URLs)
siguiente: paso 8 Shodan; reintentar Wayback al final
```

```
2026-10-28 14:19 · objetivo: airbnb.com (in scope)
$ curl -s https://internetdb.shodan.io/23.211.15.132
-> hostname: a23-211-15-132.deploy.static.akamaitechnologies.com
-> ports: [80] · tags: ["cdn"] · vulns: [] · cpes: []
-> Shodan la etiqueta como CDN (Akamai): sin vulnerabilidades conocidas ni software identificado
-> describe el ESCUDO (Akamai), no los servidores de Airbnb
-> 443 no aparece: internetdb es un resumen y puede estar incompleto (la web va por HTTPS)
siguiente: shodan.io (web) con hostname:airbnb.com y org:"Airbnb"
```

```
2026-10-28 16:49 · objetivo: airbnb.com (in scope)
shodan.io, busqueda de texto libre: airbnb.com   (capturas guardadas, solo 1a pagina)
-> 43 resultados · Paises: US 39, Finland 4 · Puertos: 443 (35), 80 (8)
-> Organizaciones: Google LLC 25, Amazon 7+4+3, Hetzner 4 · Productos: AWS ELB 9, nginx 1
-> HALLAZGO: IPs de Google Cloud (34.107.232.111, 34.160.96.171, 34.117.239.225) con
   certificado *.admin.airbnb.com emitido a Airbnb, Inc. y hostnames:
   admin-lite, admin-staging, admin-lite-staging, dev.staging, prototype-next .airbnb.com
   -> son paneles internos EXPUESTOS DIRECTAMENTE (sin Akamai)...
   -> ...pero responden 302 a accounts.google.com/o/oauth2 con hd=airbnb.com
      = Google Identity-Aware Proxy (IAP): solo entra quien tenga cuenta Google @airbnb.com
   -> buena practica: el subdominio se ve, pero el panel esta detras de autenticacion
-> Dominios relacionados con certificado a nombre de Airbnb, Inc. (FUERA DE SCOPE, solo se anotan):
   musta.ch (d.internal.us-east-1.prod.musta.ch...) -> parece dominio interno de Airbnb
   airbnb.biz · urbandoor.com (redirige 301 a airbnb.com, empresa adquirida)
-> Ruido: thegiftease.com, newdb.gitr.com (Hetzner) -> solo mencionan "airbnb", no son de Airbnb
-> Nota: algunos hosts fuera de scope aceptan TLSv1/1.1 (obsoletos)

2026-10-28 17:00 · objetivo: airbnb.com (in scope)
shodan.io, filtro: hostname:airbnb.com   (capturas guardadas, solo 1a pagina)
-> 323 resultados · Paises: US 147, Germany 18, India 18, Brazil 11, UK 9
-> Puertos: 443 (320), 80, 2083, 8443 (1 cada uno)
-> Organizaciones: Akamai 242+8, Amazon 33+15, Google 9 · Productos: AkamaiGHost 250, nginx 20
-> La mayoria: nodos Akamai COMPARTIDOS con certificado akamai-san63.exacttarget.com (Salesforce)
   y hostnames image.email.<otras marcas> -> Airbnb usa Salesforce Marketing Cloud (ExactTarget)
   para su email marketing; esas IPs sirven a muchas empresas -> no son de Airbnb (ruido)
-> ws.airbnb.com: 166.117.186.152 (Amazon, Seattle), nginx, 404, solo TLSv1.2,
   certificado a Airbnb, Inc. tambien para ws.airbnb.co.kr/.pt/.com.ph/.com.ec/.com.ee
   -> servicio propio en AWS sin Akamai delante (probable endpoint de "web services")
-> Pendiente: ver que hay en los puertos 2083 y 8443 (solo en Shodan, sin conectar)
siguiente: paso 9 Google dorks
```

```
2026-10-28 17:46 · objetivo: airbnb.com (in scope)
Google: site:airbnb.com filetype:pdf   (capturas guardadas)
-> Muchos PDF, casi todos en assets.airbnb.com/help/ (servidor de ficheros estaticos)
-> Guias fiscales por pais (Lituania, Hungria, Grecia, Chile 2024...), Terms of Service (2018),
   folleto turistico "Goa Unseen" (www.airbnb.com/goaunseen)
-> Llamativo: "Guia general de tributacion..." en assets.airbnb.com/eyguidance, elaborada por EY,
   dice "preparada para uso UNICO de Airbnb y no deberia ser compartida" -> pero esta publica e indexada
   (probablemente intencionado para anfitriones, pero se anota)

Google: site:airbnb.com intitle:"index of"
-> 0 resultados: no hay listados de directorios abiertos indexados (bien)

Google: site:airbnb.com inurl:login
-> 0 resultados: las paginas de login no estan indexadas (bien; robots.txt/noindex hacen su trabajo)
siguiente: descargar 2-3 PDF y pasarlos por exiftool (paso 10)
```

```
2026-10-28 18:02 · objetivo: assets.airbnb.com (in scope, *.airbnb.com)
$ wget "https://assets.airbnb.com/eyguidance/es.pdf" && exiftool *.pdf
-> assets.airbnb.com resuelve a 104.90.205.66 y 88.221.213.113 (rangos de Akamai, pendiente confirmar con whois)
-> PDF 1.5, 28 paginas, idioma en-US, Title: "I"
-> Author: nombre de una persona (anonimizado aqui: "S. B."), probablemente de EY, no de Airbnb
-> Creator/Producer: Microsoft Word 2013 -> documento hecho en Word y exportado a PDF
-> Create/Modify Date: 2017-05-20 09:39 (+02:00 -> zona horaria de Europa central, coherente con EY España)
-> Fecha del fichero en servidor: 2026-09-15 (resubido/actualizado en assets, contenido de 2017)
-> Leccion: los metadatos revelan nombre de autor, software y zona horaria aunque el texto no los diga
siguiente: theHarvester (ultima tecnica obligatoria)
```

```
2026-10-28 18:13 · objetivo: airbnb.com (in scope)
$ theHarvester -d airbnb.com -b crtsh,duckduckgo,bing
-> ERROR: "engines not supported: {'bing'}" -> theHarvester 4.11.1 ya no tiene Bing (diapositiva desactualizada)
siguiente: repetir sin bing

2026-10-28 18:15 · objetivo: airbnb.com (in scope)
$ theHarvester -d airbnb.com -b crtsh,duckduckgo
-> 0 emails · 0 personas · 0 IPs · 336 hosts
-> Correos: theHarvester no encuentra ninguno con estas fuentes -> completar con Hunter.io
-> Hosts: casi todos ya vistos en subfinder/crt.sh; lo nuevo es entender los PATRONES:
   - Idiomas: es., fr., de., ja., zh-cn.... y su copia en *.next.airbnb.com
     -> "next" parece la version previa (pre-release) de la web publica, no solo un entorno dev
   - xx., xxkey., xxlong., xxrtl. -> locales de prueba (pseudo-traduccion, textos largos, derecha-izquierda)
   - Wildcards: *.internal, *.staging, *.preprod, *.prototype-internal, *.dev.staging
   - Telefonia/VoIP en iac(-dev): homer, sip-capture, rtpe, turn + airphone-poc
     -> infraestructura de llamadas (probablemente atencion al cliente)
   - Pagos: worldpay.callbacks(.staging), amex-mtls(-staging) -> integra Worldpay y American Express
   - Operaciones: deployboard, git, grafana(-iap), ogma, internal
   - Corporativo: careers, investors, newsroom, press, transparency, brand
-> Dominios de pais airbnb.com.ar/.au/.br... aparecen pero NO son *.airbnb.com -> FUERA de scope.txt
siguiente: Hunter.io (patron de correo); despues alcance.md con la politica de HackerOne
```

```
2026-10-28 18:20 · objetivo: airbnb.com (in scope)
Hunter.io Domain Search: https://hunter.io/search/airbnb.com (cuenta gratuita; captura guardada en local, no se sube)
-> 4.760 correos conocidos: personas 4.484 · directivos 414 · genericos 276
-> Empresa: Airbnb, publica, 5.001-10.000 empleados, sede en Brannan Street, San Francisco (CA)
-> Dominio de correo de los empleados: @airbnb.com (coincide con MX de Google Workspace, paso DNS)
-> Patron de correo: no visible en el plan gratuito (correos ocultos con asteriscos)
-> No se copian nombres ni correos individuales (datos personales)
siguiente: leer la politica y la tabla de activos de HackerOne para cerrar el alcance

2026-10-28 18:22 · objetivo: programa Airbnb en HackerOne
Lectura de la politica: https://hackerone.com/airbnb (actualizada 2026-08-27)
-> Bug bounty publico desde 2015, maximo 25.000 USD, 100% eficiencia de respuesta
-> En alcance (alto impacto): *.airbnb.com, *.airbnb.org, *.musta.ch, *.airbnbpayments.com
-> CORRECCION: musta.ch SI esta en alcance (antes anotado como fuera; scope.txt era provisional)
   -> los hosts *.musta.ch vistos en Shodan (d.internal.us-east-1.prod.musta.ch...) pasan a estar en alcance
-> Siguen fuera: airbnb.net, airbnb.biz, urbandoor.com, activos de terceros (Okta, Akamai, Salesforce)
-> Dominios de pais (airbnb.com.ar...): ambiguo ("sitios localizados") -> fuera hasta confirmar
-> Reglas: sin ingenieria social, sin cuentas en masa, sin fuerza bruta, no degradar servicio
-> Resultado: alcance.md y scope.txt definitivos
siguiente: confirmar que image.mail.airbnb.com es de terceros (dig CNAME) para justificar la exclusion
```

```
2026-10-28 18:30 · objetivo: image.mail.airbnb.com (EXCLUIDO en scope.txt, solo resolucion DNS)
$ dig +short image.mail.airbnb.com
-> akamai-san63.exacttarget.com.edgekey.net -> e13974.dscx.akamaiedge.net -> 23.223.100.86
-> CONFIRMADO: lo sirve Salesforce Marketing Cloud (ExactTarget) via Akamai = activo de terceros
-> exclusion !image.mail.airbnb.com justificada; enlaza con lo visto en Shodan (certificado exacttarget)

Hunter.io (cont.): departamentos con correos conocidos
-> IT/Engineering 1.251 · Operations & Logistics 560 · Management 341 · Support 256 · Sales 211
   Writing & Communication 188 · Legal 180 · Marketing 175 · Finance 173 · Arts & Design 129
   Human Resources 109 · Education 85 · Medical & Health 23
-> Ingenieria es el departamento mas grande (~26% de los correos conocidos)
Hunter.io (cont.): tecnologias (fuente TechLookup, solo una muestra)
-> Finanzas: NetSuite, SAP, Zip · Publicidad: Google Ads, TikTok Pixel
-> Analitica: Ahrefs, Google Analytics, Spark · Autenticacion/firma: DocuSign, Plaid
-> Gestion: Kakao, Retool
-> Coincide en parte con los TXT de verificacion (DocuSign) -> dos fuentes independientes
siguiente: patron de correo en Hunter y pestana Alcance de HackerOne; despues mapa.md
```

```
2026-10-28 18:35 · objetivo: programa Airbnb en HackerOne
Lectura de la tabla de activos: https://hackerone.com/airbnb/policy_scopes
-> Confirma *.airbnb.com, *.musta.ch, *.airbnbpayments.com, *.airbnb.org (Higher Impact)
-> Lista explicitamente activos vistos en el recon: assets, callbacks, api, next, one, m, support-api
-> Sitios localizados: lista en https://www.airbnb.com/sitemaps/localized
-> Ineligible (fuera): demo/admin.demo/provider.demo.urbandoor.com, luckeyhomes.com, luckey.partners/.in/.fr/.app
-> Discrepancia: *.airbnbopen.com sale en el texto de la politica pero no en la tabla -> fuera
-> Mas informes resueltos: www.airbnb.com 389 (23%) y *.airbnb.com 361 (21%)
-> scope.txt y alcance.md actualizados con la tabla oficial
siguiente: comprobar que dominios de pais incluye sitemaps/localized; despues mapa.md
```

```
2026-10-28 18:42 · objetivo: www.airbnb.com (in scope)
$ curl -s https://www.airbnb.com/sitemaps/localized | grep -oE 'https?://[a-z0-9.-]+' | sort -u
-> salida vacia; en el navegador la URL redirige a www.airbnb.com/404 ("Oops! Error code: 404")
-> el enlace oficial de HackerOne para los sitios localizados ESTA ROTO (la tabla es de 2023)
-> no se puede confirmar si los dominios de pais (airbnb.es, airbnb.com.ar...) estan en alcance
   -> se mantienen FUERA (ante la duda, no se toca)
siguiente: resolver IP de los subdominios clave para la tabla del mapa; redactar mapa.md
```

```
2026-10-28 18:45 · objetivo: 20 subdominios de *.airbnb.com (in scope)
$ for s in www api assets callbacks support-api next one m admin admin-staging git grafana.iac deployboard internal ws worldpay.callbacks amex-mtls developer staging preprod; do
    echo "== $s.airbnb.com"; dig +short $s.airbnb.com | tail -n 3
  done | tee ips_clave.txt
-> Akamai (CDN): www, api, assets, next, preprod (88.221.213.x, 104.90.205.x, 23.204.132.x;
   assets -> akamaiedge, preprod -> airbnb.edgekey.net)
-> AWS directo, region us-east-1, via *.ingress.airbnb.net:
   callbacks -> 34.227.64.120 · worldpay.callbacks -> 3.215.46.20 · ws (ingress "kratos") -> 54.90.191.231
-> AWS directo sin CNAME: staging -> 34.236.224.3
-> AWS Global Accelerator (a9170e05f136aeb06.awsglobalaccelerator.com, 13.248.132.14 / 76.223.34.100):
   m, git, deployboard, developer
-> Google IAP via airbnb.biz: one, admin -> internal-tools.iap.airbnb.biz (136.107.58.148)
   admin-staging -> 136.67.55.64 · grafana.iac -> iap-generic01-global-prod.airbnb.biz (34.160.96.171, visto en Shodan)
-> Sin respuesta publica: support-api, internal, amex-mtls (solo internos o retirados)
-> ws: Shodan lo vio en 166.117.186.152 y ahora resuelve a 54.90.191.231 -> la IP ha cambiado
-> Nombres intermedios en airbnb.net / airbnb.biz: FUERA de alcance, solo se anotan
-> Resultado guardado en ips_clave.txt
siguiente: redactar mapa.md
```

```
2026-10-28 19:05 · objetivo: airbnb.com (in scope)
$ curl -s 'https://web.archive.org/cdx/search/cdx?url=airbnb.com/api/*&collapse=urlkey&fl=original&limit=5000' | tee wayback_api.txt | wc -l
$ head -30 wayback_api.txt
-> Wayback ya funciona: 5000 URLs (= el limit, hay MAS archivadas)
-> APIs antiguas por version: /api/annotations/1.0, /api/earth/1.0, /api/feeds/1.0, /api/language/1.0, /api/picker/1.0
-> Endpoints con nombre: /api/AirlockApi, /api/Batch, /api/featureStatus, /api/core_guest_spa_data, /api/create_widget, /api/clpost
-> DATO SENSIBLE: URLs /api/reservation/requested?code=<codigo de reserva>&key=<clave API>
   -> codigos de reserva reales y una clave de API del frontend quedaron archivados en Wayback
   -> NO se copian aqui (sustituidos por <...>), NO se visitan esas URLs
   -> segun las reglas del proyecto: anotado sin el dato y comunicado al profesor
-> wayback_api.txt contiene esos datos -> NO se sube al repositorio sin redactar
-> 20:18: redactado con sed -E 's/(code|key)=[^&]*/\1=<REDACTED>/g' (comprobado: 0 valores reales,
   sin tokens/correos/sesiones en otros parametros). Copia redactada en evidencias/; el original se queda en Kali
siguiente: Have I Been Pwned

2026-10-28 19:06 · objetivo: airbnb.com (in scope)
$ curl -s "https://haveibeenpwned.com/api/v3/breaches?domain=airbnb.com"
-> [] : ninguna filtracion registrada en HIBP para el dominio airbnb.com
-> limitacion: buscar los correos @airbnb.com en filtraciones de terceros exige verificar la propiedad del dominio
siguiente: Sherlock

2026-10-28 19:07 · objetivo: alias "airbnb" (cuentas corporativas)
$ sherlock airbnb --print-found
-> 149 plataformas con un usuario "airbnb" (Sherlock 0.16.0)
-> La mayoria NO son de la empresa: Sherlock solo comprueba que el nombre existe
   (Chess, Roblox, RuneScape, Minecraft, Duolingo... nombre ocupado por cualquiera)
-> Falsos positivos claros: Discord (devuelve solo https://discord.com), LinkedIn /in/airbnb
   (la pagina de empresa seria /company/airbnb)
-> Probablemente oficiales (por coherencia con la marca, sin verificar): GitHub, Docker Hub, Hugging Face,
   Medium, YouTube, Threads, Snapchat, Dribbble, Behance, SpeakerDeck, HackerOne (confirmado: es el programa)
-> GitHub github.com/airbnb: codigo abierto de la empresa -> fuente para tecnologias que usan
siguiente: actualizar mapa.md
```
