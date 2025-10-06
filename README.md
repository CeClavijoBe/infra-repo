# Infrastructure Repository - GitOps

Este repositorio contiene la configuración de infraestructura para el clúster de Kubernetes gestionado mediante GitOps con ArgoCD.

## Estructura del Repositorio

```
infra-repo/
├── argocd/
│   └── applications/
│       ├── monitoring-app.yaml    # ArgoCD Application para VictoriaMetrics + Grafana
│       └── my-app.yaml            # ArgoCD Application para la aplicación
├── monitoring/
│   ├── Chart.yaml                 # Helm chart del stack de observabilidad
│   ├── values.yaml                # Configuración de VictoriaMetrics y Grafana
│   └── charts/                    # Dependencias (generadas automáticamente)
└── README.md
```

## Requisitos Previos

- Clúster de Kubernetes (Kind/K3s/Minikube o en la nube)
- ArgoCD instalado en el clúster
- kubectl configurado
- Helm 3.x

## Instalación

### 1. Actualizar Dependencias del Helm Chart

```bash
cd monitoring
helm dependency update
cd ..
```

### 2. Desplegar las Applications en ArgoCD

```bash
# Aplicar el monitoring stack
kubectl apply -f argocd/applications/monitoring-app.yaml

# Aplicar la aplicación
kubectl apply -f argocd/applications/my-app.yaml
```

### 3. Verificar el Estado

```bash
# Ver las applications en ArgoCD
kubectl get applications -n argocd

# Ver el estado del monitoring
kubectl get pods -n monitoring

# Ver el estado de la aplicación
kubectl get pods -n default
```

## Acceso a Grafana

Grafana está configurado con NodePort en el puerto 30080.

```bash
# Obtener la URL de Grafana (para Kind)
kubectl get svc -n monitoring | grep grafana

# Acceder a Grafana
# Para Kind, necesitas el port-forward:
kubectl port-forward -n monitoring svc/monitoring-stack-grafana 3000:80

# Luego abre: http://localhost:3000
# Usuario: admin
# Password: admin (CAMBIAR en producción)
```

## Stack de Observabilidad

### Componentes Incluidos

- **VictoriaMetrics Single**: Base de datos de métricas de alta performance
- **VMAgent**: Recolector de métricas (reemplaza a Prometheus)
- **VMAlert**: Sistema de alertas
- **Grafana**: Visualización de métricas
- **Node Exporter**: Métricas de nodos
- **Kube State Metrics**: Métricas del estado del clúster

### Dashboards Preconfigurados

- Kubernetes Cluster Overview (GrafanaNet ID: 7249)
- Node Exporter Full (GrafanaNet ID: 1860)

### Datasource

VictoriaMetrics está preconfigurado como datasource por defecto en Grafana:

- **URL**: `http://vmsingle-monitoring-stack-victoria-metrics-k8s-stack.monitoring.svc:8429`
- **Type**: Prometheus

## Configuración de ArgoCD

### Sync Policy

Ambas applications están configuradas con:

- **Auto-sync**: Habilitado
- **Self-heal**: Habilitado
- **Prune**: Habilitado
- **CreateNamespace**: Habilitado

Esto significa que ArgoCD sincronizará automáticamente los cambios del repositorio Git con el clúster.

## Modificar la Configuración

### Cambiar Recursos de VictoriaMetrics

Edita `monitoring/values.yaml`:

```yaml
victoria-metrics-k8s-stack:
  vmsingle:
    spec:
      resources:
        requests:
          cpu: 200m
          memory: 512Mi
```

### Cambiar Retención de Datos

Edita `monitoring/values.yaml`:

```yaml
victoria-metrics-k8s-stack:
  vmsingle:
    spec:
      retentionPeriod: "30d" # Cambiar según necesidad
```

### Agregar Dashboards Personalizados

Edita `monitoring/values.yaml`:

```yaml
victoria-metrics-k8s-stack:
  grafana:
    dashboards:
      default:
        mi-dashboard:
          gnetId: XXXX
          revision: 1
          datasource: VictoriaMetrics
```

## Troubleshooting

### ArgoCD no sincroniza

```bash
# Revisar el estado de la application
kubectl describe application monitoring-stack -n argocd

# Forzar sincronización manual
argocd app sync monitoring-stack
```

### Grafana no inicia

```bash
# Ver logs de Grafana
kubectl logs -n monitoring -l app.kubernetes.io/name=grafana

# Revisar el PVC
kubectl get pvc -n monitoring
```

### VictoriaMetrics sin métricas

```bash
# Verificar VMAgent
kubectl get vmagent -n monitoring
kubectl logs -n monitoring -l app.kubernetes.io/name=vmagent

# Verificar ServiceMonitors
kubectl get servicemonitor -n monitoring
```

## Limpieza

```bash
# Eliminar las applications (esto eliminará todos los recursos)
kubectl delete -f argocd/applications/

# O desde ArgoCD CLI
argocd app delete monitoring-stack
argocd app delete my-app
```

## Recursos Adicionales

- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [VictoriaMetrics Documentation](https://docs.victoriametrics.com/)
- [Grafana Documentation](https://grafana.com/docs/)
- [Helm Documentation](https://helm.sh/docs/)
