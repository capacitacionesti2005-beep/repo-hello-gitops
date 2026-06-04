# Reto GitOps con AKS, Flux y Kustomize

Este repositorio contiene la solucion del reto GitOps: desplegar `nginx` en AKS usando Flux CD y Kustomize, sin ejecutar `kubectl apply` para aplicar la aplicacion.

## Objetivo

Flux debe sincronizar el path `./clusters/dev` del repositorio y aplicar los manifiestos generados por Kustomize. El resultado esperado es un `Deployment` y un `Service` de `nginx` corriendo en el namespace `hello-gitops-dev`.

## Estructura

```text
repo-hello-gitops/
+-- apps/
|   +-- nginx/
|       +-- deployment.yaml
|       +-- service.yaml
|       +-- kustomization.yaml
+-- clusters/
    +-- dev/
        +-- namespace.yaml
        +-- kustomization.yaml
```

## Correccion aplicada

El problema del reto es conceptual: Flux puede sincronizar el repositorio, pero si el `kustomization.yaml` del cluster no referencia la base de la aplicacion, no hay recursos que desplegar.

La conexion correcta es:

- `apps/nginx/kustomization.yaml` incluye los manifiestos base `deployment.yaml` y `service.yaml`.
- `clusters/dev/kustomization.yaml` referencia `../../apps/nginx`.
- Flux debe apuntar al path `./clusters/dev`.

Tambien se separo la configuracion por entorno:

- La base `apps/nginx` no fija namespace.
- El entorno `clusters/dev` crea y usa el namespace `hello-gitops-dev`.
- Las etiquetas comunes se agregan desde el overlay `dev`.

## Configurar Flux desde Azure Portal

1. Sube este repositorio a GitHub o Azure DevOps.
2. Entra al cluster AKS en Azure Portal.
3. Abre **GitOps**.
4. Crea una configuracion de Flux.
5. Usa estos valores:

```text
Scope: Cluster
Repository URL: URL del repositorio Git
Branch: main
Path: ./clusters/dev
Sync interval: 1 minute
```

Flux ejecutara el equivalente de `kustomize build ./clusters/dev` y aplicara el resultado al cluster.

## Validacion

No apliques los YAML manualmente con `kubectl apply`. Solo valida el estado:

```bash
kubectl get kustomizations -n flux-system
kubectl get pods -n hello-gitops-dev
kubectl get deploy,svc -n hello-gitops-dev
```

El resultado esperado debe incluir:

```text
deployment.apps/nginx
service/nginx
```

Para probar el servicio desde tu maquina:

```bash
kubectl port-forward -n hello-gitops-dev svc/nginx 8080:80
```

Luego abre:

```text
http://localhost:8080
```

## Prueba de reconciliacion

Cambia `spec.replicas` en `apps/nginx/deployment.yaml`, haz commit y push. Flux debe detectar el cambio y ajustar el numero de replicas en el cluster.

Ejemplo:

```yaml
spec:
  replicas: 3
```

Validacion:

```bash
kubectl get deploy nginx -n hello-gitops-dev
```

## Comandos utiles

Ver el estado de Flux:

```bash
flux get sources git -A
flux get kustomizations -A
```

Forzar reconciliacion:

```bash
flux reconcile source git <nombre-source> -n flux-system
flux reconcile kustomization <nombre-kustomization> -n flux-system
```

Ver eventos si algo falla:

```bash
kubectl describe kustomization <nombre-kustomization> -n flux-system
kubectl get events -A --sort-by=.lastTimestamp
```
