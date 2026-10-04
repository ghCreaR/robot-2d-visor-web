# robot-2d-visor-web

Visor web de la [carrera de robots autónomos](https://github.com/ojgarciab/carrera-robots-autonomos): para ver la carrera y los sensores de los robots.

> **Estado:** en diseño. Todavía no hay código; el plan de implementación está en [`Plan.md`](Plan.md).

## Qué es

Son **solo ficheros estáticos** (HTML, CSS y JavaScript): se pueden servir desde cualquier servidor web. El navegador se conecta directamente a la [pasarela](https://github.com/ghCreaR/robot-2d-pasarela) por **WebSocket**, aunque esta esté en **otro origen** (otro dominio o puerto). La dirección de la pasarela es configurable.

Es de **solo lectura**: no puede controlar ningún robot. Se autentica con un **token de API de solo lectura** (`crt_ro_…`) que el usuario genera en la [interfaz de gestión](https://github.com/ghCreaR/robot-2d-interfaz-web).

```
  servidor web ──(HTML/CSS/JS)──► navegador ──WebSocket + REST──► pasarela
  (este repo)                     (visor)       (otro origen, CORS)
```

## Vistas

| Vista | Qué muestra | Quién | Uso típico |
|-------|-------------|-------|------------|
| **Usuario** | **Solo los valores de los sensores** de un robot, con su marca de tiempo, el intervalo entre muestras y la latencia estimada. No muestra la posición real ni a los demás robots. | Cualquier token de solo lectura. | Que el dueño depure su algoritmo viendo lo mismo que "ve" su robot, o que otras personas lo sigan. |
| **Administrador** | El circuito y **todos** los robots con su **posición y orientación exactas**, dibujados con sus capas SVG y con el **nombre de su usuario** encima. | Solo usuarios con rol de administrador. | Proyectar la carrera en pantallas grandes. |

Así un participante solo dispone de la información que dan los sensores de su robot, mientras que el administrador tiene la vista completa.

**Robot fuera del mundo.** Si el robot no está en el mundo (su dueño se desconectó hace más de 5 minutos o salió voluntariamente), el visor muestra **"El robot no está actualmente en el mundo"**. No da la conexión por perdida: cuando el robot vuelve a entrar, sigue mostrando sus sensores sin recargar.

## Configuración

| Variable | Por defecto | Descripción |
|----------|-------------|-------------|
| `PASARELA_URL` | `http://localhost:8080` | Dirección de la pasarela tal y como la ve el navegador. Al arrancar el contenedor se escribe en un `config.js` que carga el visor. |

El origen del visor debe estar en `CORS_ORIGENES` de la pasarela; en el despliegue local ya lo está por defecto (`http://localhost:8081`).

El contenedor sirve los ficheros en el puerto `8080` (publicado en el `8081` en el despliegue local). Fuera de Docker, basta con copiar los ficheros a cualquier servidor web y editar `config.js`.

## Documentación relacionada

- [Visor web](https://github.com/ojgarciab/carrera-robots-autonomos#visor-web) y [CORS y orígenes permitidos](https://github.com/ojgarciab/carrera-robots-autonomos#cors-y-orígenes-permitidos)
- [Capas SVG de los robots](https://github.com/ojgarciab/carrera-robots-autonomos/blob/main/robots/README.md#capas-svg)
- [Plan de implementación](Plan.md)

## Licencia

[GPL-3.0](LICENSE).
