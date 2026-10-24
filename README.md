# teslamate fix addrs

修复 [TeslaMate](https://github.com/teslamate-org/teslamate) 中缺失的地址，并使用腾讯地图把地址更新为更准确的中文地址。

## 致谢

本项目 fork 自 **[@hipudding](https://github.com/hipudding)** 的开源项目 [hipudding/teslamate_fix_addrs](https://github.com/hipudding/teslamate_fix_addrs)。感谢原作者的开源工作，本仓库的全部功能都建立在原项目的基础之上。

原作者同样感谢 [@WayneJz](https://github.com/WayneJz) 提供的思路，请参见他用 Go 编写的地址修复工具 [teslamate-addr-fix](https://github.com/WayneJz/teslamate-addr-fix)。

本仓库的修改已经以 Pull Request 的形式提交给原作者。原项目合并并发布新版本后，建议改回使用原作者的镜像。

---

## 本仓库的修改

相对于原项目最新版本（`c9cb4a5`，使用腾讯地图的版本），本仓库主要做了以下修改。

### 位置准确，不偏移

- **改用车辆实际记录的 GPS 位置查询地址。** 原版用 `addresses` 表里的坐标查询腾讯地图，这个坐标是 OpenStreetMap（OSM）匹配到的地物的中心点，可能离车辆实际停车位置几百米，长道路上偏得更远。现在改为使用与该地址关联的充电位置、行程终点和行程起点的 GPS 坐标。
- **共用地址时选最常去的位置。** TeslaMate 会让落在同一个 OSM 地物上的所有充电和行程共用一条地址记录。程序会取全部关联位置的中位点，选出离中位点最近的那个真实位置，代表"最常去的地方"，不会被某一次偶然停车带偏。关联位置相距超过 300 米时，日志会打印警告。
- **不做本地坐标换算。** 坐标按 WGS84（GPS 坐标系）发送，并带上 `coord_type=1`，由腾讯在服务端转换为火星坐标（GCJ-02）。程序本地不做任何换算，**数据库中的坐标也从不修改**。

### 修复的问题

- OSM 对海上等无法解析的位置会返回 `{"error": "Unable to geocode"}`，原版遇到这种情况会直接崩溃，容器随之陷入不断重启的循环。现在会记录日志并跳过该记录。
- 模式 0 中，解析失败的记录不会再堵住后面的记录，下一轮会自动重试。行程只补缺失的那一端。
- 模式 1 中，查询失败的地址会记录下来，之后每轮重试，失败 3 次后才放弃。遇到网络错误、配额用完或 key 错误时，会直接停止本轮，**不会漏掉地址**。
- 默认只更新中国境内的地址。腾讯返回的空值不会覆盖已有值，空字符串和 NULL 视为相同。`road`（道路）和 `house_number`（门牌号）总是一起更新，不会出现道路和门牌号对不上的情况。
- 覆盖前会把原来的值备份到 `addresses.raw` 字段，可以用 `--restore-original` 一键回滚。
- 地址去重改为按 TeslaMate 的唯一键 `(osm_id, osm_type)`。
- 依赖版本原来没有锁定。新版 SQLAlchemy 2.1 连接 `postgresql://` 时默认使用 psycopg 3 驱动，而镜像里只装了 psycopg2，导致**新构建的原版镜像连不上数据库**。现在已经修复并锁定了依赖版本。

### 更好用

- 新增试运行模式 `DRY_RUN=true`：只把将要做的修改打印到日志，不写数据库。
- 容器启动后，日志第一行会打印版本号和构建号，方便确认正在运行的版本。
- 遇到 429 或 5xx 错误会自动重试（带退避）。日志里的 key 和签名会打码。
- `docker stop` 时会先保存当前这一批，再退出。常驻模式下数据库重启等错误不会导致进程退出。
- 进度文件改存到 `/data/checkpoint.json`，挂载数据卷后，重建容器也不会丢失进度。
- 新增自动化测试（SQLite 和 PostgreSQL）。由 GitHub Actions 自动构建 amd64 和 arm64 镜像。

---

## 注意事项

**使用前务必先[备份数据库](https://docs.teslamate.org/docs/maintenance/backup_restore)。**

```bash
docker compose exec -T database pg_dump -U teslamate teslamate > teslamate_$(date +%F).bck
```

**第一次使用时请先用 `DRY_RUN=true` 试运行，确认日志里的修改没有问题后，再正式写库。**

## 前提条件

- 你的 TeslaMate 存在[地址缺失问题](https://github.com/adriankumpf/teslamate/issues/2956)。
- 可以通过代理访问 openstreetmap.org（用于补全空地址）。
- 如果要用腾讯地图更新地址，需要先[申请腾讯位置服务 key](https://lbs.qq.com/dev/console/application/mine)，并开启 WebService 功能。

---

## 快速开始

### 1. 修改 TeslaMate 的 `docker-compose.yml`

在 TeslaMate 的 compose 文件中加入以下服务，和 `database` 放在同一个网络：

```yaml
  teslamate_fix_addrs:
    image: ghcr.io/blue-eric/teslamate_fix_addrs:latest   # 也可以固定版本，如 :2.1.0
    container_name: teslamate_fix_addrs
    restart: unless-stopped
    environment:
    - HTTP_PROXY=http://代理IP:端口
    - HTTPS_PROXY=http://代理IP:端口      # OSM 是 https 地址，必须设置这一项
    - NO_PROXY=apis.map.qq.com            # 腾讯地图在国内，直接连接，不走代理
    - DB_USER=teslamate
    - DB_PASSWD=数据库密码
    - DB_HOST=database
    - DB_PORT=5432
    - DB_NAME=teslamate
    - MODE=2                              # 0 只补空地址；1 只用腾讯地图更新；2 两者都做
    - INTERVAL=600                        # 每隔多少秒运行一轮，0 表示只运行一次
    - TZ=Asia/Shanghai
    - TENCENT_KEY=你的腾讯key
    - TENCENT_SK=你的腾讯SK               # key 开启了签名校验时才需要
    - CHECKPOINT_FILE=/data/checkpoint.json
    - DRY_RUN=true                        # 试运行，确认无误后删除这一行
    volumes:
    - ./fix_addrs:/data                   # 保存进度，必须挂载
    depends_on:
    - database
```

### 2. 启动并查看日志

```bash
docker compose pull teslamate_fix_addrs
docker compose up -d teslamate_fix_addrs
docker compose logs -f teslamate_fix_addrs
```

日志第一行类似 `teslamate_fix_addrs 2.1.0 (build 4979798...)`，说明运行的是本仓库的镜像。

### 3. 核对结果（见下文"如何确认地址正确"）

### 4. 正式运行

确认无误后，删除 `DRY_RUN=true` 这一行，然后重新启动容器：

```bash
docker compose up -d teslamate_fix_addrs
```

日志里出现 `saving...` 和 `Checkpoint saved to /data/checkpoint.json`，说明已经在写库。

> 如果之前用旧版运行过，正式运行前请先把 `./fix_addrs/checkpoint.json` 改名备份。否则新版会沿用旧进度，跳过已经处理过的地址。

### 自己构建镜像（可选）

不想使用发布好的镜像，也可以自己从源码构建：

```bash
git clone https://github.com/blue-eric/teslamate_fix_addrs.git
```

然后把 compose 中的 `image:` 一行换成 `build: ./teslamate_fix_addrs`（填你克隆下来的目录路径），再执行 `docker compose up -d --build teslamate_fix_addrs`。

---

## 如何确认地址正确

TeslaMate Grafana 中 **Charge Details（充电详情）** 和 **Drive Details（行程详情）** 页面的地图，使用的是车辆记录的真实 GPS 坐标，底图是 OpenStreetMap，两者坐标系一致，位置不会偏。可以用它和地址名称对照。

**试运行阶段**：数据库还没有更新，新地址只在日志里。可以用下面的命令把日志整理成对照表，每行包含地址 id、位置来源、坐标、新名称和 OSM 地图链接：

```bash
docker compose logs --no-log-prefix teslamate_fix_addrs | awk '
/geocoding address id=/ { match($0,/id=[0-9]+/); id=substr($0,RSTART+3,RLENGTH-3)
  match($0,/\([0-9.]+, [0-9.]+\)/); split(substr($0,RSTART+1,RLENGTH-2),c,", ")
  match($0,/from .* position/); src=substr($0,RSTART+5,RLENGTH-14) }
/Tencent raw:/ && id { match($0,/recommend=[^,]*/); name=substr($0,RSTART+10,RLENGTH-10)
  printf "%s\t%s\t%s,%s\t%s\thttps://www.openstreetmap.org/?mlat=%s&mlon=%s#map=18/%s/%s\n", id, src, c[1], c[2], name, c[1], c[2], c[1], c[2]; id="" }' > check.tsv
```

打开链接，看地图上标记的位置是否就是新名称所说的地方。`typical of N` 表示这条地址被 N 次充电或行程共用，数字越大越值得优先核对。

**正式运行后**：在 Grafana 的 Charges（充电）和 Drives（行程）列表里可以看到更新后的地址名称。点进详情页，对照地图上的位置即可。

> **不要把 GPS 坐标直接贴到高德或腾讯地图里核对。** 它们使用火星坐标（GCJ-02），GPS 坐标在上面会偏几百米，那是地图坐标系不同造成的，不是程序的错误。

## 回滚

把腾讯地图改过的地址全部恢复为原来的值：

```bash
docker compose run --rm teslamate_fix_addrs --restore-original
```

如果要恢复整个数据库，请使用使用前做的备份。

---

## 详细说明

### 工作原理

本程序有两个主要功能。

**补全空地址（模式 0）**

使用 [OpenStreetMap Nominatim API](https://nominatim.openstreetmap.org/ui/reverse.html) 补全空地址。TeslaMate 的每次行程和充电都会记录一个带经纬度的位置，程序用 OSM 接口把这些位置解析成地址。

程序会查找所有地址为空的行程和充电记录，把解析出的地址写入 `addresses` 表，并关联到对应的记录上。

- 按 id 顺序扫描。OSM 无法解析的记录（比如海上的位置）会记录日志并跳过，不影响后面的记录，下一轮会重试。
- 行程只解析缺失的那一端（起点或终点）。一端解析成功，即使另一端失败也会保存。
- 地址按 `(osm_id, osm_type)` 去重，与 TeslaMate 的规则一致。
- 如果完全访问不了 OSM，本轮直接停止，不会把任何记录标记为失败，下一轮重试。

**更新地址详情（模式 1）**

使用[腾讯地图逆地址解析 API](https://lbs.qq.com/service/webService/webServiceGuide/webServiceGcoder) 更新地址。OSM 对国内地址的解析常常不够准确，用腾讯地图获取的国内地址更准确，速度也更快。

- **位置准确**：使用车辆实际记录的 GPS 位置查询，具体见前面"位置准确，不偏移"一节。
- **只更新国内地址**：默认只更新中国境内的地址。设置 `UPDATE_OVERSEAS=true` 可以同时更新境外地址。
- **不用空值覆盖**：腾讯返回的空值不会覆盖已有值。`road` 和 `house_number` 总是一起更新。
- **可回滚**：第一次覆盖前，原值会备份在 `addresses.raw` 的 `teslamate_fix_addrs` 键下，可以用 `--restore-original` 恢复。
- **进度和重试**：第一轮会处理所有地址，之后每轮只处理新增的地址（按 id 判断，进度保存在 checkpoint 文件中）。查询失败的地址每轮重试，失败 `GEOCODER_MAX_ATTEMPTS` 次（默认 3 次）后放弃，失败和放弃的地址 id 都记录在 checkpoint 文件里。遇到网络错误、配额用完（状态码 120/121）、key 或签名错误时，本轮停止，不会跳过任何地址，下一轮从停下的地方继续。

### 运行模式

通过 `-m` / `--mode` 参数或环境变量 `MODE` 设置：

| 值 | 说明 |
|---|---|
| `0` | 只补全空地址 |
| `1` | 只用腾讯地图更新地址 |
| `2` | 两者都做 |

使用模式 1 或 2 时，需要通过 `-k` / `--key` 或环境变量 `TENCENT_KEY` 提供腾讯地图 key。如果 key 开启了签名校验，还需要通过 `--sk` 或环境变量 `TENCENT_SK` 提供 SK。

### 常驻运行

通过 `-i` / `--interval` 或环境变量 `INTERVAL` 设置每轮之间的间隔（单位：秒）。

- `INTERVAL=0`：只运行一次就退出。如果地图服务不可用，退出码为 1。
- `INTERVAL` 大于 0：每隔这么多秒运行一轮。数据库重启等错误会记录日志，下一轮自动重试。

收到 `SIGINT` / `SIGTERM`（例如执行 `docker stop`）时，程序会处理完当前这条记录、保存后再退出。

### 进度文件

进度保存在 `checkpoint.json` 中，路径由 `-c` / `--checkpoint` 或环境变量 `CHECKPOINT_FILE` 设置。Docker 镜像中默认是 `/data/checkpoint.json`。

**请务必把 `/data` 挂载到数据卷或宿主机目录。** 否则重建容器后进度会丢失，模式 1 会把所有地址重新调用一遍腾讯接口，浪费配额。

使用 `--reset-checkpoint` 可以清空进度，从头开始。

### 节省内存

通过 `-b` / `--batch` 或环境变量 `BATCH` 限制每批处理的记录数，可以减少内存占用。每批处理完都会提交一次数据库。

### 参数优先级

所有参数都可以通过命令行或环境变量设置，优先级为：

> 命令行参数 > 环境变量 > 默认值

值为空的环境变量会被忽略，使用默认值。

### 代理

如果你所在的地区无法直接访问 OpenStreetMap，需要设置代理：

```
HTTP_PROXY=http://代理IP:端口
HTTPS_PROXY=http://代理IP:端口
```

也支持 socks5 代理：

```
HTTP_PROXY=socks5://代理IP:端口
HTTPS_PROXY=socks5://代理IP:端口
```

OSM 和腾讯地图的接口都是 https 地址，**必须设置 `HTTPS_PROXY`**，只设置 `HTTP_PROXY` 不起作用。建议同时设置 `NO_PROXY=apis.map.qq.com`，让腾讯地图直接连接。

### 访问数据库

如果你用 Docker 安装了 TeslaMate，可以任选以下一种方式让本程序访问数据库。

1. 在 TeslaMate 的 docker-compose.yml 中通过 `ports` 暴露数据库端口：

   ```yaml
   services:
     database:
       image: postgres:15
       restart: always
       environment:
         - POSTGRES_USER=teslamate
         - POSTGRES_PASSWORD=123456
         - POSTGRES_DB=teslamate
       ports:
         - 5432:5432
   ```

2. 把 teslamate_fix_addrs 加入 TeslaMate 的 docker-compose.yml，与数据库处于同一网络（推荐，见前面"快速开始"一节）。

### 修改 TeslaMate 语言设置的影响

在 TeslaMate 设置中修改语言时，TeslaMate 会用 OSM 重新解析所有地址，并覆盖 city、state、county、country、neighbourhood、display_name 这几个字段，腾讯地图写入的这些值会被替换掉。修改语言后，请用 `--reset-checkpoint` 重新运行一次模式 1。

### 全部参数

| 命令行参数 | 环境变量 | 默认值 | 说明 |
|---|---|---|---|
| `--db-url` | `DB_URL` | | 完整的数据库连接地址，设置后会覆盖下面几项数据库参数 |
| `-u` / `--user` | `DB_USER` | | 数据库用户名 |
| `-p` / `--password` | `DB_PASSWD` | | 数据库密码 |
| `-H` / `--host` | `DB_HOST` | | 数据库主机名或 IP |
| `-P` / `--port` | `DB_PORT` | | 数据库端口 |
| `-d` / `--dbname` | `DB_NAME` | | 数据库名 |
| `-b` / `--batch` | `BATCH` | `10` | 每批处理的记录数 |
| `-t` / `--timeout` | `HTTP_TIMEOUT` | `5` | HTTP 请求超时时间（秒） |
| `-r` / `--retry` | `HTTP_RETRY` | `5` | HTTP 请求最大重试次数（包括 429/5xx 错误） |
| `-i` / `--interval` | `INTERVAL` | `0` | 每轮运行的间隔（秒），0 表示只运行一次 |
| `-m` / `--mode` | `MODE` | `0` | 运行模式：0、1、2 |
| `-k` / `--key` | `TENCENT_KEY` | | 腾讯地图 key |
| `--sk` | `TENCENT_SK` | | 腾讯地图签名 SK |
| `-s` / `--since` | `SINCE` | | 只更新这个日期之后改动过的地址（格式 YYYY-mm-dd） |
| `-ua` / `--user_agent` | `USER_AGENT` | `teslamate_fix_addrs/版本号` | HTTP 请求的 User-Agent。OSM 的使用政策要求如实标识应用，不建议伪装成浏览器 |
| `--osm-interval` | `OSM_INTERVAL` | `1.0` | 两次 OSM 请求之间的间隔（秒） |
| `--geocoder-interval` | `GEOCODER_INTERVAL` | `0.3` | 两次腾讯地图请求之间的间隔（秒） |
| `--geocoder-max-attempts` | `GEOCODER_MAX_ATTEMPTS` | `3` | 同一地址查询失败多少次后放弃 |
| `--update-overseas` | `UPDATE_OVERSEAS` | `false` | 是否同时用腾讯地图更新境外地址 |
| `-c` / `--checkpoint` | `CHECKPOINT_FILE` | `checkpoint.json` | 进度文件路径（Docker 镜像中默认为 `/data/checkpoint.json`） |
| `--reset-checkpoint` | | | 清空进度，从头开始 |
| `--dry-run` | `DRY_RUN` | `false` | 试运行：只打印将要做的修改，不写数据库 |
| `--restore-original` | | | 把腾讯地图改过的地址恢复为原值，然后退出 |

### 直接运行 Python 脚本

需要 Python 3.9 或以上版本：

```bash
pip install -r requirements.txt
python3 teslamate_fix_addrs.py -u teslamate -p 123456 -H 127.0.0.1 -P 5432 -d teslamate -m 2 -k KEY --sk SK --dry-run
```

### 在沙箱中试用

担心损坏现有数据？可以先在沙箱中试用（也可以直接用 `DRY_RUN=true` 试运行）。

1. 准备一台和 TeslaMate 不同的机器，或者另外一组 Docker 容器。
2. [备份](https://docs.teslamate.org/docs/maintenance/backup_restore/) TeslaMate 数据库。
3. 在第 1 步准备的环境中启动一个简单的演示环境：

   ```yaml
   services:
     database:
       image: postgres:15
       restart: always
       environment:
         - POSTGRES_USER=teslamate
         - POSTGRES_PASSWORD=123456
         - POSTGRES_DB=teslamate

     grafana:
       image: teslamate/grafana:latest
       restart: always
       environment:
         - DATABASE_USER=teslamate
         - DATABASE_PASS=123456
         - DATABASE_NAME=teslamate
         - DATABASE_HOST=database
       ports:
         - 3000:3000
       volumes:
         - teslamate-grafana-data:/var/lib/grafana

     teslamate_fix_addrs:
       image: ghcr.io/blue-eric/teslamate_fix_addrs:latest
       container_name: teslamate_fix_addrs
       restart: unless-stopped
       environment:
       - DB_USER=teslamate
       - DB_PASSWD=123456
       - DB_HOST=database
       - DB_PORT=5432
       - DB_NAME=teslamate
       - INTERVAL=300
       - MODE=2
       - TENCENT_KEY=你的腾讯key
       - TENCENT_SK=你的腾讯SK
       volumes:
       - teslamate-fix-addrs-data:/data

   volumes:
     teslamate-grafana-data:
     teslamate-fix-addrs-data:
   ```

4. 把备份[恢复](https://docs.teslamate.org/docs/maintenance/backup_restore/)到这个演示环境。
5. 稍等片刻即可查看效果。Grafana 的默认用户名和密码都是 admin。

---

## 常见问题

**怎么确认运行的是哪个版本？**

看容器日志的第一行 `teslamate_fix_addrs 版本号 (build 提交号)`。如果日志里没有这一行，而是出现 `checking empty records...`，说明运行的还是原项目的旧版本。

**日志里出现 `Failed to load checkpoint`？**

说明进度文件是空的或已损坏，程序会从头开始处理。请确认挂载的是**目录**（如 `./fix_addrs:/data`），而不是单个文件。

**为什么补空地址时连不上 OSM？**

请检查是否设置了 `HTTPS_PROXY`，只设置 `HTTP_PROXY` 不起作用。

**腾讯配额够用吗？**

每个地址查询一次。第一轮会处理全部地址，之后每轮只处理新增的地址。一般个人用户的数据量，腾讯的个人配额足够使用。

---

## 开发与测试

```bash
pip install -r requirements-dev.txt
pytest -q tests                  # 离线测试：SQLite + 模拟 HTTP
TEST_DB_URL=postgresql+psycopg2://用户名:密码@127.0.0.1:5432/test pytest -q tests   # 在 PostgreSQL 上测试（会删除并重建表！）
python3 test_with_sqlite.py      # 联网测试：调用真实的 OSM 和腾讯地图接口（需要设置 TENCENT_KEY）
```

推送到 `main` 分支后，GitHub Actions 会运行测试，并构建 amd64/arm64 镜像发布到 `ghcr.io/<用户名>/teslamate_fix_addrs`。镜像标签为 `latest` 和代码中的版本号（`__version__`）。

---

## 免责声明

请在做好数据备份之后再使用本程序。对于使用本程序造成的任何数据丢失或软件故障，作者**概不负责**。

本项目仅供学习研究使用，**不提供任何网络代理（或其下载链接）**。如果使用网络代理违反了当地法律法规，后果由使用者自行承担。

当你下载、复制、编译或运行本项目的源代码或二进制程序时，即表示你已接受上述免责声明。

## 贡献与反馈

欢迎提交代码或 issue。

- 原项目：[hipudding/teslamate_fix_addrs](https://github.com/hipudding/teslamate_fix_addrs)
- 本仓库：[blue-eric/teslamate_fix_addrs](https://github.com/blue-eric/teslamate_fix_addrs)
