## Objetivo
Explorar cómo crear una acción personalizada en JavaScript en GitHub Actions.

Crearemos una acción personalizada en JavaScript que compruebe si hay actualizaciones en las dependencias de npm de un proyecto, y abstraer esa lógica detrás de una acción reutilizable fácil de usar.

## Tareas

1. Crear la carpeta .github/actions/js-dependency-update. Esta carpeta es donde alojaremos todos los archivos de nuestra acción personalizada en JavaScript.
2. Abrir una terminal y cambiar a este directorio.
3. Inicializar un proyecto npm ejecutando el comando:
```bash
npm init -y
```
4. Instalar las dependencias de github actions necesarias ejecutando:
```bash
npm install @actions/core@1.10.1 @actions/exec@1.1.1 @actions/github@6.0.0 --save-exact
```
5. Crear un archivo index.js en la carpeta .github/actions/js-dependency-update y añadir el siguiente código al archivo:
    ```javascript
    const core = require('@actions/core');
    async function run() {
        // Escritura en la salida de la acción
      core.info('I am a custom JS action');
    }
    // Ejecuta la función run
    run();
    ```
   El código anterior aprovecha el paquete @actions/core para escribir una línea en la salida de nuestra acción personalizada.
6. Crear un archivo llamado action.yaml en la carpeta .github/actions/js-dependency-update.
7. En el archivo action.yaml, añadir las siguientes propiedades:
   - Un 'name' de Update NPM Dependencies;
   - Un 'description' de "Checks if there are updates to NPM packages, and creates a PR with the updated package*.json files";
   - Añadir una key 'runs' a nivel de workflow. Esta es la clave principal para definir nuestra acción personalizada en JavaScript. Para una acción personalizada en JavaScript, la key 'runs' tiene la siguiente forma:
```yaml
runs:
  using: node20
  main: index.js
```
   donde:
        - 'using: <Node version>' define con qué versión de NodeJS se ejecutará la acción.
        - 'main: <JavaScript file>' define **qué archivo se ejecutará como punto de entrada** de nuestra acción personalizada en JavaScript.

8. Modificar el archivo .gitignore en la raíz del repositorio para que la carpeta node_modules bajo la carpeta js-dependency-update no esté ignorada. Debería ser subida al repositorio si queremos que nuestra acción funcione correctamente. Una forma de hacerlo es añadir la siguiente línea al archivo: !.github/actions/**/node_modules. Esto asegurará que todas las carpetas node_modules bajo cualquier subdirectorio de la carpeta .github/actions sean incluidas en los commits de github, mientras que aún se ignoran los directorios node_modules de otras carpetas.
10. Confirmar los cambios y hacer push del código. Desencadenar el flujo de trabajo desde la interfaz de usuario y tómate unos momentos para inspeccionar la salida de la ejecución del flujo de trabajo.

11. Modificar el archivo action.yaml bajo la carpeta .github/actions/js-dependency-update añadiendo varios inputs necesarios:
   - **base-branch**, que debería:
      - Tener una descripción de 'The branch used as the base for the dependency update checks.'
      - Tener un valor por defecto de main.
      - No ser requerido.
   - **target-branch**, que debería:
      - Tener una descripción de 'The branch from which the PR is created.'
      - Tener un valor por defecto de update-dependencies.
      - No ser requerido.
   - **working-directory**, que debería:
      - Tener una descripción de 'The working directory of the project to check for dependency updates.'
      - Ser requerido.
   - **gh-token**, que debería:
      - Tener una descripción de 'Authentication token with repository access. Must have write access to contents and pull-requests.'
      - ser requerido.
   - **debug**, que debería:
      - Tener una descripción de 'Whether the output debug messages to the console.'
      - No ser requerido.

12. Aprovecha el paquete @actions/exec para ejecutar scripts de shell. Para ello, usa el método exec del paquete mencionado, o el método getExecOutput cuando necesites acceso al stdout y stderr del comando:
      - Ejecuta el comando npm update dentro del directorio de trabajo proporcionado (consulta la documentación del método exec para ver cómo proporcionar el directorio de trabajo para el comando). 
        Por ejemplo, puedes usar el siguiente código para ejecutar el comando npm update:
        ```javascript
        const exec = require('@actions/exec');
        await exec.exec('npm', ['update'], { cwd: workingDirectory });
        ```
      - Ejecuta el comando git status -s package*.json para comprobar si hay actualizaciones en los archivos package*.json. Usa el método getExecOutput y almacena el valor de retorno del método en una variable para su uso posterior.
        Por ejemplo, puedes usar el siguiente código para ejecutar el comando git status:
        ```javascript
        
        let gitStatusOutput = await exec.getExecOutput('git', ['status', '-s', 'package*.json'], { cwd: workingDirectory });
        ```
      - Si el stdout del comando git status tiene algún carácter, imprime un mensaje diciendo que hay actualizaciones disponibles. De lo contrario, imprime un mensaje diciendo que no hay actualizaciones en este momento.
        Por ejemplo, puedes usar el siguiente código para comprobar si hay actualizaciones disponibles:
        ```javascript
            if (gitStatusOutput.stdout) {
                core.info('There are updates available.');
            } else {
                core.info('There are no updates at this point in time.');
            }
        ```
13. Confirmar los cambios y hacer push del código. Desencadenar el flujo de trabajo desde la interfaz de usuario, y tómate unos momentos para inspeccionar la salida de la ejecución del flujo de trabajo. ¿Cómo manejó la acción diferentes inputs?    

14. Actualiza el archivo index.js para ejecutar los siguientes comandos si el stdout del comando git status no está vacío:
     - Ejecuta un comando git para cambiar a la nueva rama proporcionada a través del input target-branch.
     - Añade tanto los archivos package.json como package-lock.json a los archivos en staged para un commit.
     - Hacer commit de ambos archivos con el mensaje que consideres adecuado.
     - Hacer push de los cambios a la rama remota proporcionada a través del input target-branch. Es posible que tengas que añadir un -u origin ${targetBranch} después de git push para que funcione correctamente.
     - Abre un PR usando la API de Octokit. Aquí tienes el fragmento necesario para abrir el PR:
       ```javascript
       // Al principio del archivo, importa el paquete @actions/github
       const github = require('@actions/github');
       // Código restante
       const octokit = github.getOctokit(ghToken);
       try {
           await octokit.rest.pulls.create({
               owner: github.context.repo.owner,
               repo: github.context.repo.repo,
               title: `Update NPM dependencies`,
               body: `This pull request updates NPM packages`,
               base: baseBranch,
               head: targetBranch
           });
       } catch (e) {
              core.error('[js-dependency-update] : Something went wrong while creating the PR. Check logs below.');
              core.setFailed(e.message);
              core.error(e);
       }
       ```
15. Confirmar los cambios y hacer push del código. Desencadenar el flujo de trabajo desde la interfaz de usuario, y tómate unos momentos para inspeccionar la salida de la ejecución del flujo de trabajo. ¿Qué sucedió cuando el flujo de trabajo se ejecutó una segunda vez y el PR ya estaba abierto?
16. ¡Felicidades! Has creado una acción personalizada en JavaScript que comprueba si hay actualizaciones en las dependencias de npm de un proyecto y crea un PR con los archivos package*.json actualizados. 