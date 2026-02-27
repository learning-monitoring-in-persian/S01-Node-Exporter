[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Node Exporter

> ### نکته
> اگر قصد دارید **Node Exporter** را روی سیستمی نصب کنید که از قبل **Docker** دارد، یا می‌خواهید ابزارهایی مثل **Prometheus** و **Grafana** را هم کنار آن اجرا کنید، پیشنهاد ‌می‌کنم از نسخه‌ی **Docker**‌ی Node Exporter استفاده کنید تا سرویس‌تان منعطف‌تر باشد.
>
> اما اگر این ماشین فقط قرار است Node Exporter را اجرا کند و Docker هم ندارد، بهتر است برای جلوگیری از سربار اضافی، نسخه‌ی باینری را مستقیم نصب کنید.

## نصب نسخه‌ی باینری Node Exporter

برای راه‌اندازی نسخه‌ی باینری Node Exporter طبق مستندات رسمی، دستورات زیر را اجرا کنید:
```bash
# نکته‌ مهم
# اگر معماری سیستمی که Node EXporter قرار است روی آن نصب شود amd64 نیست دستورات زیر به درستی برای شما کار نخواهند کرد.
# برای مثال اگر معماری شما arm64 است باید تمامی  `amd64` ها را با `arm64` در کامند‌های زیر جایگزین کنید:

VER="$(curl -s https://api.github.com/repos/prometheus/node_exporter/releases/latest | grep -m1 '"tag_name"' | cut -d'"' -f4)"
TAR="node_exporter-${VER#v}.linux-amd64.tar.gz"

wget "https://github.com/prometheus/node_exporter/releases/download/$VER/$TAR" \
  && tar xvfz "$TAR" \
  && cd "node_exporter-${VER#v}.linux-amd64"

./node_exporter
```

بعد از اجرا، Node Exporter روی پورت **9100** در دسترس است. برای تست:
```bash
curl http://{IP_ADDRESS}:9100/metrics
```

> ### نکته مهم
> اگر Node Exporter را روی یک سرور اجرا می‌کنید و ابزارهای دیگری (مثل Prometheus) باید آن را scrape کنند، پورت Node Exporter باید قابل دسترسی باشد.
> اگر فایروال فعال دارید، باید پورت را با **ufw** یا **iptables** یا **nftables** باز کنید.

> ### نکته
> اگر نمی‌خواهید پورت Node Exporter را به بیرون از ماشین expose کنید و Prometheus روی یک ماشین دیگر قرار دارد، می‌توانید **Prometheus Agent** را کنار Node Exporter روی همین ماشین اجرا کنید.
> درواقع Agent میاد Node Exporter را روی `localhost:9100` اسکریپ می‌کند و متریک‌ها را به Prometheus Server روی ماشین دیگر ارسال می‌کند.

> ### نکته
> برای تغییر پورت، Node Exporter را با این فلگ اجرا کنید:
>
> `./node_exporter --web.listen-address="0.0.0.0:9200"`
>
> برای دیدن همه گزینه‌هایی که میتوانید تنظیم کنید هم دستور پایین را اجرا کنید:
>
> `./node_exporter --help`

اگر ماشین ری‌استارت شود، اجرای دستی Node Exporter متوقف می‌شود. برای اینکه به شکل سرویس اجرا شود، مراحل زیر را انجام دهید.

بهتر است فایل باینری Node Exporter را به یک مسیر اجرایی مثل `/usr/local/bin/` منتقل کنید:
```bash
sudo mv node_exporter /usr/local/bin/
```

همچنین پیشنهاد می‌شود یک کاربر بدون امکان لاگین بسازید و Node Exporter را به‌صورت سرویس **systemd** اجرا کنید:
```bash
sudo useradd --no-create-home --shell /bin/false node_exporter
```

سپس فایل `/etc/systemd/system/node_exporter.service` را با محتوای زیر بسازید:
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

در نهایت سرویس را فعال و اجرا کنید:
```bash
sudo systemctl daemon-reload
sudo systemctl enable node_exporter
sudo systemctl start node_exporter
```

## راه‌اندازی Node Exporter با Docker

دستور زیر را اجرا کنید:
```bash
docker run -d \
  --net="host" \
  --pid="host" \
  -p 9100:9100 \
  -v "/:/host:ro,rslave" \
  quay.io/prometheus/node-exporter:latest \
  --path.rootfs=/host
```

> ### نکته
> گزینه‌ی `rslave` نوع mount را به حالت **recursive slave** از نظر mount propagation تنظیم می‌کند.
>
> - حالت **slave** یعنی رویدادهای mount روی میزبان (مثل وصل شدن دیسک جدید یا mountهای جدید در Kubernetes) به داخل کانتینر منتقل می‌شوند.
> - اما mountهایی که داخل کانتینر رخ می‌دهند، به میزبان برنمی‌گردند.
>
> پیشوند `r` یعنی **recursive**؛ یعنی این رفتار فقط برای همان mount point نیست و شامل همه‌ی زیرشاخه‌ها (submounts) در حال حاضر و آینده هم می‌شود.
>
> خلاصه: `rslave` باعث می‌شود کانتینر mountهای جدید میزبان را ببیند، ولی تغییرات mount داخل کانتینر روی سیستم میزبان اثر نگذارد.

نسخه‌ی Docker Compose معادل دستور بالا:
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

## نکته: Collectorها و فیلتر کردن Filesystem

طبق README رسمی Node Exporter: https://github.com/prometheus/node_exporter

ابزار Node Exporter از چندین **collector** تشکیل شده است (مثل CPU، meminfo، filesystem، netstat و ...).  
این collectorها با فلگ‌های خط فرمان کنترل می‌شوند:

- فعال کردن یک collector به‌صورت صریح:
```text
--collector.<name>
```

- غیرفعال کردن یک collector که به‌صورت پیش‌فرض فعال است:
```text
--no-collector.<name>
```

- فعال کردن *فقط* مجموعه‌ای مشخص از collectorها:
```text
--collector.disable-defaults --collector.<name> --collector.<name>
```

### فلگ‌های Include و Exclude

برخی collectorها (معمولاً `filesystem`) از فیلترهای **include/exclude** پشتیبانی می‌کنند.

- الگوی **Exclude** → همه چیز جمع‌آوری می‌شود **به‌جز** مواردی که با الگو match شوند.
- الگوی **Include** → فقط مواردی جمع‌آوری می‌شوند که با الگو match شوند.
- اگر هر دو پشتیبانی شوند، **همزمان قابل استفاده نیستند** (یا include یا exclude).

مثال: حذف mount pointهای سیستمی/مجازی مثل `/sys`، `/proc`، `/dev` برای جلوگیری از متریک‌های نویزی یا بی‌استفاده:
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



> ### نکته
> در آینده در مورد نوشتن اکسپورتر کاستوم توسط آپشن `textfile` ابزار `Node Exporter` در یک ریپازیتوری دیگر صحبت خواهم کرد :) 
