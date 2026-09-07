# webAvanzada

Laboratorio evaluado — **Ingeniería Web Avanzada (OII436-1)**
Git, Angular, Integración Continua y CD básico con Terraform · Segundo semestre 2026

| Campo            | Valor                                                               |
| ---------------- | ------------------------------------------------------------------- |
| Integrante 1     | Matias Eliseo Villegas Muñoz                                        |
| Integrante 2     | _(completar)_                                                       |
| Sección          | OII436-1                                                            |
| Fecha            | 07 de septiembre de 2026                                            |
| Rama obligatoria | `devops/ci-cd`                                                      |
| Repositorio      | https://github.com/TheVillegas/webAvanzada                          |
| Pull Request     | https://github.com/TheVillegas/webAvanzada/pull/1                   |
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

## Parte I: Repositorio y frontend Angular

### Pregunta 1 (2 pts). ¿Por qué no se recomienda desarrollar directamente sobre `main`?

Porque `main` representa el estado **estable e integrable** del proyecto: es la rama desde la que se despliega y sobre la que otras personas basan su trabajo. Si se commitea directo sobre ella, cada cambio entra sin haber pasado por ninguna validación, y un error queda inmediatamente visible para todo el equipo y para el pipeline de entrega.

Trabajar en una rama de trabajo (`devops/ci-cd`) permite:

- **Aislar el trabajo en progreso**: los commits intermedios, incluso los que rompen, no afectan a nadie más.
- **Habilitar la revisión mediante Pull Request**: el código se discute antes de integrarse, no después.
- **Ejecutar CI como compuerta de calidad**: el workflow corre sobre el PR y bloquea la integración si las pruebas o el build fallan. Si commiteáramos sobre `main`, la validación llegaría tarde: el problema ya estaría integrado.
- **Poder revertir con bajo costo**: se descarta o corrige la rama sin reescribir la historia de `main`.
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

### Pregunta 9 (3 pts). ¿Debería integrarse este Pull Request a `main` mientras el pipeline está fallando? Justifique.

**No.** debido a lo que significa `main`.

- `main` es la rama desde la que se despliega. En este mismo laboratorio, `cd.yml` se dispara con `push` sobre `main`: integrar código roto no lo deja "roto y quieto", lo **despliega automáticamente a staging**. El error se propaga solo.
- Un pipeline en rojo es información concreta: el software **no cumple su contrato verificable**. Ignorarla convierte al CI en decoración. Un check que se saltea cuando molesta deja de ser una compuerta de calidad y pasa a ser ruido y el equipo aprende a no mirarlo.
- El costo de arreglar sube con el tiempo. En la rama, el fallo afecta a una persona y se corrige con un commit. En `main`, bloquea a todo el equipo: cada quien que ramifique parte de una base rota y no sabrá si el error es suyo o heredado.
- Se pierde la trazabilidad: cuando `main` está siempre verde, cualquier fallo nuevo apunta al último cambio. Cuando se tolera el rojo, ya no se puede distinguir el fallo nuevo del preexistente.

El procedimiento correcto es el que se aplicó: corregir en la rama `devops/ci-cd`, pushear, esperar el check en verde y **recién entonces** integrar. Por eso conviene además proteger `main` con *branch protection* que exija el check aprobado , donde así la regla no depende de la disciplina de nadie.

---

## Parte III Gestión segura de secretos y configuración

### Pregunta 10 (4 pts). Clasifique cada elemento

