# 🏥 Historia Clínica Distribuida con PostgreSQL + Citus en Minikube

Este laboratorio implementa una solución de base de datos distribuida para almacenar historias clínicas usando PostgreSQL con la extensión Citus. El despliegue se realiza sobre Minikube (entorno Kubernetes local), simulando un entorno productivo y escalable.

---

## 🎯 Objetivo

- Desplegar un clúster PostgreSQL-Citus (1 Coordinator + 2 Workers)
- Fragmentar automáticamente la tabla `usuario` por `documento_id`
- Exponer un middleware Python como cliente para realizar consultas distribuidas
- Integrar este laboratorio con prácticas de Gobierno del Dato

---

## ⚙️ Requisitos

- [Minikube](https://minikube.sigs.k8s.io/)
- `kubectl` configurado localmente
- Docker funcionando
- Python 3 (para pruebas locales opcionales)

---

## 🗂️ Estructura

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

## 🚀 Instrucciones de despliegue

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

## 📌 Notas adicionales

- Puedes escalar horizontalmente los workers con:

```bash
kubectl scale deploy citus-worker-1 --replicas=2
```

- Este laboratorio puede extenderse integrando Prometheus, Grafana o FastAPI para observabilidad y API REST.

---

## 👨‍🏫 Uso en el aula

Este laboratorio está pensado para clases de:
- Bases de datos distribuidas
- Gobierno del dato
- Interoperabilidad en salud
- Arquitectura cloud-native

---

## 📘 Licencia

MIT © 2025 – Proyecto académico para fines educativos.
