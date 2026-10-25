# php83-el7

在 CentOS 7 上构建 PHP 8.3 的 RPM 包。`php83.spec` 取自 remi 的 php83 源包，只做 EL7 适配，其余保持上游逻辑。

## 目录结构

```text
php83.spec                      # RPM spec（remi 上游 + EL7 适配）
ci/el7/build.sh                 # 构建脚本：配源 → 解包 src.rpm → 校验 → 装依赖 → rpmbuild
.github/workflows/build-el7.yml # CI：下载 src.rpm → centos:7 构建 → 上传产物
```

## 本地构建

```bash
mkdir -p .ci/input dist
curl -fsSL -o .ci/input/php-8.3.36-1.remi.src.rpm \
  https://rpms.remirepo.net/SRPMS/php-8.3.36-1.remi.src.rpm

docker run --rm -v "$PWD:/work" -w /work centos:7 bash ci/el7/build.sh
```

产物在 `dist/`：34 个二进制包 + 1 个 src.rpm。

构建依赖全部来自已 EOL 的归档源（vault.centos.org、EPEL 7 archive、remi el7），网络不稳时脚本会走阿里云镜像兜底并重试。

## CI

`.github/workflows/build-el7.yml`：

- `workflow_dispatch` 手动触发；`push` tag `v83*` 触发
- 产物 artifact：`php83-el7`（保留 30 天）
- 勾选 `push_package_cloud`（或打 tag）时推送到 `wuwx/php83/el/7`，需要 secret `PACKAGECLOUD_TOKEN`

## EL7 适配点

| 问题 | 处理 |
|---|---|
| EL7 仓库只有 Oracle Instant Client 21 | `oraclever/oracledir` 覆盖为 21（`--with oci8` 使用） |
| 系统 OpenSSL 1.0.2 缺 `ASN1_STRING_get0_data` | 改用 EPEL7 的 `openssl11`（1.1.1k），构建时预置 `OPENSSL_CFLAGS/OPENSSL_LIBS` 跳过 pkg-config |
| glibc 2.17 把 `strncmp` 定义成宏，与 `ZEND_STRL` 的实参计数冲突 | `-D_HAVE_STRING_ARCH_strncmp` 抑制该宏 |
| rpm 4.11 不支持富依赖 `(A with B)` | oci8 的 BuildRequires 降级为普通版本约束 |
| EL7 的 `unixODBC-devel` 没有 pkgconfig 文件 | 直接依赖 `unixODBC-devel` |
| 上游要求 ICU >= 74，EL7 只有 remi 的 libicu73 | EL7 分支放宽到 `>= 73` |
| rpm 4.11 不认识 `%bcond_with(out)` | 构建前写入 `/etc/rpm/macros.d/macros.bcond` |
| el9 打的 src.rpm 无法 `rpm -i` | 改用 `rpm2cpio \| cpio` 解包（payload 是 gzip，rpm 4.11 能读） |

## 已知限制

- **不产出 `php-zip`**：EL7 的 libzip 是 0.10.1，低于 PHP 要求的 0.11，而 remi/EPEL7 都没有更高版本。
- **运行期依赖 `openssl11-libs`**（EPEL7），不是系统自带的 OpenSSL 1.0.2。
- 构建强依赖已归档的 EL7 源，长期需要关注这些镜像是否还在。

## 升级 PHP 版本

1. `.github/workflows/build-el7.yml` 里下载地址与文件名改为新的 src.rpm
2. `php83.spec` 里 `%global upver` 改为新版本号
3. 重新导出 remi 的新 spec，`git diff` 对比后把 EL7 适配部分套回去（适配点见上表）
4. 本地跑一遍 `ci/el7/build.sh`，重点看 `%prep` 的补丁是否能干净应用