| Elemento | Clasificación | Justificación |
|---|---|---|
| `package.json` | **Versionable** | Define dependencias y scripts del proyecto. Es código: debe estar en el repositorio para que cualquiera —incluido el runner de CI— pueda reconstruir el mismo entorno. No contiene información sensible. |
| API_URL pública | **Variable / configuración** | Cambia según el ambiente (desarrollo, staging, producción) pero no es secreta: viaja en cada request y es visible en el navegador. Va en una GitHub Variable o en un `.env.example`, nunca hardcodeada. |
| `AWS_REGION` | **Variable / configuración** | Identifica *dónde* se despliega, no *quién* tiene permiso. Conocer la región no otorga ningún acceso. Es configuración por ambiente. |
| `DB_PASSWORD` | **Secreto / no versionable** | Credencial de acceso directo a la base de datos. Su exposición compromete todos los datos. Va en un GitHub Secret o un gestor de secretos, jamás en el repositorio. |
| `API_TOKEN` | **Secreto / no versionable** | Credencial que autentica y autoriza llamadas en nombre del sistema. Quien lo obtiene puede suplantar a la aplicación. Mismo tratamiento que una contraseña. |
| `terraform.tfstate` | **Secreto / no versionable** | Es el registro del estado real de la infraestructura y almacena en **texto plano** los valores gestionados, incluidos atributos marcados como sensibles (contraseñas generadas, claves, endpoints privados). Además provoca conflictos de merge y corrupción de estado si dos personas lo versionan. Su lugar correcto es un *remote backend* con bloqueo (S3 + DynamoDB, Terraform Cloud, etc.). |

**El criterio que ordena la tabla**: la pregunta no es "¿es importante?" sino **"¿qué pasa si un desconocido lo lee?"**. Si la respuesta es "nada", es versionable o configuración. Si la respuesta es "obtiene acceso", es un secreto.

Y una distinción que se confunde seguido: **configuración ≠ secreto**. Ambos cambian por ambiente y ambos salen del código, pero por motivos distintos. La configuración sale para hacer el binario portable; el secreto sale para no filtrar acceso. Por eso GitHub los separa en dos mecanismos: `vars` (legibles, auditables) y `secrets` (write-only, enmascarados).

### Pregunta 11 (2 pts). ¿Por qué una contraseña o token no debe escribirse directamente dentro de `ci.yml`, `cd.yml` o un archivo TypeScript del frontend?

**1. Un secreto en el repositorio es un secreto público.** El valor queda en la historia de Git y se replica en cada clon, fork, mirror y backup. En un repositorio público lo indexan bots en minutos debido a que existen crawlers dedicados a escanear repositorios en GitHub buscando credenciales. Y aunque el repositorio sea privado, cualquier persona con acceso de lectura, presente o futura, lo obtiene, donde ademas se pierde el principio de menor privilegio.

**2. Se pierde la rotación y la trazabilidad.** Un secreto gestionado se cambia en un solo lugar y sigue funcionando. Uno hardcodeado obliga a un commit, un PR y un despliegue por cada rotación. así que en la práctica **no se rota nunca**. Además no hay registro de quién lo usó ni cuándo.

**3. En el frontend es directamente inútil como protección.** Este punto es el más importante y el que más se subestima: TypeScript **se compila y se entrega al navegador**. Cualquier valor puesto ahí termina dentro del bundle JavaScript que se descarga el usuario, y se lee abriendo DevTools. No existe forma de ocultar un secreto en código de cliente, no es una mala práctica, es una imposibilidad técnica. Si una operación necesita una credencial, esa operación pertenece al backend.

Por eso el workflow usa `${{ secrets.DEMO_TOKEN }}`: GitHub inyecta el valor como variable de entorno solo durante la ejecución del job, y además **enmascara** cualquier aparición del valor en los logs (lo reemplaza por `***`).

### Pregunta 12 (2 pts). Si un secreto real fue incluido en un commit y luego se agrega su archivo a `.gitignore`, ¿queda solucionado el problema?

**No queda solucionado en absoluto.**

`.gitignore` solo evita que Git empiece a rastrear archivos **no rastreados**. No tiene ningún efecto retroactivo:

- El archivo **ya está en la historia**. El blob con el secreto sigue accesible con `git log`, `git show <commit>:<archivo>` o navegando el commit en GitHub.
- Si el archivo ya estaba siendo rastreado, `.gitignore` **ni siquiera lo deja de rastrear**: hay que ejecutar además `git rm --cached <archivo>`.
- Cada clon, fork y backup existente **ya tiene una copia** del secreto. Aunque limpies el repositorio remoto, no controlás esas copias.

**La acción adicional imprescindible es ROTAR EL SECRETO**, y es lo primero que hay que hacer. Desde el instante en que se publicó, hay que considerarlo comprometido: revocar la credencial en el proveedor y emitir una nueva.

