# Plan de implementación · robot-2d-visor-web

Este plan detalla cómo construir el visor web descrito en el [README del repositorio común](https://github.com/ojgarciab/carrera-robots-autonomos). Todavía no hay código: es una propuesta para revisar antes de empezar.

La API y el protocolo WebSocket que usa el visor están definidos en [`contratos/api-cliente.md`](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/contratos/api-cliente.md) del repositorio común, y el formato de los circuitos en [`circuitos/README.md`](https://github.com/ojgarciab/carrera-robots-autonomos/blob/master/circuitos/README.md).

## 1. Decisiones técnicas propuestas

| Tema | Propuesta | Motivo |
|------|-----------|--------|
| Lenguaje | **JavaScript moderno con módulos ES**, sin compilación | El README común pide ficheros estáticos que se puedan servir desde cualquier servidor web. Sin paso de compilación, el repositorio es directamente lo que se publica. |
| Dibujo del circuito y de los robots | **SVG** en el DOM | Las capas de los robots ya son SVG con 1 unidad = 1 mm, así que se insertan tal cual con un `transform` por robot. Para 4 robots a 30 Hz el rendimiento es de sobra. Si algún día hay muchos robots, se puede pasar a Canvas. |
| Gráficas de sensores | SVG propio, sin librerías | Pocas series y pocos datos; evita dependencias externas. |
| Estilos | CSS propio, con tema claro y oscuro | |
| Servidor del contenedor | **nginx:alpine** | Sirve los estáticos y escribe `config.js` al arrancar a partir de `PASARELA_URL`. |
| Pruebas | **Vitest** con `jsdom` para la lógica, y **Playwright** para el extremo a extremo | Solo herramientas de desarrollo: no afectan a lo que se publica. |
| Calidad | **ESLint** y **Prettier** | |

## 2. Estructura del repositorio

```
robot-2d-visor-web/
├── Dockerfile
├── docker/
│   ├── nginx.conf
│   └── config.sh             # genera config.js desde PASARELA_URL al arrancar
├── public/                   # lo que se publica
│   ├── index.html            # acceso: token, mundo y vista
│   ├── usuario.html          # vista de usuario (sensores)
│   ├── admin.html            # vista de administrador (carrera)
│   ├── config.js             # { pasarelaUrl } (generado en el contenedor)
│   ├── css/
│   └── js/
│       ├── api.js            # REST: /yo, /mundos, /robots, /circuitos, /ping
│       ├── conexion.js       # WebSocket: autenticar, reconectar, despachar mensajes
│       ├── latencia.js       # ping periódico: rtt y desfase de reloj
│       ├── token.js          # guardar y leer el token (sessionStorage)
│       ├── robots.js         # definición de robot y capas SVG
│       ├── circuito.js       # dibujo del circuito
│       ├── panel-sensores.js # sensores de un robot: silueta, tabla, gráfica y métricas
│       ├── vista-usuario.js
│       └── vista-admin.js
├── tests/
└── package.json              # solo para herramientas de desarrollo
```

## 3. Comportamiento

### 3.1. Acceso

1. La persona pega su **token de solo lectura**. Se guarda en `sessionStorage`, que se borra al cerrar la pestaña, y **nunca en la URL**.
2. Con `GET /yo` se comprueba el token y se averigua si su dueño es administrador.
3. Con `GET /mundos` se elige el mundo; se marcan los inactivos. Se muestra **un mundo cada vez**: al cambiar de mundo se cancelan las suscripciones del anterior (`dejar`).
4. Se elige la vista: la de administrador solo aparece si el usuario lo es.

Si llega un token de lectura-escritura, el visor avisa de que **no conviene usarlo** aquí, porque el visor no lo necesita y quedaría expuesto en el navegador. Funcionaría igual, pero se recomienda el de solo lectura.

### 3.2. Conexión WebSocket

- Abre `/ws` y manda primero `{"tipo": "autenticar", "token": …}`, porque los navegadores no permiten la cabecera `Authorization` en un WebSocket.
- Después manda `seguir` (vista de usuario) u `observar` (vista de administrador).
- Si la conexión se corta, **reconecta sola** con espera exponencial (de 1 s a 30 s) y vuelve a suscribirse. Muestra el estado de la conexión de forma discreta.
- Si el token se rechaza (caducado o revocado), vuelve a la pantalla de acceso con un mensaje claro.

### 3.3. Vista de usuario

La muestra el componente `panel-sensores.js`, que también usa la vista de administrador para ver los sensores de cualquier robot.


- Carga la definición del robot (`GET /robots/<id>`) para saber qué sensores tiene y dónde están.
- **Dibujo de los sensores** con sus posiciones reales sobre la silueta del robot (capas SVG), con una intensidad proporcional a su valor. Con los sensores digitales actuales se ven encendidos (`1`) o apagados (`0`); con el sensor promediado previsto se verán en escala de grises sin cambiar el código. Así se ve lo mismo que "ve" el robot.
- **Tabla de valores** con el último dato de cada sensor.
- **Gráfica temporal** de los últimos segundos de cada sensor, como un cronograma.
- **Métricas:** marca de tiempo de la última muestra, intervalo entre muestras (unos 100 ms), muestras perdidas (huecos en `seq`), latencia (`rtt`) y retraso de llegada estimado con el desfase de reloj calculado con `ping`, como explica el README común.
- **Robot fuera del mundo:** al recibir `{"tipo": "robot", "en_mundo": false}` muestra "El robot no está actualmente en el mundo" y deja de actualizar, sin cerrar la conexión. Cuando llega `en_mundo: true`, vuelve a mostrar los sensores.

### 3.4. Vista de administrador

- Dibuja el circuito con `GET /circuitos/<id>`: rectas y arcos en un SVG con `viewBox` del tamaño del mapa (`dimensiones`), en milímetros, y la `y` invertida. Los arcos se pasan a comandos `A` de SVG teniendo en cuenta su sentido (antihorario si `fin > inicio`).
- Por cada robot del `estado`, un grupo `<g>` con sus capas SVG en orden (cargadas una vez y reutilizadas) y un `transform="translate(x, y) rotate(θ)"`. **La `y` cambia de signo**, igual que en el convenio de las capas, para que el dibujo coincida con el mundo.
- **Etiqueta con el nombre visible** del usuario sobre cada robot, sin girar con él.
- **Interpolación** entre dos estados (llegan a 30 Hz) para que el movimiento sea suave en pantallas a 60 Hz o más.
- **Modo pantalla completa** pensado para proyectar: sin controles visibles, con el nombre del mundo y el número de robots.
- **Sensores de cualquier robot:** al pulsar sobre un robot (o elegirlo en la lista de robots del mundo) se abre un panel lateral con sus sensores. Usa `seguir` con el `usuario` de ese robot y el mismo componente `panel-sensores.js` que la vista de usuario. Al elegir otro robot se cancela el anterior con `dejar`. El robot seleccionado se resalta en el circuito. En pantalla completa el panel se oculta.
- Los robots que entran o salen aparecen o desaparecen con una transición breve.

## 4. Despliegue

- `Dockerfile` basado en `nginx:alpine` que copia `public/` y ejecuta `config.sh` al arrancar. El script escribe `config.js` con `PASARELA_URL` (validada: solo `http://` o `https://`).
- nginx escucha en el `8080` sin privilegios, con caché larga para CSS y JS y sin caché para `config.js` y los HTML.
- **CSP** que solo permite conectar con el propio origen y con la pasarela (`connect-src`).
- Fuera de Docker: se copian los ficheros de `public/` y se edita `config.js`.

## 5. Fases de implementación

### Fase 0 · Esqueleto
- Estructura, ESLint, Prettier, Vitest y GitHub Actions.
- `Dockerfile` con nginx y generación de `config.js`.
- Página de acceso que solo muestra la `pasarelaUrl` configurada.

### Fase 1 · Acceso y API
- `token.js`, `api.js` (`/yo` y `/mundos`) y el flujo de la sección 3.1.
- Pruebas con un servidor falso (`fetch` simulado).

### Fase 2 · WebSocket
- `conexion.js` con autenticación, suscripción, reconexión y gestión de errores.
- `latencia.js` con `ping` periódico.
- Servidor WebSocket falso para las pruebas, que emite sensores y eventos de presencia.

### Fase 3 · Vista de usuario
- Sensores sobre la silueta, tabla, gráfica temporal, métricas y mensaje de robot fuera del mundo.

### Fase 4 · Vista de administrador
- Circuito, robots con capas SVG, etiquetas, interpolación y pantalla completa.
- Selección de un robot y panel con sus sensores (`seguir` con `usuario`).
- Prueba visual con Playwright contra el servidor falso (captura de pantalla de referencia).

### Fase 5 · Integración
- Prueba de extremo a extremo con el `compose.yaml` del repositorio común: visor en `localhost:8081` y pasarela en `localhost:8080` (orígenes distintos, para probar CORS).
- Accesibilidad básica: contraste, textos alternativos y uso con teclado.

## 6. Dependencias con otros repositorios

| Depende de | Qué necesita |
|------------|--------------|
| `robot-2d-pasarela` | `GET /yo`, `GET /mundos`, `GET /robots/…`, `GET /circuitos/…`, `GET /ping` y el protocolo WebSocket (`autenticar`, `seguir`, `observar` y los mensajes `sensores`, `robot` y `estado`). Hasta que exista, se trabaja con el servidor falso. |
| Repositorio común | Contrato de la API, capas SVG de los robots y formato de los circuitos (ya existen). |

## 7. Decisiones tomadas

- **Sensores IR:** digitales para empezar; el visor pinta la intensidad del valor, así que el sensor promediado futuro no necesita cambios.
- **Circuitos:** se obtienen de la pasarela (`GET /circuitos/<id>`), que los carga del repositorio común.
- **Administradores:** pueden ver los sensores de cualquier robot desde la vista de administrador.
- **Un mundo cada vez:** no hay vista de varios mundos en paralelo.

## 8. Preguntas abiertas

Ninguna por ahora.
