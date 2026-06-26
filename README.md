# Bonos Service (VidalCasino 2.0)

## Descripción

**Bonos Service** es el microservicio encargado de la gestión de bonos y promociones de **VidalCasino 2.0**, desarrollado con **Python** y **FastAPI**.

Este servicio fue migrado desde una instancia **Amazon EC2** hacia un clúster de **Amazon EKS**, permitiendo:

* Alta disponibilidad.
* Tolerancia a fallos.
* Escalado automático mediante Kubernetes.

---

# Arquitectura e Integración

## Base de Datos

El microservicio se conecta directamente a una base de datos **PostgreSQL** compartida con el servicio principal (`casino-backend`).

Comparte las siguientes tablas:

* `usuarios`
* `transacciones`

## Autenticación

El servicio no posee un sistema de autenticación propio.

Valida los tokens JWT utilizando el mismo `JWT_SECRET` configurado en el backend principal.

---

# API

## Prefijo de rutas

```
/api/bonos
```

## Documentación Swagger

```
/docs
```

---

# Endpoints

| Método | Endpoint                       | Descripción                                                                                         |
| ------ | ------------------------------ | --------------------------------------------------------------------------------------------------- |
| GET    | `/api/bonos`                   | Obtiene la lista de bonos disponibles.                                                              |
| GET    | `/api/bonos/mis-bonos`         | Obtiene los bonos reclamados por el usuario autenticado.                                            |
| POST   | `/api/bonos/{codigo}/reclamar` | Reclama un bono, acredita el saldo y registra la transacción.                                       |
| GET    | `/livez`                       | Liveness Probe. Verifica que el contenedor continúe en ejecución. Retorna `200 OK`.                 |
| GET    | `/readyz`                      | Readiness Probe. Verifica la conexión con PostgreSQL. Retorna `200 OK` o `503 Service Unavailable`. |

---

# Desarrollo Local

## Requisitos

* Python 3.12 o superior
* PostgreSQL accesible con las tablas del backend

## 1. Crear el entorno virtual

```bash
python -m venv .venv
```

### Linux / macOS

```bash
source .venv/bin/activate
```

### Windows

```bash
.venv\Scripts\activate
```

---

## 2. Instalar dependencias

```bash
pip install -r requirements.txt
```

---

## 3. Configurar variables de entorno

Copiar el archivo de ejemplo:

```bash
cp .env.example .env
```

Configurar los valores correspondientes al entorno local.

**Importante:** nunca subir el archivo `.env` al repositorio.

---

## 4. Ejecutar el servidor

```bash
uvicorn app.main:app --reload --port 8004
```

El servicio estará disponible en:

```
http://localhost:8004
```

La documentación Swagger estará disponible en:

```
http://localhost:8004/docs
```

---

# Estructura del Proyecto

```text
bonos-service/
├── app/
│   ├── main.py
│   ├── db.py
│   ├── auth.py
│   └── ...
│
├── k8s/
│   ├── deployment.yaml
│   ├── service.yaml
│   └── hpa.yaml
│
├── .github/
│   └── workflows/
│       └── deploy.yaml
│
├── Dockerfile
├── .dockerignore
├── requirements.txt
├── .env.example
└── README.md
```

---

# Despliegue en Kubernetes

Los manifiestos de Kubernetes se encuentran en la carpeta:

```
k8s/
```

## deployment.yaml

Configura:

* Número de réplicas.
* Recursos (`requests` y `limits`).
* `livenessProbe`.
* `readinessProbe`.

## service.yaml

Expone el microservicio mediante un servicio de tipo:

```
ClusterIP
```

para permitir la comunicación interna dentro del clúster.

## hpa.yaml

Configura el **Horizontal Pod Autoscaler (HPA)**.

Escalado configurado:

* Mínimo: 2 réplicas.
* Máximo: 6 réplicas.
* Basado en utilización de CPU.

---

# Contenedorización

El proyecto utiliza un `Dockerfile` basado en:

```
python:3.12-slim
```

Además, incorpora un archivo `.dockerignore` para excluir archivos innecesarios durante la construcción de la imagen, como:

* `.venv`
* `__pycache__`
* archivos temporales

---

# Estrategia de Ramas

El proyecto sigue un flujo **polirepo** con tres ramas principales.

## main

Rama estable del proyecto.

## dev

Desarrollo diario y pruebas locales.

## deploy

Cada `push` o `merge` sobre esta rama dispara automáticamente el pipeline de despliegue.

---

# Pipeline CI/CD

El workflow se encuentra en:

```
.github/workflows/deploy.yaml
```

El pipeline realiza las siguientes etapas.

## 1. Construcción

Genera la imagen Docker utilizando tres etiquetas:

* `vX.Y.Z`
* `latest`
* `${{ github.sha }}`

## 2. Publicación

Publica la imagen en un repositorio privado de **Amazon ECR**.

## 3. Despliegue

Actualiza automáticamente el clúster **Amazon EKS** ejecutando:

```bash
kubectl apply
```

sobre los manifiestos del directorio `k8s/`.

---

# Seguridad

Las credenciales sensibles no se almacenan en el código fuente.

## Kubernetes Secrets

Se utilizan para almacenar:

* Credenciales de PostgreSQL.
* Variables de entorno.
* `JWT_SECRET`.

## GitHub Secrets

Las credenciales temporales de AWS Academy utilizadas por el pipeline de CI/CD se almacenan de forma cifrada mediante **GitHub Secrets**.

No se permiten credenciales en texto plano dentro del repositorio ni en los manifiestos de Kubernetes.