Procedimiento completo, en orden de prioridad:

1. **Rotar / revocar la credencial expuesta.** Urgente e inegociable. Todo lo demás es secundario.
2. **Guardar el nuevo valor donde corresponde**: GitHub Secret, gestor de secretos o `.env` local ignorado.
3. **Dejar de rastrear el archivo**: `git rm --cached .env` y commitear, con la regla ya presente en `.gitignore`.
4. **Purgar la historia** si el repositorio es público o el riesgo lo amerita, con `git filter-repo` o BFG Repo-Cleaner. Ojo: esto **reescribe la historia**, cambia todos los hashes posteriores y obliga a un `push --force` coordinado con el equipo.
5. **Auditar el uso** de la credencial expuesta por si hubo accesos indebidos, y **prevenir la reincidencia** con herramientas como `gitleaks` o `git-secrets` en un pre-commit hook.

---

## Parte IV Terraform y CD básico a staging

### Pregunta 13 (2 pts). ¿Qué diferencia existe entre `terraform validate`, `terraform plan` y `terraform apply`?

Son tres niveles de verificación, cada uno más profundo y más costoso que el anterior:

| Comando | Qué hace | Consulta el estado real | Modifica infraestructura |
|---|---|---|---|
| `terraform validate` | Verifica sintaxis HCL, tipos, referencias a variables y coherencia interna de la configuración. | No | No |
| `terraform plan` | Compara la configuración deseada contra el estado actual y calcula el diff: qué se crea, modifica o destruye. | Sí | No |
| `terraform apply` | Ejecuta ese diff contra los proveedores. Es la única etapa con efectos reales. | Sí | **Sí** |

En detalle:

- **`validate`** es un chequeo **offline**. No necesita credenciales ni acceso a red. Responde: *"¿esta configuración está bien escrita?"*. Detecta un `var.enviroment` mal tipeado o un tipo incompatible, pero no sabe nada del mundo real. Es el equivalente a que compile el código.
- **`plan`** es el **dry-run**. Lee el estado (`terraform.tfstate`), consulta la infraestructura existente y produce el plan de ejecución. Responde: *"¿qué pasaría si aplico esto?"*. En este laboratorio devolvió `Plan: 1 to add, 0 to change, 0 to destroy`. Es la etapa donde se revisa y donde se detecta un `destroy` inesperado **antes** de que ocurra.
- **`apply`** **ejecuta**. Crea, modifica o destruye recursos y actualiza el estado. Es irreversible sin un `destroy` o un rollback explícito. Por eso `-auto-approve` (que salta la confirmación interactiva) solo es aceptable en un pipeline donde el plan ya fue revisado o el alcance está acotado, como acá.
### Pregunta 14 (2 pts). ¿Por qué `ci.yml` se activa con `pull_request` y `cd.yml` se activa con `push` sobre `main`?

Porque **responden preguntas distintas, en momentos distintos del ciclo de vida del cambio**.

**`ci.yml` —>`pull_request`: ¿este cambio es apto para entrar?**
El Pull Request es el instante exacto en que un cambio *pide permiso* para integrarse. Validar ahí permite rechazarlo antes de que contamine la rama estable: el costo de arreglar es mínimo y el impacto queda contenido en la rama. Validar después de integrar sería llegar tarde — el problema ya estaría adentro.

**`cd.yml` — >`push` sobre `main`: este cambio ya fue aprobado, hay que entregarlo.**
Un push a `main` solo puede ocurrir tras la integración de un PR aprobado y con CI en verde. O sea, es la **señal de que existe una versión validada y lista para publicar**. Disparar el despliegue ahí garantiza que solo se entrega código que ya pasó la compuerta de calidad.

Lo que sostiene el diseño es la relación entre ambos:

- Si el CD se disparara con `pull_request`, se desplegaría código **no aprobado**, incluso de forks, y cada rama en progreso pisaría el ambiente de staging. El ambiente dejaría de reflejar un estado confiable.
- Si el CI se disparara solo con `push` a `main`, la validación llegaría **después** de la integración: `main` podría romperse y recién ahí nos enteraríamos.

