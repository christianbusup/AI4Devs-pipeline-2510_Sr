# Prompts utilizados

## 1. Prompt para Tests

**Prompt:**

```text
Actúa como un experto en DevOps y GitHub Actions.
Necesito un job de GitHub Actions para ejecutar tests de un backend Node.js en un monorepo.
Contexto:
- Directorio del backend: ./backend
- Stack: Node.js 20
- Comando de test: "npm ci && npm test"
Requisitos:
- El job debe llamarse "tests".
- Debe usar caching para npm.
- Debe validar que existen PRs abiertos para la rama actual antes de correr (usando gh api).
- Debe validar si hubo cambios en ./backend (git diff) antes de correr.
- Si no hay PR o no hay cambios, debe marcarse como skipped o éxito sin ejecutar nada pesado.
Genera el YAML para este job incluyendo los pasos de chequeo previo.
```

**Decisiones clave:**

- Separar la lógica de "check-conditions" en un job previo o steps iniciales para mantener el workflow limpio.
- Usar `gh api` para verificar PRs abiertos, aprovechando el `GITHUB_TOKEN` integrado.
- Implementar `git diff --name-only HEAD^ HEAD` para detectar cambios específicos en la carpeta `backend/`.
- Usar `fetch-depth: 2` en el checkout para asegurar que git pueda comparar con el commit anterior.

## 2. Prompt para Build

**Prompt:**

```text
Actúa como experto en CI/CD.
Necesito un job "build" para GitHub Actions que dependa del job "tests".
Contexto:
- Backend en Node.js 20, directorio ./backend.
- Comando: "npm ci && npm run build".
Requisitos:
- Ejecutar el build solo si los tests pasaron.
- Generar un artefacto (archivo .tar.gz) que contenga todo el directorio backend listo para deploy (excluyendo o incluyendo node_modules según best practice para transporte, prefiero excluir y reinstalar en destino).
- El artefacto debe tener el nombre "backend-build-{SHA}".
- Subir el artefacto usando actions/upload-artifact.
Entrégame el snippet YAML del job build.
```

**Decisiones clave:**

- Empaquetar el código fuente/build como `.tar.gz` para preservar permisos y facilitar la transferencia.
- Excluir `node_modules` del artefacto para reducir tamaño de transferencia (instalación limpia en el servidor).
- Versionar el nombre del artefacto con `${{ github.sha }}` para trazabilidad.

## 3. Prompt para Deploy en EC2

**Prompt:**

```text
Actúa como experto en AWS y seguridad.
Necesito un job "deploy" que despliegue un artefacto Node.js a una instancia EC2.
Requisitos:
- Depende del job "build".
- Autenticación SSH usando llave privada (secret EC2_SSH_KEY).
- Pasos en el servidor remoto:
  1. Crear carpeta releases/<SHA>.
  2. Copiar el tar.gz via SCP.
  3. Descomprimir e instalar dependencias (npm ci --production).
  4. Actualizar symlink 'current' apuntando al nuevo release.
  5. Reiniciar servicio systemd "backend".
- Hardening: StrictHostKeyChecking activado (usar ssh-keyscan), manejo seguro de la llave privada.
- Variables disponibles en secrets: EC2_HOST, EC2_USER, EC2_SSH_KEY, APP_DIR.
Dame el YAML completo para este job.
```

**Decisiones clave:**

- Uso de `ssh-keyscan` para poblar `known_hosts` dinámicamente pero manteniendo verificación estricta después.
- Estructura de despliegue atómico con directorios `releases/` y symlink `current` (patrón Capistrano/Deployer) para permitir rollbacks rápidos y evitar tiempo de inactividad durante la copia.
- Uso de `sudo systemctl restart` asumiendo que el usuario tiene permisos sudoers configurados para ese servicio específico.
