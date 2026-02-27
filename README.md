[English](README.md) | [فارسی](README-persian.md)

# Set up Node Exporter

> [!NOTE]
> If you plan to install **Node Exporter** on a machine that already has **Docker**, or if you want to run tools like **Prometheus** and **Grafana** alongside it, I recommend using the **Docker** version of Node Exporter. This keeps your setup more flexible.
>
> However, if the machine will only run Node Exporter and doesn’t have Docker, it’s better to avoid extra overhead and install the Node Exporter binary directly.

## Install the Node Exporter binary

To set up the Node Exporter binary based on the official documentation, run the commands below:
```bash
VER="$(curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)"
TAR="node_exporter-${VER#v}.linux-amd64.tar.gz"

wget "https://github.com/prometheus/node_exporter/releases/download/$VER/$TAR" \
  && tar xvfz "$TAR" \
  && cd "node_exporter-${VER#v}.linux-amd64"

./node_exporter
```

After that, Node Exporter will be available on port **9100**. You can test it with:
```bash
curl http://{IP_ADDRESS}:9100/metrics
```

> [!IMPORTANT]
> If you run Node Exporter on a server and other tools need to scrape it (such as Prometheus), the Node Exporter port must be accessible.
> If you have an active firewall, allow the port using **ufw**, **iptables**, or **nftables**.

> [!NOTE]
> If you don’t want to expose the Node Exporter port publicly and Prometheus is running on another machine, you can run a **Prometheus Agent** alongside Node Exporter.
> The agent scrapes Node Exporter on `localhost:9100` and forwards the metrics to the Prometheus Server on the other machine.

> [!NOTE]
> To change the listening port, run Node Exporter with:
>
> `./node_exporter --web.listen-address="0.0.0.0:9200"`
>
> For a full list of options:
>
> `./node_exporter --help`

If your machine restarts, the process will stop. To run Node Exporter as a service, follow the steps below.

It’s recommended to move the Node Exporter binary to an executable path such as `/usr/local/bin/`:
```bash
sudo mv node_exporter /usr/local/bin/
```

It’s also recommended to create a non-login user and run Node Exporter as a **systemd** service:
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

Create the file `/etc/systemd/system/node_exporter.service` with the following content:
```ini
[Unit]
Description=Node Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
```

Finally, enable and start the service:
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

## Set up Node Exporter with Docker

Run the following command:
```bash
docker run -d \
  --net="host" \
  --pid="host" \
  -p 9100:9100 \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

> [!NOTE]
> The `rslave` option configures the mount as a recursive slave in terms of mount propagation.
>
> - **slave** means mount events on the host (e.g., attaching a new disk, Kubernetes creating new mounts) are propagated into the container.
> - Mount events inside the container are **not** propagated back to the host.
>
> The `r` prefix stands for **recursive**, meaning this applies to the specified mount point and all of its existing and future submounts.
>
> In short: `rslave` lets the container see new mounts created on the host, while preventing mount changes inside the container from affecting the host.

The Docker Compose version of the command above:
```yaml
services:
  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - '--path.rootfs=/host'
    network_mode: host
    pid: host
    ports:
      - "9100:9100"
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'
```

## Note: Node Exporter collectors & filesystem filtering

Based on the Node Exporter README file: https://github.com/prometheus/node_exporter

Node Exporter is built from multiple **collectors** (CPU, meminfo, filesystem, netstat, etc.).  
Collectors are controlled via CLI flags:

- Enable a collector explicitly:
```text
--collector.<name>
```

- Disable a default-enabled collector:
```text
--no-collector.<name>
```

- Enable *only* a specific set of collectors:
```text
--collector.disable-defaults --collector.<name> --collector.<name>
```

### Include & exclude flags

Some collectors (most commonly the `filesystem` collector) support **include/exclude** filters.

- **Exclude flags** → collect everything **except** the matching patterns.
- **Include flags** → collect **only** the matching patterns.
- If a collector supports both, they are **mutually exclusive** (use one or the other, not both).

Example use case: exclude virtual/system mount points like `/sys`, `/proc`, `/dev` to avoid noisy or irrelevant metrics.
```yaml
services:
  node_exporter:
    image: quay.io/prometheus/node-exporter:latest
    container_name: node_exporter
    command:
      - '--path.rootfs=/host'
      - "--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)"
    network_mode: host
    pid: host
    ports:
      - "9100:9100"
    restart: unless-stopped
    volumes:
      - '/:/host:ro,rslave'
```

---

> [!note]
> I will talk about setup custom exporter using `textfile` feature of `Node Exporter` in another repo :)
