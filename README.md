# Historia Clínica Distribuida con Citus en Minikube

Este proyecto implementa un sistema de historia clínica distribuida utilizando **PostgreSQL Citus** como base de datos distribuida y **Flask** como middleware API, todo desplegado en **Kubernetes** con **Minikube**.

## 🏗️ Arquitectura del Sistema

```
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│   Middleware    │───▶│  Citus Coordinator │───▶│  Citus Worker 1 │
│  (Flask API)    │    │   (PostgreSQL)    │    │  (PostgreSQL)   │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                  │
                                  ▼
                       ┌─────────────────┐
                       │  Citus Worker 2 │
                       │  (PostgreSQL)   │
                       └─────────────────┘
```

### Componentes:
- **Citus Coordinator**: Nodo coordinador que maneja las consultas distribuidas
- **Citus Workers**: Nodos trabajadores que almacenan los datos distribuidos
- **Middleware Flask**: API REST que expone endpoints para interactuar con la base de datos
- **Kubernetes**: Orquestador de contenedores para el despliegue

## 📋 Prerrequisitos

### Software Requerido:
- **Docker Desktop** o **Podman** (para contenedores)
- **Minikube** (para Kubernetes local)
- **kubectl** (cliente de Kubernetes)
- **curl** (para pruebas de API)
- **WSL2** (en Windows, opcional pero recomendado)

### Instalación de Prerrequisitos en Windows:

```powershell
# Instalar Chocolatey (gestor de paquetes)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Instalar herramientas necesarias
choco install minikube kubernetes-cli docker-desktop -y
```

### En Linux/WSL2:
```bash
# Instalar Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Instalar kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl

# Instalar Minikube
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64
sudo install minikube /usr/local/bin/
```

## 🚀 Instalación y Despliegue

### 1. Clonar el Repositorio
```bash
git clone https://github.com/jaiderreyes/historia-clinica-distribuida-minikube.git
cd historia-clinica-distribuida-minikube
```

### 2. Iniciar Minikube
```bash
# Iniciar Minikube con Docker como driver
minikube start --driver=docker

# Verificar que está funcionando
minikube status
kubectl get nodes
```

### 3. Desplegar ConfigMaps
```bash
# Aplicar configuraciones SQL y del middleware
kubectl apply -f configmap-sqls.yaml
kubectl apply -f middleware-configmap.yaml
```

### 4. Desplegar Citus Coordinator
```bash
# Desplegar el nodo coordinador
kubectl apply -f coordinator-deployment.yaml

# Esperar a que esté listo (puede tardar varios minutos)
kubectl wait --for=condition=ready --timeout=300s pod -l app=citus-coordinator

# Verificar logs de inicialización
kubectl logs -l app=citus-coordinator
```

### 5. Desplegar Citus Workers
```bash
# Desplegar los nodos trabajadores
kubectl apply -f worker1-deployment.yaml
kubectl apply -f worker2-deployment.yaml

# Esperar a que estén listos
kubectl wait --for=condition=ready --timeout=300s pod -l app=citus-worker

# Verificar que todos los pods estén funcionando
kubectl get pods
```

### 6. Configurar el Cluster Citus
```bash
# Conectar workers al coordinador
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_add_node('citus-worker-1', 5432);"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_add_node('citus-worker-2', 5432);"

# Verificar workers conectados
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT * FROM master_get_active_worker_nodes();"
```

### 7. Distribuir las Tablas
```bash
# Crear tablas de referencia (replicadas en todos los nodos)
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_reference_table('medicos');"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_reference_table('usuario');"

# Distribuir tablas principales
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_distributed_table('pacientes', 'cedula');"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT create_distributed_table('consultas', 'paciente_id');"

# Verificar distribución de tablas
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT table_name, citus_table_type FROM citus_tables;"
```

### 8. Desplegar Middleware
```bash
# Desplegar el middleware Flask
kubectl apply -f middleware-deployment.yaml
kubectl apply -f middleware-service.yaml

# Esperar a que esté listo
kubectl wait --for=condition=ready --timeout=300s pod -l app=middleware

# Ver logs del middleware
kubectl logs -f -l app=middleware
```

### 9. Acceder al Sistema
```bash
# Hacer port-forward para acceder desde localhost
kubectl port-forward service/citus-coordinator 15432:5432 &
kubectl port-forward service/middleware-service 8080:8080 &

# Verificar que los port-forwards estén funcionando
jobs
```

