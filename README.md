# Proyecto 1 · Know Your Target

- **Alumno:** Sergio Estudillo Marabot
- **Modulo:** Hacking Etico · UD1 Eyes on Target
- **Objetivo elegido:** Airbnb - programa publico de bug bounty en HackerOne (<https://hackerone.com/airbnb>)
- **Por que Airbnb:** programa publico y activo desde 2015, con alcance amplio (`*.airbnb.com`, `*.musta.ch`, `*.airbnb.org`...)

## Contenido

| Fichero | Que contiene |
|---|---|
| [`alcance.md`](alcance.md) | Programa, fecha de consulta, que esta dentro y fuera del alcance y reglas que afectan al proyecto |
| [`scope.txt`](scope.txt) | El alcance en formato maquina (un objetivo por linea, `!` para exclusiones). Lo usa tambien el agente |
| [`cuaderno.md`](cuaderno.md) | Cuaderno de trabajo de la Parte 1: cada accion con cuando, objetivo, que hice, que salio y siguiente paso |
| [`mapa.md`](mapa.md) | Mapa de superficie de ataque: resumen, subdominios, infraestructura, correos y personas, puntos de interes |
| [`evidencias/`](evidencias/) | Salidas de los comandos del cuaderno: `subdominios.txt` (subfinder), `crtsh.txt`, `ips_clave.txt` y `wayback_api.txt` (codigos de reserva y claves sustituidos por `<REDACTED>`) |
| [`agente/`](agente/) | Parte 2: herramientas anadidas, `pruebas.md`, `instrucciones.md`, traza e informe del agente |
| [`comparativa.md`](comparativa.md) | Parte 3: comparacion entre el reconocimiento manual y el del agente |

## Metodo

Solo tecnicas pasivas, sin escaneos, sin fuerza bruta y sin contacto con personas. Los datos personales
encontrados se anotan anonimizados.
