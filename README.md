# Laboratorio CI/CD con GitHub Actions y Jenkins

Este repositorio contiene una aplicación web mínima y dos definiciones de pipeline:

* **CI con GitHub Actions:** checkout, instalación de dependencias, análisis estático, pruebas y validación de la imagen Docker.
* **CD con Jenkins:** clonación del repositorio, construcción de la imagen Docker, validación, ejecución del contenedor y prueba de funcionamiento.

## Arquitectura

```text
Desarrollador
     |
     | push / pull request
     v
  GitHub
     |
     +----> GitHub Actions
     |        - Checkout
     |        - npm install
     |        - ESLint
     |        - Jest
     |        - docker build
     |
     +----> Jenkins
              - Clone repository
              - Check Docker
              - Build Docker image
              - Validate Docker image
              - Run Docker container
              - Check container
              - Smoke test
              - Cleanup
               |
               v
               (Exporta métricas en /prometheus/)
```

## Aplicación

La aplicación usa Node.js y Express.

### Endpoints

* `GET /`
* `GET /health`

### Ejecución local

Para ejecutar la aplicación directamente en un entorno de desarrollo:

```bash
npm install
npm test
npm run lint
npm start
```

La aplicación utiliza el puerto `3000`.

## CI con GitHub Actions

Configuración:

```text
.github/workflows/ci.yml
```

El workflow se ejecuta automáticamente ante:

* `push` a `main` o `develop`
* `pull_request` hacia `main` o `develop`

### Pasos

1. Checkout del código.
2. Configuración de Node.js 20.
3. Instalación de dependencias.
4. Análisis estático con ESLint.
5. Pruebas con Jest.
6. Validación de la construcción de la imagen Docker.

## CD con Jenkins

Configuración:

```text
Jenkinsfile
```

El pipeline de Jenkins se ejecuta en el servidor de integración configurado por el equipo.

### Acceso al servidor Jenkins

El servidor Jenkins utilizado para este proyecto se encuentra disponible en:

**http://3.144.124.190:3000**

Desde esta instancia se puede acceder al Job configurado para ejecutar el pipeline definido en el `Jenkinsfile`.

### Stages definidos

1. `Clone repository`
2. `Check Docker`
3. `Build Docker image`
4. `Validate Docker image`
5. `Remove previous container`
6. `Run Docker container`
7. `Check container`
8. `Smoke test`
9. `Cleanup old images`

### Flujo del pipeline

```text
GitHub
   |
   v
Servidor Jenkins
   |
   +--> Clone repository
   |
   +--> Check Docker
   |
   +--> Build Docker image
   |
   +--> Validate Docker image
   |
   +--> Run Docker container
   |
   +--> Check container
   |
   +--> Smoke test
   |
   +--> Cleanup old images
   |
   v
Pipeline exitoso
```

El pipeline construye una imagen Docker utilizando el número de ejecución de Jenkins (`BUILD_NUMBER`) como identificador de la versión y también genera el tag `latest`.

Posteriormente ejecuta un contenedor temporal y realiza una prueba de funcionamiento sobre el endpoint `/health`.

### Configuración del Job

Para configurar el Job de Jenkins:

1. Crear un nuevo Job de tipo **Pipeline**.
2. Seleccionar **Pipeline script from SCM**.
3. Seleccionar **Git** como SCM.
4. Introducir la URL del repositorio:

```text
https://github.com/juansebastian-br/CI-CD-with-jenkins.git
```

5. Configurar la rama:

```text
*/main
```

6. Configurar el Script Path:

```text
Jenkinsfile
```

7. Guardar la configuración.
8. Ejecutar **Build with Parameters**.

### Parámetros

| Parámetro        | Ejemplo                                                      | Uso                        |
| ---------------- | ------------------------------------------------------------ | -------------------------- |
| `GIT_REPOSITORY` | `https://github.com/juansebastian-br/CI-CD-with-jenkins.git` | Repositorio                |
| `GIT_BRANCH`     | `main`                                                       | Rama                       |
| `IMAGE_NAME`     | `cicd-lab-webapp`                                            | Nombre de la imagen Docker |

## Docker

Construcción manual de la imagen:

```bash
docker build -t cicd-lab-webapp:local .
```

Ejecución:

```bash
docker run --rm -p 3000:3000 cicd-lab-webapp:local
```

Validación:

```bash
curl http://3.144.124.190:3000/health
```

