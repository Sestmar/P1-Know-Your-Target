# Alcance - Airbnb (HackerOne)

- **Programa:** Airbnb Bug Bounty en HackerOne - <https://hackerone.com/airbnb>
- **Lista oficial de activos:** <https://hackerone.com/airbnb/policy_scopes>
- **Tipo:** bug bounty publico y activo (recompensas desde febrero de 2015, maximo 25.000 USD)
- **Consultado:** 2026-10-28 (politica actualizada el 2026-08-27)
- **Confirmado tambien por:** `https://www.airbnb.com/.well-known/security.txt` -> `Contact: https://hackerone.com/airbnb/`

## Dentro del alcance

Fuente: tabla de activos de HackerOne. Todos con severidad maxima *Critical* y recompensa *Eligible*.

### Higher Impact Scope

| Activo | Tipo | Informes resueltos | Notas |
|---|---|---|---|
| `www.airbnb.com` | Dominio | 389 (23%) | El activo con mas informes |
| `*.airbnb.com` | Wildcard | 361 (21%) | Cubre todos los subdominios del recon |
| `api.airbnb.com` | Dominio | 18 | |
| `assets.airbnb.com` | Dominio | 3 | Donde estan los PDF (paso exiftool) |
| `callbacks.airbnb.com` | Dominio | 0 | Visto en recon: `worldpay.callbacks.airbnb.com` |
| `support-api.airbnb.com` | Dominio | 2 | |
| `next.airbnb.com`, `one.airbnb.com`, `m.airbnb.com` | Dominio | 3 / 2 / 0 | |
| Sitios localizados | Otro | 22 | Lista en <https://www.airbnb.com/sitemaps/localized> |
| `*.musta.ch` | Wildcard | 2 | Dominio interno de Airbnb (anadido 2025-09-23) |
| `*.airbnbpayments.com` | Wildcard | 0 | Anadido 2025-09-23 |
| `*.airbnb.org` | Wildcard | 0 | Anadido 2025-08-09 |
| `com.airbnb.app` / `com.airbnb.android` | App iOS / Android | 14 / 70 | No aplican en recon pasivo de dominios |

### Lower Impact Scope

`*.withairbnb.com`, `*.muscache.com`, `*.luxuryretreats.com`, `*.hoteltonight.com` (y `www.hoteltonight.com`),
`*.hoteltonight-test.com`, `*.byairbnb.com`, `*.atairbnb.com`, `*.airbnbcitizen.com`, `*.airbnb-aws.com`,
`open.airbnb.com`, app iOS `com.luxuryretreats.ios`.

## Fuera del alcance

| Activo | Motivo |
|---|---|
| `demo.urbandoor.com`, `admin.demo.urbandoor.com`, `provider.demo.urbandoor.com` | Marcados *Ineligible* en HackerOne |
| `luckeyhomes.com`, `luckey.partners`, `luckey.in`, `luckey.fr`, `luckey.app` | Marcados *Ineligible* en HackerOne |
| `image.mail.airbnb.com` | Activo de terceros: CNAME a `akamai-san63.exacttarget.com` (Salesforce) |
| `airbnb.okta.com`, nodos compartidos de Akamai, Salesforce Marketing Cloud | Activos de terceros, no cubiertos por el programa |
| `airbnb.net`, `airbnb.biz`, `urbandoor.com` | No aparecen en la lista de activos |
| `*.airbnbopen.com` | Aparece en el texto de la politica pero no en la tabla de activos -> **discrepancia**, se trata como fuera |
| Dominios de pais (`airbnb.es`, `airbnb.com.ar`...) | Solo estarian en alcance si figuran en `sitemaps/localized`, pero ese enlace da 404 (comprobado 2026-10-28). No se puede confirmar -> fuera |

Tipos de vulnerabilidad excluidos (no son activos, pero conviene saberlos): DoS, phishing, ingenieria social,
divulgacion de version de software, paneles de login publicos, debilidades de CSP, falsificacion de correo,
claves de Google Maps expuestas, resultados de escaner sin validar, problemas sin impacto de seguridad claro.

## Notas sobre scope.txt

- Segun las reglas del `scope-guard`, `*.airbnb.com` **no** incluye `airbnb.com`, por eso el dominio raiz va en su propia linea.
- Las exclusiones `!` de urbandoor y luckey no son estrictamente necesarias (ningun comodin las cubre), pero dejan
  constancia de lo que el programa marca como no elegible.

## Reglas que afectan a este proyecto

| Regla del programa | Como afecta aqui |
|---|---|
| Prohibida la ingenieria social (phishing, vishing, smishing) | No se contacta con empleados (coincide con las reglas del proyecto) |
| No crear cuentas en masa | No se crean cuentas (el proyecto lo prohibe igualmente) |
| No hacer fuerza bruta para comprobar limites de velocidad | Nada de fuerza bruta; solo tecnicas pasivas |
| Evitar violaciones de privacidad, destruccion de datos o degradacion del servicio | Los datos personales se anotan anonimizados |
| Interactuar solo con cuentas propias | No aplica: no se usan cuentas del objetivo |
| Activos de terceros no cubiertos | Lo que resuelve a Akamai, Okta, Salesforce... se anota, no se investiga |
| Identificacion: cabecera `X-HackerOne-Investigador: <usuario>` e IP de origen en los informes | Solo relevante en fase activa (UD2); en recon pasivo no se envia trafico de pruebas |
| Herramientas automaticas / limite de peticiones | La politica **no** lo regula explicitamente; se aplica la regla general de no degradar el servicio |

Reglas propias del proyecto (enunciado): solo tecnicas pasivas, no interactuar con personas, no crear cuentas,
y si aparece algo grave, anotarlo sin el dato sensible y avisar al profesor.
