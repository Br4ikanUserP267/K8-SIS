# 🏥 Historia Clínica Distribuida con Citus y Minikube

Este proyecto implementa un **sistema distribuido de historia clínica electrónica**, utilizando **PostgreSQL Citus** como base de datos distribuida y **Flask** como middleware, todo desplegado en un entorno **Kubernetes local (Minikube)**.  

Su objetivo es demostrar un modelo de **gobierno del dato y escalabilidad horizontal** en sistemas de información en salud.

------------------------------------------------------------

## 🧱 Arquitectura General

┌──────────────────┐    ┌────────────────────┐    ┌──────────────────┐
│   Middleware     │───▶│  Citus Coordinator │───▶│  Citus Worker 1  │
│  (Flask API)     │    │   (PostgreSQL)    │    │  (PostgreSQL)    │
└──────────────────┘    └────────────────────┘    └──────────────────┘
                                  │
                                  ▼
                       ┌──────────────────┐
                       │  Citus Worker 2  │
                       │  (PostgreSQL)    │
                       └──────────────────┘

**Componentes:**
- Citus Coordinator: Coordina las consultas distribuidas.
- Citus Workers: Almacenan fragmentos de datos distribuidos.
- Middleware Flask: API REST para interactuar con el sistema.
- Kubernetes (Minikube): Orquestador de contenedores.

------------------------------------------------------------

## 🎯 Objetivos del Laboratorio

- Desplegar un clúster PostgreSQL–Citus (1 coordinador + 2 workers).
- Fragmentar la tabla `usuario` por `documento_id`.
- Exponer un middleware Python con endpoints REST para consultas distribuidas.
- Integrar la práctica con los principios de Gobierno del Dato.

------------------------------------------------------------

## ⚙️ Requisitos Previos

### Software Requerido
- Docker Desktop o Podman
- Minikube
- kubectl
- curl
- Python 3 (para pruebas locales)

------------------------------------------------------------

## 🚀 Despliegue Paso a Paso

### 1️⃣ Iniciar Minikube

minikube start --cpus=4 --memory=6g --driver=docker
minikube status
kubectl get nodes

------------------------------------------------------------

### 2️⃣ Aplicar ConfigMaps y Manifiestos

kubectl apply -f configmap-sqls.yaml
kubectl apply -f middleware-configmap.yaml

kubectl apply -f coordinator-deployment.yaml
kubectl apply -f worker1-deployment.yaml
kubectl apply -f worker2-deployment.yaml

------------------------------------------------------------

### 3️⃣ Conectar Workers al Coordinador

kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_add_node('citus-worker-1', 5432);"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_add_node('citus-worker-2', 5432);"

kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_get_active_worker_nodes();"

------------------------------------------------------------

### 4️⃣ Distribuir Tablas

kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_reference_table('medicos');"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_reference_table('usuario');"

kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_distributed_table('pacientes', 'cedula');"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_distributed_table('consultas', 'paciente_id');"

------------------------------------------------------------

### 5️⃣ Desplegar Middleware

kubectl apply -f middleware-deployment.yaml
kubectl apply -f middleware-service.yaml
kubectl wait --for=condition=ready --timeout=300s pod -l app=middleware

------------------------------------------------------------

### 6️⃣ Acceder al Sistema

kubectl port-forward service/citus-coordinator 15432:5432 &
kubectl port-forward service/middleware-service 8080:8080 &

------------------------------------------------------------

## 🧪 Pruebas del Sistema

curl http://localhost:8080
curl http://localhost:8080/status
curl http://localhost:8080/usuarios
curl http://localhost:8080/citus/workers

------------------------------------------------------------

## 📊 Estructura de la Base de Datos

Tablas distribuidas:
- pacientes → Distribuida por cedula
- consultas → Distribuida por paciente_id

Tablas de referencia:
- medicos
- usuario

------------------------------------------------------------

## 🧰 Diagnóstico y Solución de Problemas

kubectl get pods
kubectl logs -l app=middleware
kubectl describe pod -l app=citus-coordinator

# Reiniciar Minikube si hay errores
minikube delete && minikube start --driver=docker

------------------------------------------------------------

## 🧹 Limpieza

kubectl delete -f .
minikube delete

------------------------------------------------------------

## 📘 Licencia

MIT © 2025 – Proyecto académico para fines educativos.
