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
| Pull Request | https://github.com/TheVillegas/webAvanzada/pull/1 |
| URL ejecución CI | https://github.com/TheVillegas/webAvanzada/actions/runs/34125137657 |

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

---

## Parte II — Integración Continua con GitHub Actions

### Pregunta 5 (2 pts). ¿Qué evento activa el workflow `ci.yml`?

Lo activa el evento **`pull_request` con `branches: [main]`**: cada vez que se abre un Pull Request cuyo destino (base) es `main`, y también en cada nuevo push a la rama de origen mientras el PR siga abierto (los tipos por defecto son `opened`, `synchronize` y `reopened`).

Lo importante conceptualmente: la validación se dispara **antes de integrar**, sobre el resultado de fusionar la rama con `main`. Ese es el rol de CI — actuar como compuerta de calidad en el momento en que el cambio pide entrar, no después de que ya entró.

### Pregunta 6 (2 pts). En `runs-on: ubuntu-latest`, ¿qué representa `ubuntu-latest`?

Es la **etiqueta del runner**: la máquina virtual efímera y limpia que GitHub aprovisiona para ejecutar el job. `ubuntu-latest` apunta a la imagen Ubuntu LTS más reciente mantenida por GitHub, con un conjunto de herramientas preinstaladas (Git, Node, navegadores, Docker, etc.).

Dos propiedades que importan:

- **Es efímera y reproducible**: nace vacía para cada ejecución y se destruye al terminar. No hay estado heredado de corridas anteriores, así que el pipeline no puede "funcionar por casualidad" gracias a algo instalado a mano. Esto es lo que elimina el clásico *"en mi máquina funciona"*.
- **`latest` es un alias móvil**: hoy resuelve a una versión concreta de Ubuntu, y GitHub la va migrando con el tiempo. Es cómodo, pero implica que el entorno puede cambiar bajo tus pies. Para pipelines críticos se prefiere fijar la versión (`ubuntu-24.04`) y así garantizar reproducibilidad a largo plazo.

### Pregunta 7 (2 pts). Ordene las etapas del job `frontend` y explique por qué `npm ci` se ejecuta antes que las pruebas.

Orden de ejecución:

1. **Obtener código** (`actions/checkout@v4`) — clona el repositorio en el runner. Sin esto el runner está vacío.
2. **Configurar Node.js** (`actions/setup-node@v4`) — instala Node 20 y habilita la caché de npm usando `frontend/package-lock.json` como clave.
3. **Instalar dependencias** (`npm ci`) — materializa `node_modules/`.
4. **Ejecutar pruebas** (`npm test -- --watch=false`) — valida el comportamiento.
5. **Construir Angular** (`npm run build`) — genera el artefacto de producción.

`npm ci` va antes de las pruebas por una razón obvia y otra de fondo:

- **La obvia**: sin dependencias instaladas no existen ni Angular, ni Karma, ni Jasmine. El comando de pruebas directamente no podría ejecutarse.
- **La de fondo**: `npm ci` instala **exactamente** las versiones fijadas en `package-lock.json`, borrando cualquier `node_modules/` previo. A diferencia de `npm install`, no resuelve rangos ni modifica el lockfile. Eso hace que la validación sea **determinista**: si el CI pasa hoy y falla mañana, el cambio está en tu código, no en una dependencia transitiva que se actualizó sola. Probar contra dependencias no fijadas es probar contra un blanco móvil.

El orden general del pipeline también sigue un criterio de costo: cada etapa es más cara que la anterior, y las baratas fallan primero.

### Pregunta 8 (3 pts). Después del push, ¿qué etapa del pipeline falla y qué ocurre con las etapas siguientes?

Falla la etapa **"Ejecutar pruebas"** (`npm test -- --watch=false`). Jasmine reporta que el `<h1>` contiene `Catálogo de Recursos` pero la aserción esperaba `Título incorrecto`; el proceso termina con **exit code 1**.

Las etapas siguientes **no se ejecutan**. En la vista de Actions, "Construir Angular" queda marcada como omitida (`-`), no como fallida:

```
✓ Instalar dependencias
X Ejecutar pruebas
- Construir Angular      ← nunca se ejecutó
```

Esto ocurre porque los `steps` de un job son **secuenciales y con cortocircuito**: si un step devuelve un código de salida distinto de cero, el job se marca como fallido y se saltan los steps restantes (salvo que se declare explícitamente `if: always()` o `continue-on-error`). El job pasa a estado `failure`, el check del Pull Request queda en rojo y GitHub marca el PR como no apto para integrar.

Es un comportamiento **deseado**, no una limitación: si el código no supera las pruebas, construir el artefacto sería desperdiciar tiempo de runner sobre algo que ya sabemos que está mal. El pipeline falla rápido y barato.

**Nota sobre un fallo adicional encontrado durante el laboratorio**: la primera ejecución del workflow falló antes del fallo controlado, con `Missing X server or $DISPLAY / Chrome failed 2 times (cannot start)`. Angular 20 intenta abrir Chrome en modo ventana y el runner no tiene entorno gráfico. Se corrigió declarando `"browsers": "ChromeHeadless"` en las opciones del target `test` de `frontend/angular.json`, lo que además vuelve reproducible la ejecución local. Es un caso ilustrativo de la Pregunta 6: el runner es un entorno distinto al de desarrollo, y el pipeline expone esas diferencias.

### Pregunta 9 (3 pts). ¿Debería integrarse este Pull Request a `main` mientras el pipeline está fallando? Justifique.

**No.** Y no por formalismo burocrático, sino por lo que significa `main`.

- `main` es la rama desde la que se despliega. En este mismo laboratorio, `cd.yml` se dispara con `push` sobre `main`: integrar código roto no lo deja "roto y quieto", lo **despliega automáticamente a staging**. El error se propaga solo.
- Un pipeline en rojo es información concreta: el software **no cumple su contrato verificable**. Ignorarla convierte al CI en decoración. Un check que se saltea cuando molesta deja de ser una compuerta de calidad y pasa a ser ruido — y el equipo aprende a no mirarlo.
- El costo de arreglar sube con el tiempo. En la rama, el fallo afecta a una persona y se corrige con un commit. En `main`, bloquea a todo el equipo: cada quien que ramifique parte de una base rota y no sabrá si el error es suyo o heredado.
- Se pierde la trazabilidad: cuando `main` está siempre verde, cualquier fallo nuevo apunta al último cambio. Cuando se tolera el rojo, ya no se puede distinguir el fallo nuevo del preexistente.

El procedimiento correcto es el que se aplicó: corregir en la rama `devops/ci-cd`, pushear, esperar el check en verde y **recién entonces** integrar. Por eso conviene además proteger `main` con *branch protection* que exija el check aprobado — así la regla no depende de la disciplina de nadie.
