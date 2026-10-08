# Hallazgos de seguridad

Aquí guardo los informes de vulnerabilidades que voy encontrando en programas de bug
bounty y de divulgación de vulnerabilidades (VDP), en plataformas como Intigriti,
Bugcrowd y YesWeHack.

Todos los datos de las organizaciones afectadas están censurados. No aparece ningún
dominio, ni el nombre de la empresa u organismo, ni nada que permita saber a quién
afecta cada hallazgo. Lo que dejo es la parte técnica: qué fallo es, cómo se reproduce,
qué impacto tiene y cómo se arregla.

## Qué he censurado

- Dominios y subdominios, tanto de la organización como de terceros.
- Nombres de organizaciones, proveedores, aplicaciones y proyectos.
- Direcciones IP, correos, contraseñas, tokens, identificadores y hashes.
- Rutas internas de los servidores y nombres de paquetes propios de un proveedor.
- Los nombres de los ficheros, que también son genéricos por el mismo motivo.

De cada organización solo digo lo que es en general (una universidad, un portal, un
organismo público), sin más detalle.

## Cómo se ve la censura

Los datos quitados los sustituyo por etiquetas entre corchetes:

- `[TARGET]`, `[TARGET-1]`, `[TARGET-2]`: el host afectado en ese informe.
- `[TARGET-DOMAIN]`: el dominio principal de la organización.
- `[TARGET-PORTAL]`, `[TARGET-AUTH]`: otros hosts de la misma organización.
- `[THIRD-PARTY-DOMAIN]`: el dominio de un tercero.
- `[INTERNAL-DOMAIN]`: un dominio de red interna.
- `[ORG]`, `[APP]`, `[VENDOR]`, `[FRAMEWORK]`: nombres de organización, aplicación,
  proveedor o framework.
- `[IP]`, `[EMAIL]`, `[PASSWORD]`, `[TOKEN]`, `[ID]`, `[UUID]`: valores concretos.

Hay alguna más (`[REALM]`, `[ENV]`, `[POD-SUFFIX]`...) que se explica sola por el
contexto. Cada etiqueta vale solo dentro de su informe: un `[TARGET]` de un fichero no es
el mismo host que el `[TARGET]` de otro.

## Informes

- `01` y `02`: el hash de la contraseña del usuario se devuelve en la API de cuenta de
  Keycloak de un portal. Son mis notas y el informe final.
- `03`, `04` y `05`: un filtro de seguridad mal configurado en una API de registro
  devuelve trazas de pila de Java completas y datos de la infraestructura de Kubernetes.
  Son mis notas, la primera versión y el informe final.
- `06`: una API REST sin autenticación expone datos personales de usuarios de una
  aplicación universitaria.
- `07`: continuación del anterior. Se puede crear una cuenta de administrador sin
  autenticarse.
- `08`: Django con `DEBUG=True` en producción en dos sitios web de proyectos de
  investigación.
- `09`: la consola JSONWS y el esquema GraphQL de un Liferay de un organismo público
  se pueden usar sin autenticación.
- `10` y `11`: toma de control de subdominios (subdomain takeover) por registros CNAME
  que apuntan a un dominio de un tercero a punto de caducar. Son el informe y el
  contenido que preparé para rellenar el formulario del programa.

## Notas

- Varios ficheros son versiones del mismo hallazgo (notas, borrador e informe final). Los
  dejo todos para que se vea cómo fui afinando el informe.
- Los informes están en inglés porque es el idioma de las plataformas.
- Las capturas de pantalla que mencionan los informes no están incluidas.

## Licencia

GPL-3.0. Ver el fichero LICENSE.
