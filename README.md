# TechMarket Orders — EFT AUY1104


## Caso de negocio

TechMarket requiere desplegar nuevas versiones del servicio Orders sin interrumpir completamente la atención de usuarios. La solución implementa:

- Pruebas automatizadas.
- Construcción y publicación de imágenes.
- Versión Stable conocida.
- Despliegue Canary aislado.
- Validación de salud.
- Tráfico aproximado 75/25.
- Promoción automática.
- Rollback automático.

## Adaptación del laboratorio

La pauta original plantea Amazon EKS y ECR. Por restricciones de los laboratorios se utiliza Amazon EC2 con K3s y Docker Hub:

La adaptación conserva Kubernetes, manifiestos declarativos, estrategias de despliegue y remediación automática.

## Arquitectura

```text
GitHub Actions
      |
      | workflow_call
      v
AUY1104-SDLC-II/devops-library
├── Build y Push Docker
└── Deploy Canary
      |
      +------> Docker Hub
      |
      +------> SSH/SCP
                    |
                    v
              Amazon EC2
                    |
                    v
                  K3s
          ┌─────────┴─────────┐
          │                   │
     Stable 3 pods       Canary 1 pod
          │                   │
          └────── Service ────┘
```

## Estructura relevante

```text
my-apps/
├── .github/workflows/
│   └── eft-canary-strategy.yaml
├── k8s/
│   ├── canary-namespace.yaml
│   ├── canary-stable-deployment.yaml
│   ├── canary-deployment.yaml
│   └── canary-service.yaml
├── src/
│   ├── index.js
│   └── lib/ejemplo.js
├── tests/
├── Dockerfile
├── package.json
└── README.md
```

## Aplicación

La API escucha en el puerto `3000`.

Endpoints:

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/health` | Estado de salud. |
| GET | `/api/saludo` | Saludo JSON. |
| POST | `/api/echo` | Devuelve el cuerpo recibido. |
| GET | `/api/suma` | Suma valores por query. |
| POST | `/api/suma` | Suma valores enviados en JSON. |

Ejecución local:

```bash
npm install
npm start
```

Pruebas:

```bash
npm test
```

## Docker

Construcción local:

```bash
docker build -t auy1104-app .
```

Ejecución:

```bash
docker run --rm -p 3000:3000 auy1104-app
```

Validación:

```bash
curl http://localhost:3000/health
```

La imagen del pipeline utiliza:

```text
mafernandezz/auy1104-app:<github.sha>
```

El SHA entrega trazabilidad entre código, imagen y despliegue.

## Pipeline principal

Archivo:

```text
.github/workflows/eft-canary-strategy.yaml
```

Flujo:

```text
workflow_dispatch
      |
      v
Build reutilizable
      |
      | needs: build
      v
Deploy Canary reutilizable
```

El pipeline consume:

```text
AUY1104-SDLC-II/devops-library
```

Ejemplo:

```yaml
jobs:
  build:
    uses: AUY1104-SDLC-II/devops-library/.github/workflows/eft-docker-deploy.yaml@main
    secrets: inherit

  deploy:
    needs: build
    uses: AUY1104-SDLC-II/devops-library/.github/workflows/eft-canary-deploy.yaml@main
    with:
      stable_image: mafernandezz/auy1104-app@sha256:8cf9ad3c7a1ae510034a85eaa00107edf167ac785e29165e8f5985581c5c59b4
      server_ip: ${{ vars.K3S_SERVER_PUBLIC_IP }}
      image_name: auy1104-app
      namespace: canary-orders
    secrets: inherit
```

## Configuración de GitHub

Secrets:

```text
DOCKER_USERNAME
DOCKER_PASSWORD
EA2_SSH_PRIVATE_KEY
```

Variable:

```text
K3S_SERVER_PUBLIC_IP
```

Los valores sensibles no se almacenan en el repositorio.

## Estrategia Canary

### Estado inicial

```text
orders-api-service
→ track: stable
→ 3 pods Stable
```

Canary se despliega con:

```text
track: canary
1 réplica
```

La validación utiliza:

```text
orders-api-canary-validation
```

Este Service permite comprobar Canary antes de habilitar tráfico público.

### Tráfico 75/25

Después de validar Canary, el selector público queda temporalmente:

```yaml
selector:
  app: orders-api
