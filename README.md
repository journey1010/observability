# observability-stack

Stack de observabilidad (logs + metrics) para todos los quadlets rootless
del VPS, pa no seguir a `podman logs -f <container>` uno por uno.

**Loki + Promtail** → logs, vía journald (log-driver default de este
podman rootless — confirmado con `podman info --format
'{{.Host.LogDriver}}'`). **Prometheus + prometheus-podman-exporter +
node-exporter** → metrics por container y de host. **Grafana** → un solo
dashboard con logs y métricas correlacionados.

No se usa cAdvisor: en rootless pelea con cgroup delegation. En su lugar,
`prometheus-podman-exporter` solo llama al socket de podman, cero
privilegios extra.

## Por qué no hace falta tocar los otros repos

Los containers de cualquier otro proyecto (pide-api, etc.) ya loguean a
journald por default — Promtail los levanta automático, sin agregar
labels ni cambiar sus quadlets. El `container` name que ves en Grafana es
el mismo `ContainerName=` que ya pusiste en cada `.container` existente.

## Instalación

```bash
# 1. Directorio de quadlets del usuario systemd rootless
mkdir -p ~/.config/containers/systemd/observability

# 2. Copiar quadlets + config
cp quadlets/*.container quadlets/*.pod quadlets/*.volume \
   ~/.config/containers/systemd/
cp -r config/* ~/.config/containers/systemd/observability/

# 3. Habilitar el socket de podman (lo usa prometheus-podman-exporter)
systemctl --user enable --now podman.socket

# 4. Journal persistente (si tu journal es solo volátil en /run, Promtail
#    no va a tener historial al reiniciar). Opcional pero recomendado:
sudo mkdir -p /var/log/journal && sudo systemctl restart systemd-journald

# 5. Secret de Grafana (mismo patrón que los secrets de pide-api)
podman secret create observability_grafana_admin_pass - <<< "tu_password_real"

# 6. Reload + arrancar
systemctl --user daemon-reload
systemctl --user start observability-pod
```

Grafana queda en `http://<host>:3000` (user `admin`, password del secret).
Datasources (Prometheus + Loki) y el dashboard "Containers Overview" ya
vienen provisionados — no hay que armar nada a mano al entrar.

## Verificar

```bash
# logs llegando a loki
curl -s http://localhost:3100/ready

# targets de prometheus arriba
curl -s http://localhost:9090/api/v1/targets | grep health

# métricas del exporter (confirmar nombres si cambia de versión)
curl -s http://localhost:9882/metrics | grep ^podman_container_ | head
```

## Estructura

```
quadlets/                  # unidades systemd rootless (podman quadlet)
  observability.pod        # pod compartido, solo Grafana sale al host (3000)
  loki.container
  promtail.container
  prometheus.container
  podman-exporter.container
  node-exporter.container
  grafana.container
  *.volume                 # volúmenes nombrados (datos persistentes)
config/
  loki-config.yaml
  promtail-config.yaml     # scrape journald, relabel container/container_tag
  prometheus.yml           # scrape prometheus + podman-exporter + node-exporter
  grafana/provisioning/    # datasources + dashboard provider (auto-carga)
  grafana/dashboards/containers-overview.json
```

## Notas / gotchas

- **Retención de Loki**: 15 días (`limits_config.retention_period` en
  `loki-config.yaml`). Ajustar según disco.
- **Retención de Prometheus**: 15 días (`--storage.tsdb.retention.time`
  en `prometheus.container`).
- Los queries de metrics por container hacen join `* on(id) group_left(name)
  podman_container_info` porque el exporter etiqueta todo por `id`, no
  por nombre — si un container se recrea (nuevo id), sus series viejas
  quedan huérfanas hasta que expira la retención (normal en Prometheus).
- Si algún container corre con `--log-driver=k8s-file` en vez de
  journald (override explícito), Promtail no lo va a ver — agregar un
  job de scrape de archivo aparte para ese caso.
- Nombres de métricas verificados contra
  `navidys/prometheus-podman-exporter` v1.14.0 — si se sube de versión
  mayor, correr el `curl :9882/metrics` de arriba antes de confiar en los
  dashboards.
