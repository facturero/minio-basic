# minio-basic

Proyecto de **infraestructura básica** (no es un microservicio) para desplegar y versionar **MinIO** en tu cluster **k3s**, usando GitOps simple: cualquier cambio en `k8s/` que se suba a `main` se aplica automáticamente vía el runner self-hosted de GitHub Actions.

## ¿Qué hace?

- Despliega **MinIO** (`minio-basic`) con almacenamiento persistente (`PersistentVolumeClaim`, storage class `local-path` de k3s).
- Expone la **API S3** y la **consola web** vía `NodePort` para acceder desde fuera del cluster.
- Crea el Secret `minio-credentials` (mismo nombre y claves que espera `document-service`: `root-user` y `root-password`).

## Puertos expuestos

| Servicio | Puerto interno | NodePort | Acceso |
|---|---|---|---|
| S3 (API) | 9000 | **30900** | endpoint S3: `http://<ip-servidor>:30900` |
| Consola | 9001 | **30901** | navegador: `http://<ip-servidor>:30901` |

## Configuración inicial (una sola vez)

1. Agrega estos **Secrets** en GitHub (Settings → Secrets and variables → Actions) del repo `minio-basic`:

   | Secret | Ejemplo |
   |---|---|
   | `SOPS_AGE_KEY` | `AGE-SECRET-KEY-...` (la clave privada age que desencripta los `.enc`) |

   Las credenciales de MinIO (`MINIO_ROOT_USER` / `MINIO_ROOT_PASSWORD`) viven en `config/secrets.production.env`, cifradas en `config/secrets.production.enc` (el `.env` plano está gitignored y **no** se sube).

2. Confirma que tu runner self-hosted tiene `kubectl` configurado apuntando a tu cluster k3s (normalmente vía `KUBECONFIG=~/.kube/config`, como quedó en la instalación).

3. Sube el proyecto:

   ```bash
   git add .
   git commit -m "infra: minio-basic"
   git push -u origin main
   ```

## Flujo día a día

```bash
git add k8s/
git commit -m "infra: ajuste de recursos minio-basic"
git push
```

El runner aplica los cambios automáticamente. También puedes disparar el deploy manualmente desde la pestaña **Actions** de GitHub (`workflow_dispatch`).

## Re-cifrar credenciales

```bash
sops --encrypt config/secrets.production.env > config/secrets.production.enc
git add config/secrets.production.enc
git commit -m "fix(secrets): actualizar credenciales minio-basic"
git push
```

## Verificar estado manualmente

```bash
kubectl get pods -l app=minio-basic
kubectl logs deployment/minio-basic
```

## Si el auto-despliegue no se dispara

- Confirma en GitHub → pestaña **Actions** si aparece el workflow "Deploy minio-basic to k3s" corriendo o con error.
- Confirma que tu runner self-hosted está **online** (Settings → Actions → Runners, debe verse verde/"Idle").
- Confirma que el push fue a la rama `main` (la que espera el trigger) y que tocó archivos dentro de `k8s/` o `config/`.
- Si el runner está online pero el job falla en el paso de `kubectl`, revisa que el usuario que corre el runner tenga acceso al `kubeconfig` de k3s (variable `KUBECONFIG` visible para el servicio, no solo para tu sesión interactiva).

## Notas

- **Recursos:** límites de memoria/CPU ajustados pensando en un servidor con ~7GB de RAM (ver `resources` en `k8s/minio-basic-deployment.yaml`).
- **Backups:** el PVC sobrevive a caídas de pod y reinicios, pero NO es un backup real. Considera un job `mc mirror` o `lakeFS` hacia otro bucket.
- **Namespace:** se despliega en `default`.
- **Compatibilidad:** el Secret que crea el deploy (`minio-credentials`, claves `root-user`/`root-password`) es el mismo que referencia `document-service`; no lo renombres sin actualizar al mismo tiempo ese servicio.