```

El Service selecciona:

```text
3 pods Stable
1 pod Canary
```

La proporción esperada es aproximadamente 75%/25%.

### Promoción

Si Canary funciona:

1. Se detiene su tráfico.
2. Stable adopta la imagen Canary.
3. Se espera `rollout status`.
4. Canary se elimina.
5. Se valida Stable.

### Rollback

Si Canary falla:

1. El Service vuelve a `track: stable`.
2. Stable recupera la imagen conocida.
3. Se espera su rollout.
4. Canary se elimina.
5. Se valida `/health`.
6. El resultado queda en `GITHUB_STEP_SUMMARY`.

## Estrategias comparadas

| Estrategia | Disponibilidad | Costo | Velocidad | Rollback |
|---|---|---|---|---|
| Recreate/All-in-one | Baja durante el cambio | Bajo | Alta | Requiere reemplazar nuevamente. |
| Rolling Update | Alta | Medio | Media | `rollout undo`. |
| Canary | Alta y menor radio de impacto | Medio | Media | Retirar Canary y conservar Stable. |
| Blue-Green | Muy alta | Alto | Alta durante el switch | Cambiar selector al ambiente anterior. |

Canary fue seleccionada porque:

- Limita el impacto de una versión defectuosa.
- Mantiene tres pods Stable.
- Permite validar antes de promover.
- Consume menos recursos que dos ambientes completos Blue-Green.
- Facilita rollback rápido.

## Escenarios de error

### Fallo en tests

```text
npm test falla
→ no se construye imagen
→ Deploy no comienza
```

### Imagen inexistente

```text
ImagePullBackOff
→ rollout status falla
→ rollback
→ Stable continúa activa
```

### Probe inválida

```text
Canary no queda Ready
→ no recibe tráfico
→ rollout status falla
→ rollback
```

### Error HTTP o latencia

Las solicitudes utilizan `curl -f` y tiempo máximo. Un error HTTP o timeout detiene la promoción y activa rollback.

## Validación en EC2

Pods e imágenes:

```bash
sudo k3s kubectl get pods \
  -n canary-orders \
  -o custom-columns='NAME:.metadata.name,READY:.status.containerStatuses[0].ready,IMAGE:.spec.containers[0].image,TRACK:.metadata.labels.track'
```

Selector:

```bash
sudo k3s kubectl get service orders-api-service \
  -n canary-orders \
  -o jsonpath='{.spec.selector}{"\n"}'
```

Endpoints:

```bash
sudo k3s kubectl get endpointslices \
  -n canary-orders \
  -l kubernetes.io/service-name=orders-api-service \
  -o wide
```

Health:

```bash
curl http://IP_EC2:30080/health
```

Eventos:

```bash
sudo k3s kubectl get events \
  -n canary-orders \
  --sort-by=.lastTimestamp
```

## Impacto técnico y de negocio

- Las pruebas bloquean artefactos defectuosos.
- Los workflows reutilizables reducen duplicación.
- Los tags SHA mejoran trazabilidad.
- Canary disminuye el radio de impacto.
- El rollback reduce MTTR.
- Stable protege la continuidad operativa.
- La automatización reduce errores manuales.
- La promoción acelera el time-to-market.

## Costos y disponibilidad

Canary utiliza temporalmente cuatro pods: tres Stable y uno Canary. Esto incrementa el consumo durante la validación, pero cuesta menos que Blue-Green, que mantendría dos ambientes completos.

En el laboratorio, todos los pods utilizan una misma EC2. No existe costo adicional de EKS, pero sí consumo de CPU, memoria, almacenamiento y transferencia de la instancia.

## Limitaciones

- K3s se ejecuta en una única EC2.
- Un fallo total de la instancia afecta todos los pods.
- El reparto 75/25 es aproximado.
- Docker Hub sustituye ECR.
- `stable_image` debe actualizarse después de promoción.
- La observabilidad se limita a probes, rollouts, endpoints y logs.

## Citas y referencias

Las citas directas, paráfrasis y referencias externas deben presentarse según normas APA, séptima edición.

Docker. (s. f.). *Build and push Docker images*. GitHub Marketplace. Recuperado el 17 de julio de 2026, de https://github.com/marketplace/actions/build-and-push-docker-images

GitHub. (s. f.). *Reuse workflows*. GitHub Docs. Recuperado el 17 de julio de 2026, de https://docs.github.com/en/actions/how-tos/reuse-automations/reuse-workflows

Labels and Selectors. (2026, 15 marzo). Kubernetes. https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/

Kubernetes Authors. (s. f.). *Deployments*. Kubernetes Documentation. Recuperado el 17 de julio de 2026, de https://kubernetes.io/docs/concepts/workloads/controllers/deployment/

Kubernetes Authors. (s. f.). *Service*. Kubernetes Documentation. Recuperado el 17 de julio de 2026, de https://kubernetes.io/docs/concepts/services-networking/service/

Kubernetes Authors. (s. f.). *Configure liveness, readiness and startup probes*. Kubernetes Documentation. Recuperado el 17 de julio de 2026, de https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/

Walia, A. S. (2025, 9 enero). Mastering sed Command in Linux: A Comprehensive Guide. DigitalOcean. https://www.digitalocean.com/community/tutorials/linux-sed-command
## Declaración de uso de inteligencia artificial

Se utilizó inteligencia artificial generativa como apoyo para revisar workflows, analizar errores, estructurar pruebas y organizar esta documentación. La responsabilidad sobre la exactitud, ejecución y entrega corresponde a Matías Fernández

## Autor

Matías Fernández — AUY1104, Ciclo de Vida del Software II.