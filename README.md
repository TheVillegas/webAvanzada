# webAvanzada

Laboratorio evaluado — **Ingeniería Web Avanzada (OII436-1)**
Git, Angular, Integración Continua y CD básico con Terraform · Segundo semestre 2026

| Campo | Valor |
|---|---|
| Integrante 1 | _(completar)_ |
| Integrante 2 | _(completar)_ |
| Sección | _(completar)_ |
| Fecha | 07 de septiembre de 2026 |
| Rama obligatoria | `devops/ci-cd` |
| Repositorio | https://github.com/TheVillegas/webAvanzada |
| URL ejecución CI | _(completar tras el primer Pull Request)_ |

## Estructura

```
webAvanzada/
├── frontend/          # Aplicación Angular 20
│   ├── src/
│   ├── package.json
│   └── package-lock.json
├── README.md
└── .git/
```

---

## Parte I — Repositorio y frontend Angular

### Pregunta 1 (2 pts). ¿Por qué no se recomienda desarrollar directamente sobre `main`?

Porque `main` representa el estado **estable e integrable** del proyecto: es la rama desde la que se despliega y sobre la que otras personas basan su trabajo. Si se commitea directo sobre ella, cada cambio entra sin haber pasado por ninguna validación, y un error queda inmediatamente visible para todo el equipo y para el pipeline de entrega.

Trabajar en una rama de trabajo (`devops/ci-cd`) permite:

- **Aislar el trabajo en progreso**: los commits intermedios, incluso los que rompen, no afectan a nadie más.
- **Habilitar la revisión mediante Pull Request**: el código se discute antes de integrarse, no después.
- **Ejecutar CI como compuerta de calidad**: el workflow corre sobre el PR y bloquea la integración si las pruebas o el build fallan. Si commiteáramos sobre `main`, la validación llegaría tarde: el problema ya estaría integrado.
- **Poder revertir con bajo costo**: se descarta o corrige la rama sin reescribir la historia de `main`.

En resumen: `main` es el resultado de un trabajo ya validado, no el lugar donde se hace ese trabajo.

### Pregunta 2 (2 pts). ¿Qué problema se evita al utilizar `--skip-git` al crear el proyecto Angular?

Angular CLI, por defecto, inicializa un repositorio Git propio en la carpeta que crea. Como `frontend/` vive **dentro** de un repositorio ya inicializado (`webAvanzada`), eso generaría un `.git/` anidado: dos repositorios independientes, uno adentro del otro.

Las consecuencias serían:

- El repositorio raíz **no versionaría el contenido real** de `frontend/`. Git no entra en directorios con su propio `.git/`; en el mejor de los casos lo registraría como un submódulo/gitlink roto (una referencia a un commit que no existe en ningún remoto).
- Al clonar el repositorio en otra máquina —o en el runner de GitHub Actions— la carpeta `frontend/` llegaría **vacía**, y el pipeline de CI fallaría al no encontrar `package.json` ni el código fuente.
- La historia del frontend quedaría fuera del historial del proyecto, rompiendo la trazabilidad.

Con `--skip-git` se conserva **un solo repositorio** para todo el laboratorio, tal como exigen las reglas.

### Pregunta 3 (2 pts). ¿Qué verifica `npm run build` en esta etapa del laboratorio?

`npm run build` ejecuta `ng build`, que realiza la **compilación de producción** de la aplicación. Verifica que el proyecto sea construible de punta a punta, es decir:

- Que **compile TypeScript** sin errores de tipos.
- Que las **plantillas Angular compilen** (AOT): selectores, bindings e imports de componentes son válidos.
- Que **todas las dependencias se resuelvan** y el bundler pueda generar el grafo de módulos completo.
- Que se produzca el **artefacto desplegable** en `frontend/dist/frontend/browser/` (`index.html`, JS y CSS). Ese directorio es exactamente lo que después consumirá Terraform en la Parte IV para publicar a staging.

La distinción importante frente a `npm test`: las pruebas validan **comportamiento**; el build valida que exista **algo publicable**. Una aplicación puede tener sus pruebas verdes y aun así no compilar en modo producción. Por eso el pipeline ejecuta ambas etapas.

### Pregunta 4 (2 pts). ¿Qué utilidad tiene revisar `git status` o `git diff --cached` antes de realizar un commit?

Sirven para **confirmar qué se va a versionar antes de que sea permanente**, y responden preguntas distintas:

- `git status` responde **qué archivos** están preparados (staged), modificados sin preparar y sin rastrear. Es la forma de detectar que se está por incluir algo que no corresponde: `node_modules/`, `dist/`, un `.env` con credenciales, o un archivo temporal.
- `git diff --cached` responde **qué contenido exacto** quedó preparado, línea por línea. Es lo que realmente entrará al commit. Ahí aparecen los `console.log` olvidados, un token pegado por accidente o cambios de depuración que no debían viajar.

El valor de fondo es de **higiene y seguridad**: un secreto commiteado queda en la historia de Git aunque después se borre el archivo (ver Pregunta 12). Revisar antes cuesta segundos; limpiar la historia después es caro y ruidoso. Además, mirar el diff completo obliga a construir commits **atómicos y con un mensaje honesto**, en lugar de un `git add .` a ciegas.