## 🧪 Pruebas del Sistema

### API REST del Middleware
```bash
# Health check
curl http://localhost:8080

# Estado de la base de datos
curl http://localhost:8080/status

# Obtener usuarios
curl http://localhost:8080/usuarios

# Ver workers de Citus
curl http://localhost:8080/citus/workers
```

### Consultas Directas a la Base de Datos
```bash
# Conectar directamente al coordinador
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica

# Consulta distribuida de ejemplo
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "
SELECT 
    p.nombre, p.apellido, p.ciudad,
    m.nombre as medico_nombre, m.especialidad,
    c.diagnostico, c.fecha_consulta
FROM pacientes p
JOIN consultas c ON p.id = c.paciente_id
JOIN medicos m ON c.medico_id = m.id
LIMIT 5;"

# Ver estadísticas
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT COUNT(*) as total_pacientes FROM pacientes;"
kubectl exec -it deployment/citus-coordinator -- psql -U admin -d historia_clinica -c "SELECT COUNT(*) as total_consultas FROM consultas;"
```

## 📊 Estructura de la Base de Datos

### Tablas Distribuidas:
- **pacientes**: Distribuida por `cedula`
  - Contiene información básica de pacientes
  - Particionada por cédula para distribución eficiente

- **consultas**: Distribuida por `paciente_id`
  - Almacena consultas médicas
  - Co-ubicada con pacientes para consultas eficientes

### Tablas de Referencia:
- **medicos**: Replicada en todos los nodos
  - Información de médicos
  - Acceso rápido desde cualquier nodo

- **usuario**: Replicada en todos los nodos
  - Usuarios del sistema
  - Datos de autenticación y roles

## 🔧 Endpoints de la API

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| GET | `/` | Health check del middleware |
| GET | `/status` | Estado de conexión con la base de datos |
| GET | `/usuarios` | Lista de usuarios del sistema |
| GET | `/citus/workers` | Lista de workers activos en Citus |

## 📁 Archivos del Proyecto

```
historia-clinica-distribuida-minikube/
├── README.md                      # Este archivo
├── configmap-sqls.yaml           # Scripts SQL de inicialización
├── middleware-configmap.yaml      # Código Python del middleware
├── coordinator-deployment.yaml    # Deployment del coordinador Citus
├── worker1-deployment.yaml       # Deployment del worker 1
├── worker2-deployment.yaml       # Deployment del worker 2
├── middleware-deployment.yaml     # Deployment del middleware Flask
├── middleware-service.yaml       # Servicio del middleware
└── schema-init-job.yaml          # Job de inicialización (opcional)
```

## 🐛 Solución de Problemas

### Problema: Pod en estado "Pending" o "ContainerCreating"
```bash
# Verificar recursos del cluster
kubectl describe node minikube
kubectl get events --sort-by=.metadata.creationTimestamp | tail -10

# Reiniciar Minikube si es necesario
minikube delete && minikube start --driver=docker
```

### Problema: Error de conexión a la base de datos
```bash
# Verificar que todos los pods estén corriendo
kubectl get pods

# Ver logs detallados
kubectl describe pod -l app=citus-coordinator
kubectl logs -l app=citus-coordinator
```

### Problema: Puerto 5432 ya en uso
```bash
# Usar puerto diferente para port-forward
kubectl port-forward service/citus-coordinator 15432:5432 &
```

### Problema: Middleware no responde
```bash
# Verificar logs del middleware
kubectl logs -l app=middleware

# Reiniciar deployment
kubectl rollout restart deployment/middleware-citus
```

## 🔍 Comandos Útiles de Diagnóstico

```bash
# Ver estado general
kubectl get all

# Ver eventos recientes
kubectl get events --sort-by=.metadata.creationTimestamp

# Ver logs de un pod específico
kubectl logs <pod-name>

# Ejecutar comandos dentro de un pod
kubectl exec -it <pod-name> -- /bin/bash

# Ver descripción detallada de recursos
kubectl describe pod <pod-name>
kubectl describe deployment <deployment-name>

# Ver uso de recursos
kubectl top nodes
kubectl top pods
```

## 🧹 Limpieza del Sistema

