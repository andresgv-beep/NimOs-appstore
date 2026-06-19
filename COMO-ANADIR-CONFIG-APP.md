# Cómo añadir configuración a una app del catálogo

> Sistema de config/aprovisionamiento · NimOS Beta 8.2
> Contrato completo: APP-CATALOG-SCHEMA.md

## Idea

Para no hinchar `catalog.json`, la config de cada app (los campos que pide el
modal de instalación) vive en un archivo APARTE bajo `main/configs/`. El
catálogo solo lleva una referencia.

```
repo NimOs-appstore/
├── catalog.json              ← ligero · metadatos + configRef
└── configs/
    ├── vscode.config.json     ← config de VSCode
    ├── matrix-synapse.config.json
    └── <app>.config.json      ← un archivo por app con config
```

## Pasos para añadir config a una app

### 1. Crear el archivo configs/<app>.config.json

Ejemplo · `configs/vscode.config.json`:
```json
{
  "configVersion": 1,
  "configFields": [
    {
      "key": "PASSWORD",
      "label": "Contraseña de acceso",
      "type": "password",
      "required": true,
      "secret": true,
      "hint": "La contraseña para entrar a VS Code. Elígela tú."
    }
  ]
}
```

### 2. Añadir configRef a la app en catalog.json

```json
"vscode": {
  "name": "VS Code (code-server)",
  "description": "...",
  "compose": "...",
  "port": 8443,
  "configRef": "configs/vscode.config.json"
}
```

### 3. (Importante) Que el compose USE la variable

El compose de VSCode debe leer la variable PASSWORD del .env. Ejemplo:
```yaml
services:
  code-server:
    image: codercom/code-server:latest
    environment:
      - PASSWORD=${PASSWORD}     # ← lee la del .env (que pone el usuario)
```

Antes el catálogo traía `env: { PASSWORD: "nimos" }` hardcoded. Con el sistema
de config, el usuario elige su PASSWORD en el modal y reemplaza ese default.

## Apps simples (sin config)

Las apps que no necesitan datos (Jellyfin, etc.) NO llevan configRef ni archivo
de config. Instalan directo, sin modal · como siempre.

## Flujo en NimOS

```
1. Usuario pulsa instalar VSCode
2. Frontend ve "configRef": "configs/vscode.config.json"
3. Frontend hace fetch de
   raw.githubusercontent.com/.../main/configs/vscode.config.json
4. Con esa config → muestra el modal pidiendo PASSWORD
5. Usuario pone su contraseña → va al .env (0600) → VSCode arranca con ella
```

## Tipos de campo (configFields) · resumen

```
type:     text | password | number | toggle | select
required: true → no deja instalar vacío
secret:   true → no se loguea, no se devuelve por API, .env a 0600
auto:     { provider: "domain" | "local_ip" | ... } → NimOS pre-rellena
validation: { type: "hostname"|"email"|"url"|"port" } o { regex: "..." }
immutable: true → editable al instalar, solo-lectura después (ej. SERVER_NAME)
```

## postInstall (Capa 2 · acciones tras arrancar)

Para apps que necesitan algo DESPUÉS de arrancar (ej. crear el admin de Matrix):
```json
{
  "configVersion": 1,
  "configFields": [ ... ],
  "postInstall": [
    {
      "id": "create_admin",
      "type": "exec",
      "waitFor": "healthy",
      "container": "matrix_synapse",
      "command": "register_new_matrix_user ... {{ADMIN_USER}} {{ADMIN_PASS}} ...",
      "idempotent": true,
      "fields": [
        { "key": "ADMIN_USER", "label": "Usuario admin", "type": "text", "required": true },
        { "key": "ADMIN_PASS", "label": "Contraseña", "type": "password", "required": true, "secret": true }
      ]
    }
  ]
}
```
NOTA: postInstall (Capa 2) aún NO está implementado en el motor · el modal ya
recoge sus campos, pero la ejecución del comando es trabajo futuro.