Cada evento marca una transición: `pull_request` = *"quiero entrar"* → se valida. `push` a `main` = *"ya entré"* → se entrega. **CI protege la rama; CD publica desde la rama protegida.**

### Pregunta 15 (2 pts). ¿Qué función cumple Terraform dentro de este flujo de CD?

Terraform es la capa de **Infraestructura como Código (IaC)**: define de forma declarativa y versionada *cómo* queda preparado el ambiente de destino, y ejecuta esa preparación de manera reproducible.

En este laboratorio, concretamente:

- Declara el ambiente mediante la variable `environment`, que el pipeline inyecta desde una GitHub Variable a través de `TF_VAR_environment: ${{ vars.APP_ENV }}`. La convención `TF_VAR_<nombre>` es el puente entre las variables de entorno del runner y las variables de Terraform.
- Materializa la entrega con `terraform_data` y un provisioner `local-exec`, que copia el artefacto compilado (`frontend/dist/frontend/browser/`) al directorio `staging/`.
- Expone el resultado con el output `deployment_path`, dejando explícito dónde quedó el despliegue.

Lo importante son las **propiedades** que aporta, no el `cp`:

- **Declarativo**: se describe el estado deseado, no la secuencia de pasos. Terraform calcula el diff.
- **Idempotente y con estado**: gracias a `triggers_replace`, si nada cambió no vuelve a ejecutar. Ejecutarlo diez veces deja el mismo resultado que ejecutarlo una.
- **Versionado y auditable**: la definición del ambiente vive en Git, con historia y revisión por PR. La infraestructura deja de ser un conjunto de pasos manuales en la cabeza de alguien.
- **Portable**: el mismo flujo —`init` → `validate` → `plan` → `apply`— escala a S3, CloudFront, Kubernetes o cualquier proveedor real cambiando solo los recursos. El pipeline no cambia.
### Pregunta 16 (2 pts). ¿Por qué el workflow usa `${{ secrets.DEMO_TOKEN }}` en lugar de escribir el valor directamente?

Porque `secrets` es un mecanismo con garantías que un literal en el YAML no tiene:

1. **El valor nunca entra al repositorio.** El `cd.yml` es un archivo versionado y público. Escribir el token ahí lo expondría en la historia de Git, en cada clon y en cada fork — permanentemente (ver Pregunta 12).
2. **GitHub enmascara el valor en los logs.** Si el secreto aparece en la salida de un step, se reemplaza por `***`. Con un literal no hay tal protección: cualquiera con acceso a los logs de Actions lo lee.
3. **Se inyecta solo en tiempo de ejecución y con alcance acotado.** En el workflow se expone como variable de entorno **únicamente en el step que lo necesita**:

   ```yaml
   - name: Verificar secreto configurado
     env:
       DEMO_TOKEN: ${{ secrets.DEMO_TOKEN }}
     run: test -n "$DEMO_TOKEN"
   ```

   Los demás steps no lo ven. Eso es **principio de menor privilegio** aplicado al pipeline.
4. **Permite rotar sin tocar el código.** Se actualiza en `Settings → Secrets and variables → Actions` y todos los workflows toman el nuevo valor en la siguiente ejecución. Cero commits, cero despliegues.
5. **Es write-only y auditable.** Una vez guardado, ni el dueño del repositorio puede volver a leerlo desde la interfaz — solo reemplazarlo. GitHub además restringe su exposición: por defecto no se entrega a workflows disparados desde forks.

Nótese el contraste deliberado dentro del mismo `cd.yml`: `${{ vars.APP_ENV }}` para configuración **legible** (`staging`) y `${{ secrets.DEMO_TOKEN }}` para un valor **sensible**. Mismo mecanismo de inyección, distinto nivel de protección — porque son dos categorías distintas de dato.

Y el step `test -n "$DEMO_TOKEN"` cumple un rol de **fail-fast**: verifica que el secreto esté configurado y no vacío *antes* de llegar a `terraform apply`. Si falta, el pipeline corta temprano con un error claro, en lugar de fallar de forma confusa en una etapa posterior.