### Eliminar todos los recursos del proyecto:
```bash
kubectl delete -f .
```

### Eliminar Minikube completamente:
```bash
minikube delete
```

### Reiniciar desde cero:
```bash
minikube delete
minikube start --driver=docker
# Luego seguir los pasos de despliegue
```

## 📈 Características del Sistema

### Escalabilidad:
- ✅ Base de datos distribuida horizontalmente
- ✅ Capacidad de agregar más workers
- ✅ Particionamiento automático de datos

### Alta Disponibilidad:
- ✅ Replicación de tablas de referencia
- ✅ Tolerancia a fallas de workers individuales
- ✅ Reinicio automático de contenedores

### Rendimiento:
- ✅ Consultas paralelas en múltiples nodos
- ✅ Co-ubicación de datos relacionados
- ✅ Índices distribuidos

## 🤝 Contribuciones

Para contribuir al proyecto:

1. Fork el repositorio
2. Crea una rama para tu feature (`git checkout -b feature/nueva-caracteristica`)
3. Commit tus cambios (`git commit -am 'Agregar nueva característica'`)
4. Push a la rama (`git push origin feature/nueva-caracteristica`)
5. Crea un Pull Request

## 📝 Licencia

Este proyecto está bajo la Licencia MIT. Ver el archivo `LICENSE` para más detalles.

## 👥 Autores

- **Jaider Reyes** - Desarrollo inicial - [jaiderreyes](https://github.com/jaiderreyes)

## 🙏 Agradecimientos

- [Citus Data](https://github.com/citusdata/citus) por la extensión de PostgreSQL
- [Kubernetes](https://kubernetes.io/) por la plataforma de orquestación
- [Flask](https://flask.palletsprojects.com/) por el framework web de Python

Este laboratorio implementa una solución de base de datos distribuida para almacenar historias clínicas usando PostgreSQL con la extensión Citus. El despliegue se realiza sobre Minikube (entorno Kubernetes local), simulando un entorno productivo y escalable.

---

##  Objetivo

- Desplegar un clúster PostgreSQL-Citus (1 Coordinator + 2 Workers)
- Fragmentar automáticamente la tabla `usuario` por `documento_id`
- Exponer un middleware Python como cliente para realizar consultas distribuidas
- Integrar este laboratorio con prácticas de Gobierno del Dato

---

## Requisitos

- [Minikube](https://minikube.sigs.k8s.io/)
- `kubectl` configurado localmente
- Docker funcionando
- Python 3 (para pruebas locales opcionales)

---

## Estructura

```
k8s/
├── coordinator-deployment.yaml
├── worker1-deployment.yaml
├── worker2-deployment.yaml
├── configmap-sqls.yaml
├── schema-init-job.yaml
├── middleware-deployment.yaml
└── middleware-configmap.yaml
```

---

## Instrucciones de despliegue

1. **Inicia Minikube:**

```bash
minikube start --cpus=4 --memory=6g
```

2. **Aplica los manifiestos del clúster:**

```bash
kubectl apply -f k8s/configmap-sqls.yaml
kubectl apply -f k8s/coordinator-deployment.yaml
kubectl apply -f k8s/worker1-deployment.yaml
kubectl apply -f k8s/worker2-deployment.yaml
```

3. **Inicializa el esquema y los datos:**

```bash
kubectl apply -f k8s/schema-init-job.yaml
```

4. **Despliega el middleware en Python:**

```bash
kubectl apply -f k8s/middleware-configmap.yaml
kubectl apply -f k8s/middleware-deployment.yaml
```

5. **Verifica los logs del middleware:**

```bash
kubectl logs deploy/middleware-citus
```

Deberías ver un listado de usuarios cargados desde la tabla `usuario`.

---

## Notas adicionales

- Puedes escalar horizontalmente los workers con:

```bash
kubectl scale deploy citus-worker-1 --replicas=2
```

- Este laboratorio puede extenderse integrando Prometheus, Grafana o FastAPI para observabilidad y API REST.

---

## Uso en el aula

Este laboratorio está pensado para clases de:
- Bases de datos distribuidas
- Gobierno del dato
- Interoperabilidad en salud
- Arquitectura cloud-native

---

## 📘 Licencia

MIT © 2025 – Proyecto académico para fines educativos.
#   K 8 - S I S  
 