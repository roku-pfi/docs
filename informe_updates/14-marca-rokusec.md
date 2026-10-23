# Marca RokuSec: fundamento y guía breve (B-5; ADR-0030, 0036, 0038)

Nota para la sección de diseño / identidad del informe. No es texto final:
son los argumentos y los datos, para redactar.

## 1. Por qué una marca

El prototipo no tenía identidad: un índigo genérico y un escudo. Además, el
banco de demostración usaba la paleta del IdP, y eso hacía que una aplicación
cliente pareciera parte del proveedor de identidad. Para una demo de producto
hay que distinguir tres cosas a simple vista: **el producto** (RokuSec), **sus
clientes** (Roku Bank, Kaze Labs) y **quién mira** (el usuario final o el
operador). La marca resuelve las tres (ADR-0036).

## 2. El nombre y el símbolo

- **RokuSec**: *roku* (六) es "seis" en japonés; *Sec*, seguridad.
- **El símbolo**: un candado cuyo arco abierto y cuerpo redondo dibujan el
  número **6**.
- **Abierto a propósito**: es la promesa del producto. Un inicio de sesión
  habitual se abre sin pasos extra; el candado solo se cierra cuando el
  contexto es inusual. Es la autenticación basada en riesgo dicha en una
  imagen.
- **La cerradura en el centro**: la decisión se toma en el punto de entrada, al
  iniciar sesión.
- **Wordmark**: Sora 700 (licencia SIL OFL 1.1), convertida a contornos. "Roku"
  en azul marino, "Sec" en azul de marca: producto y propósito.
- **Construcción**: el vector se redibujó midiendo el boceto original y se
  genera con un script (`design/brand/build.py` → `docs/brand/`). Es
  reproducible: no se edita a mano.

## 3. Paleta

Muestreada del logo y nombrada:

| Token | Hex | Uso |
|---|---|---|
| navy | `#152953` | "Roku", cerradura, lado en sombra |
| blue-700 | `#1150B0` | cuerpo del anillo |
| blue-600 | `#1466E5` | **acento en superficies claras** (contraste 5,2:1 sobre blanco) |
| blue-500 | `#1481F9` | "Sec" |
| sky-400 | `#17A2FF` | brillo del arco |
| (inverso) | `#4C9BFF` | **acento en la consola oscura** (6,7:1 sobre `#0D1017`) |

**Escala de severidad, que nunca usa rojo** (ADR-0030):

| Acción | Color |
|---|---|
| ALLOW | verde |
| REQUIRE_MFA | cian |
| REAUTHENTICATE | índigo |
| BLOCK | ciruela |

- Se distingue también por luminosidad, así que se ordena bien en escala de
  grises y con daltonismo rojo-verde.
- Una contraseña incorrecta no es un nivel de severidad y no toma esos colores.

## 4. Quién mira decide el tema

| Superficie | Tema | Por qué |
|---|---|---|
| Login alojado | claro | un cliente, pocas veces por semana, a menudo en el celular; una página clara inspira más confianza al pedir una contraseña |
| Consola | oscuro | un operador durante una hora siguiendo un incidente; un fondo oscuro cansa menos y deja que los colores de severidad pesen |

No hay selector de tema: la elección es por superficie, para que ningún
público pague por las necesidades del otro.

## 5. Los clientes tienen identidad propia

**Roku Bank** (banca de consumo, B2C):
- verde pino `#0B6E4F` con lima `#C6F36B`;
- tipografía Plus Jakarta Sans;
- símbolo de moneda y ranura.

**Kaze Labs** (portal interno de una empresa de software):
- blanco cálido, tinta `#1C1917` y coral `#F2542D`;
- tipografía Manrope;
- símbolo de tres trazos de viento (*kaze*, "viento").

Ninguna de las dos se parece a RokuSec ni a la otra. Así la demo muestra lo que
pasa en la realidad: RokuSec protege aplicaciones que no son suyas.

## 6. El login, según el cliente

El login siempre lo sirve el IdP: la contraseña se escribe en el origen de
RokuSec, nunca en el de la aplicación. Cada organización elige cómo se ve:

| Estilo | Aspecto | Quién lo usa |
|---|---|---|
| `tenant` | logo, colores y textos del banco, con un sello "Protegido por RokuSec" | Roku Bank (como un banco con su propio login) |
| `rokusec` | el login de RokuSec, "Continuá a Kaze Labs" | Kaze Labs (como un IdP corporativo) |

Las páginas de verificación y de bloqueo siguen la misma marca. No revelan el
motivo (ADR-0023): el usuario nunca ve un puntaje.

## 7. La consola sobre IBM Carbon (ADR-0038)

**Por qué Carbon y no una interfaz propia:** una primera versión hecha a mano se
veía "genérica de IA". Carbon (tema g100) aporta un lenguaje de producto
empresarial probado, accesible y consistente. RokuSec se pone encima:
- los azules del logo reemplazan los de Carbon;
- IBM Plex Sans/Mono para texto y datos;
- Sora solo en el logo.

**Criterios de presentación:**
- celdas de dos líneas (qué es y qué lo distingue);
- tiempo relativo con la hora exacta debajo;
- acciones destructivas siempre confirmadas;
- los secretos se muestran una sola vez en un modal con botón de copiar.

## 8. Idioma y tono

- Todo lo que una persona lee está en español rioplatense (es-AR): login,
  consola y aplicaciones. Fechas y números van en formato es-AR.
- Los logs, las métricas, los códigos de motivo y la API quedan en inglés. La
  consola traduce los códigos y muestra el original al pasar el mouse.
- **Tono escueto**: en pantalla, cifras, etiquetas y etiquetas de color. Toda
  oración que explique algo va en un tooltip (ⓘ junto al título o un término
  subrayado con puntos).

## 9. Guía breve de uso

**Logo**
- Principal (símbolo sobre wordmark): portada del informe y diapositivas de
  título.
- Horizontal: barras de navegación sobre fondo claro.
- Horizontal inverso: sobre fondo oscuro (consola).
- Mono: impresión y sellos.

**Tamaño mínimo del símbolo:** 32 px. A 16 px la cerradura se empasta; si hace
falta un ícono de 16 px, se dibuja uno propio en lugar de escalar.

**No hacer**
- recolorear el símbolo fuera de las variantes;
- cerrar el candado;
- usar rojo o naranja para estados de error o riesgo;
- usar la paleta de RokuSec en las aplicaciones cliente.

**Archivos:** `docs/brand/`, con README, SVG, PNG, favicon e íconos de 180 y
512 px.

## Para revisar en el informe

- La sección de diseño / interfaz: reemplazar las capturas anteriores a la
  reconstrucción por las de la consola Carbon, el login en sus dos estilos y
  las dos aplicaciones.
- Mencionar la escala de severidad sin rojo como decisión de accesibilidad.
- Si el informe tiene un apartado de trabajo futuro: dominios propios por
  organización. Hoy el login con la marca del banco corre en el dominio de
  RokuSec (`*.cloudfront.net`), no en el del banco.
