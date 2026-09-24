# Estado declarativo GitOps - Prácticas 8 y 9

Este repositorio es la única fuente de verdad que ArgoCD observa para desplegar la plataforma. No contiene código de los microservicios ni secretos en texto plano.

## Estructura

```text
applications/                   Definiciones de las aplicaciones ArgoCD
environments/
  dev/
    values/                     Versión y registro deseados
    rendered/                   Manifiestos consumidos por ArgoCD
  prod/
    values/
    rendered/
  p9/
    governance/                 Namespace, cuota y límites reconstruibles
    values/                     Release estable y referencias de P9
    rendered/                   Plataforma recuperable consumida por ArgoCD
applications/
  p9/                           Aplicaciones hijas del app-of-apps `sa-p9-root`
```

Los charts fuente y el renderizador viven en `P8` del repositorio `Practicas-SA-B-202200263`. El workflow de release modifica exclusivamente los tags de imagen en los valores y manifiestos de producción, y abre un Pull Request. ArgoCD nunca lee una rama de código ni recibe instrucciones desde GitHub Actions. `loans-consumer` reutiliza la imagen firmada de `loans-service`, porque es el worker del mismo microservicio y no una copia del código. `runtime-infrastructure` declara PostgreSQL, RabbitMQ e ingress-nginx para que el clúster P8 nazca vacío y todo el runtime llegue desde GitOps.

P9 agrega el patrón app-of-apps. Terraform instala únicamente ArgoCD y registra `sa-p9-root`; la raíz lee `applications/p9`, instala los controladores y después
reconcilia `environments/p9/rendered`. Los respaldos se escriben en GCS mediante Velero y los secretos vuelven desde Google Secret Manager mediante External Secrets, ambos externos al clúster que se destruye.

## Reglas de operación

- Los cambios estructurales bajo `rendered/` se generan con `P8/scripts/render-gitops.sh` y entran en un PR separado.
- El workflow de release solo cambia tags; no puede introducir otros cambios declarativos.
- Toda modificación de producción debe entrar mediante Pull Request y revisión.
- Los tags son SemVer y nunca se usa `latest`.
- Los tags iniciales `v0.0.0-bootstrap.1` son marcadores no desplegables; la primera release estable los sustituye mediante PR antes de registrar la aplicación.
- Los `ExternalSecret` solo referencian objetos de Google Secret Manager.
- ArgoCD usa sincronización automática, `prune` y `selfHeal` para corregir drift.
- Una versión rechazada por Argo Rollouts permanece declarada hasta que se promueva una versión corregida, pero no recibe tráfico: el Rollout vuelve automáticamente al ReplicaSet estable.

## Registro inicial en ArgoCD

Después de ejecutar el Terraform y cargar Secret Manager, registrar producción una sola vez:

```powershell
argocd app create --file applications/sa-p8-prod.yaml
argocd app sync sa-p8-prod
argocd app get sa-p8-prod
```

El ambiente dev es opcional porque comparte el clúster, pero requiere endpoints de base de datos y RabbitMQ accesibles desde `sa-p8-dev`:

```powershell
argocd app create --file applications/sa-p8-dev.yaml
```