Respuesta esperada:

```json
{"status":"UP"}
```

> La referencia a `localhost` en esta validación corresponde al servicio de la aplicación ejecutado localmente o al contenedor dentro del entorno donde se realiza la prueba. No corresponde a la dirección de acceso al servidor Jenkins.

## Subida a GitHub

```bash
git init
git add .
git commit -m "Initial CI/CD laboratory"
git branch -M main
git remote add origin https://github.com/juansebastian-br/CI-CD-with-jenkins.git
git push -u origin main
```

## Evidencias

Las evidencias del funcionamiento de los pipelines pueden consultarse en la carpeta:

```text
capturas/
```

Se recomienda incluir evidencias de:

* Ejecución exitosa de GitHub Actions.
* Ejecución de las pruebas.
* Construcción de la imagen Docker.
* Ejecución del pipeline de Jenkins.
* Construcción de la imagen desde Jenkins.
* Ejecución del contenedor.
* Resultado exitoso del smoke test.

## Entregables incluidos

* Código fuente.
* `.github/workflows/ci.yml`.
* `Jenkinsfile`.
* `Dockerfile`.
* `README.md`.
* `docs/documentacion-tecnica.md`.
* Carpeta `capturas/` para las evidencias.

## Requisitos del servidor Jenkins

El servidor donde se ejecuta Jenkins debe disponer de las herramientas necesarias para ejecutar el pipeline, entre ellas:

* Git
* Docker
* curl

Además, el usuario utilizado por Jenkins debe contar con permisos suficientes para ejecutar Docker.

El pipeline utiliza el Docker Engine disponible en el servidor Jenkins para construir y ejecutar la imagen de la aplicación.


---

## Pila de monitoreo

La infraestructura de monitoreo permite recopilar métricas de los contenedores Docker y del servidor Jenkins, almacenarlas en Prometheus y visualizarlas mediante Grafana.

```text
┌─────────────────────────────────────────────────────────────┐
│                    PILA DE MONITOREO                        │
└─────────────────────────────────────────────────────────────┘

              Contenedores Docker
                      │
                      ▼
              ┌───────────────┐
              │    cAdvisor   │
              │  Puerto 8081  │
              └───────┬───────┘
                      │
                      │ Métricas
                      ▼
              ┌───────────────┐
              │   Prometheus  │
              │  Puerto 9090  │
              └───────┬───────┘
                      │
                      │ Consultas
                      ▼
              ┌───────────────┐
              │    Grafana    │
              │  Puerto 3002  │
              └───────┬───────┘
                      │
             ┌────────┴────────┐
             │                 │
             ▼                 ▼
      ┌──────────────┐  ┌──────────────┐
      │  Dashboards  │  │    Alertas   │
      │ Visualización│  │  SMS / Email │
      └──────────────┘  └──────────────┘


                 Jenkins (Host)
                      │
                      │ /prometheus/
                      │ Puerto 8080
                      │
                      └──────────────► Prometheus
```

### Componentes

| Componente | Puerto | Función |
|---|---:|---|
| Jenkins | `3000` | Ejecución del pipeline CI/CD |
| cAdvisor | `8081` | Recolección de métricas de los contenedores Docker |
| Jenkins Metrics | `8080` | Exposición de métricas de Jenkins |
| Prometheus | `9090` | Recolección y almacenamiento de métricas |
| Grafana | `3002` | Visualización de métricas mediante dashboards |

### Flujo de monitoreo

```text
Contenedores Docker
        │
        ▼
    cAdvisor
        │
        │ Métricas de contenedores
        ▼
    Prometheus ◄──── Jenkins (Host)
        │                 │
        │                 │ /prometheus/
        │                 │ Puerto 8080
        │
        │ Consultas
        ▼
     Grafana
        │
        ├──────────► Dashboards
        │
        └──────────► Alertas
                      │                     
                      └──► Email
```
## Credenciales y accesos

| Servicio | URL | Usuario | Contraseña |
|---|---|---|---|
| Jenkins | http://3.145.86.19:8080/ | `admin` | `admin` |
| Aplicación | http://3.145.86.19:3001/ | — | — |
| Prometheus | http://3.145.86.19:9090/query | `admin` | `admin` |
| Grafana | http://3.145.86.19:3002/ | `admin` | `admin` |

> **Nota:** Estas credenciales corresponden al entorno de laboratorio.