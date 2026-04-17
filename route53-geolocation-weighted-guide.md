# Route 53 地理位置 + 加权路由配置指南

> 场景：`www.example.com` 仅针对印度用户，50% 流量指向 Akamai CDN，50% 指向 CloudFront CDN。其他地区保持默认行为不变。

---

## 术语速查

| 术语 | 含义 |
|------|------|
| **Route 53** | AWS 的 DNS 服务，负责将域名（如 `www.example.com`）解析为 IP 地址或其他域名 |
| **Hosted Zone** | Route 53 中管理一个域名（如 `example.com`）所有 DNS 记录的容器，类似于一个 DNS 区域文件 |
| **CNAME 记录** | 一种 DNS 记录类型，将一个域名指向另一个域名（如 `www.example.com` → `cdn.example.net`） |
| **Alias 记录** | Route 53 特有的记录类型，类似 CNAME 但只能指向 AWS 资源（CloudFront、ELB 等），且查询免费 |
| **Geolocation 路由** | 根据用户的地理位置（国家/大洲）返回不同的 DNS 结果 |
| **Weighted 路由** | 按权重比例将流量分配到多个目标（如 50/50 或 90/10） |
| **SetIdentifier** | 同名同类型的多条记录之间的唯一标识符，Route 53 用它区分同一域名下的不同路由记录 |
| **TTL** | Time To Live，DNS 缓存时间（秒）。TTL 越短，DNS 变更生效越快，但查询成本越高 |
| **Health Check** | Route 53 定期探测端点是否可用，不可用时自动将流量切到健康端点 |
| **EDNS Client Subnet** | DNS 扩展协议，让 DNS 解析器把用户的大致 IP 段告诉权威 DNS，提高地理位置判断精度 |

---

## 1. 架构原理

Route 53 **不支持**在单条记录上同时使用两种路由策略。要实现"地理位置 + 加权"组合，必须使用**两层 CNAME 链**：

```
www.example.com
├── [Geolocation: India (IN)]  → CNAME → india-weighted.example.com
└── [Geolocation: Default (*)] → CNAME → <你的默认目标>

india-weighted.example.com
├── [Weighted: weight=1, SetId=akamai]      → CNAME → www.example.com.edgekey.net
└── [Weighted: weight=1, SetId=cloudfront]  → CNAME → d111111abcdef8.cloudfront.net
```

**工作流程：**

1. 用户查询 `www.example.com` → Route 53 根据来源 IP 的地理位置匹配 Geolocation 记录
2. 印度用户命中 `geo-india` 记录 → 返回 CNAME `india-weighted.example.com`
3. 解析 `india-weighted.example.com` → Route 53 按权重随机选择 Akamai 或 CloudFront
4. 非印度用户命中 `geo-default` 记录 → 直接返回默认目标

**为什么用 CNAME 而不是 Alias：** Alias 记录只能指向 AWS 资源（CloudFront、ELB、S3 等）。Akamai 是外部域名，必须用 CNAME。为保持加权组一致性，CloudFront 目标也使用 CNAME。

**`india-weighted.example.com` 是内部路由子域名**，不需要对外宣传，仅用于实现两层链路。

---

## 2. DNS 查询单价

Route 53 按路由策略类型分档计费，**Weighted 属于 Standard 查询，不是 Geolocation 档**：

| 查询类型 | 前 10 亿次/月 | 超出 10 亿次/月 |
|---------|-------------|---------------|
| **Standard 查询**（含 Simple / Failover / Multivalue / **Weighted**） | **$0.40 / 百万次** | $0.20 / 百万次 |
| Latency-Based Routing 查询 | $0.60 / 百万次 | $0.30 / 百万次 |
| **Geolocation / Geoproximity 查询** | **$0.70 / 百万次** | $0.35 / 百万次 |
| IP-Based Routing 查询 | $0.80 / 百万次 | $0.40 / 百万次 |
| Alias 查询指向 AWS 资源 | **免费** | 免费 |

| 其他费用 | AWS 端点 | 非 AWS 端点（如 Akamai） |
|---------|---------|----------------------|
| Hosted Zone | $0.50/月（前 25 个） | — |
| Health Check（基础） | $0.50/月/个 | **$0.75/月/个** |
| Health Check（高级，含 HTTPS/字符串匹配） | $1.00/月/个 | **$2.00/月/个** |

**本方案的成本影响：**

- **印度用户**：每次查询触发两次 Route 53 解析——第一层 Geolocation（$0.70/百万次）+ 第二层 Weighted（$0.40/百万次）→ 有效成本 **$1.10/百万次查询**
- **非印度用户**：仅触发一次 Geolocation 解析 → **$0.70/百万次查询**
- Health Check（可选）：Akamai 是非 AWS 端点（$0.75/月），CloudFront 是 AWS 端点（$0.50/月）→ 合计 **$1.25/月**（基础）。如果使用自有监控系统管理故障转移，可以不创建 Health Check

> 定价参考：https://aws.amazon.com/route53/pricing/

---

## 3. AWS CLI 完整操作步骤

### 3.1 前置准备

```bash
# 设置变量（替换为实际值）
HOSTED_ZONE_ID="Z1234567890ABC"       # 你的 Hosted Zone ID（查询方法见下方）
AKAMAI_DOMAIN="www.example.com.edgekey.net"
CF_DOMAIN="d111111abcdef8.cloudfront.net"
DEFAULT_TARGET="your-default-endpoint.example.com"
```

如果不知道 Hosted Zone ID：

```bash
# 列出所有 Hosted Zone，找到你的域名对应的 ID
aws route53 list-hosted-zones --query 'HostedZones[?Name==`example.com.`].Id' --output text
# 输出类似：/hostedzone/Z1234567890ABC，取 Z1234567890ABC 部分
```

> ⚠️ **如果 `www.example.com` 已有 DNS 记录**（A / CNAME 等），需要先删除旧记录，否则 `CREATE` 会报 `RRSet already exists`。可以用 `UPSERT` 替代 `CREATE` 来自动覆盖，但请确认你理解旧记录被替换的影响。

如果还没有 Hosted Zone：

```bash
aws route53 create-hosted-zone \
  --name example.com \
  --caller-reference "$(date +%s)"
# 从输出中获取 HostedZoneId
```

### 3.2 创建 Health Check（可选配置）

> Route 53 的加权和地理位置路由**不依赖 Health Check 即可正常工作**。Health Check 的作用是在端点不可用时自动将流量切到健康端点。如果你有自己的监控系统并能通过 API 动态调整权重，可以跳过此步骤。
>
> ⚠️ 跳过 Health Check 意味着：当某个 CDN 不可用时，Route 53 **不会**自动切换流量，需要你的监控系统检测到故障后主动调用 API 将故障端点的权重设为 0。

<details>
<summary>点击展开 Health Check 配置命令</summary>

> **`ResourcePath` 必须是 CDN 上能返回 2xx 的路径。** 下面示例用 `/`，如果你的 CDN 对 `/` 返回 301/403 等非 2xx 状态码，请替换为实际可用的健康检查路径。

```bash
# Akamai Health Check（非 AWS 端点，$0.75/月）
AKAMAI_HC=$(aws route53 create-health-check \
  --caller-reference "akamai-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "www.example.com.edgekey.net",
    "Port": 443,
    "ResourcePath": "/",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }' \
  --query 'HealthCheck.Id' --output text)

# CloudFront Health Check（AWS 端点，$0.50/月）
CF_HC=$(aws route53 create-health-check \
  --caller-reference "cloudfront-$(date +%s)" \
  --health-check-config '{
    "Type": "HTTPS",
    "FullyQualifiedDomainName": "d111111abcdef8.cloudfront.net",
    "Port": 443,
    "ResourcePath": "/",
    "RequestInterval": 30,
    "FailureThreshold": 3
  }' \
  --query 'HealthCheck.Id' --output text)

echo "Akamai HC: $AKAMAI_HC"
echo "CloudFront HC: $CF_HC"

# 记录这两个 ID，回滚时需要用到
echo "请保存以下 ID 用于后续管理和回滚："
echo "  AKAMAI_HC=$AKAMAI_HC"
echo "  CF_HC=$CF_HC"
```

</details>

### 3.3 创建加权记录（第二层：india-weighted.example.com）

> 先创建第二层，再创建第一层，确保 Geolocation 记录指向的目标已存在。
>
> 如果跳过了 Health Check，删除下面命令中的 `"HealthCheckId"` 行即可。

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-akamai",
          "Weight": 1,
          "TTL": 60,
          "ResourceRecords": [{"Value": "www.example.com.edgekey.net"}],
          "HealthCheckId": "'"$AKAMAI_HC"'"
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-cloudfront",
          "Weight": 1,
          "TTL": 60,
          "ResourceRecords": [{"Value": "d111111abcdef8.cloudfront.net"}],
          "HealthCheckId": "'"$CF_HC"'"
        }
      }
    ]
  }'
```

### 3.4 创建地理位置记录（第一层：www.example.com）

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "CNAME",
          "SetIdentifier": "geo-india",
          "GeoLocation": {"CountryCode": "IN"},
          "TTL": 60,
          "ResourceRecords": [{"Value": "india-weighted.example.com"}]
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "CNAME",
          "SetIdentifier": "geo-default",
          "GeoLocation": {"CountryCode": "*"},
          "TTL": 60,
          "ResourceRecords": [{"Value": "'"$DEFAULT_TARGET"'"}]
        }
      }
    ]
  }'
```

### 3.5 验证记录

见第 5 节"验证方法"。

---

## 4. Python SDK (boto3) 完整代码

> 以下代码分为三部分，可以独立复制使用。三部分共享同一组配置参数和工具函数。

### 4.0 配置参数与工具函数

所有后续代码都依赖这段配置。复制任何一部分时，请一并复制此段。

```python
import boto3
import time

r53 = boto3.client("route53")

# ========== 配置参数（替换为实际值）==========
HOSTED_ZONE_ID = "Z1234567890ABC"
AKAMAI_DOMAIN  = "www.example.com.edgekey.net"
CF_DOMAIN      = "d111111abcdef8.cloudfront.net"
DEFAULT_TARGET = "your-default-endpoint.example.com"
ENABLE_HEALTH_CHECKS = False  # 如果有自己的监控系统可动态调权重，设为 False

# 如果不知道 Hosted Zone ID，运行以下命令查询：
# aws route53 list-hosted-zones --query 'HostedZones[?Name==`example.com.`].Id' --output text
# 或使用下方的 find_hosted_zone_id() 函数


def find_hosted_zone_id(domain: str) -> str:
    """根据域名查找 Hosted Zone ID（如果你不知道 ID）。"""
    resp = r53.list_hosted_zones()
    for zone in resp["HostedZones"]:
        if zone["Name"] == f"{domain}.":
            zone_id = zone["Id"].split("/")[-1]  # 去掉 /hostedzone/ 前缀
            print(f"找到 Hosted Zone: {domain} -> {zone_id}")
            return zone_id
    raise ValueError(f"未找到域名 {domain} 的 Hosted Zone")


def wait_for_change(change_id: str):
    """等待 Route 53 变更生效（PENDING → INSYNC）。"""
    waiter = r53.get_waiter("resource_record_sets_changed")
    waiter.wait(Id=change_id)


def apply_change(change_batch: dict) -> str:
    """提交变更并等待生效，返回 Change ID。"""
    resp = r53.change_resource_record_sets(
        HostedZoneId=HOSTED_ZONE_ID,
        ChangeBatch=change_batch,
    )
    change_id = resp["ChangeInfo"]["Id"]
    print(f"  Change {change_id} 状态: {resp['ChangeInfo']['Status']}，等待生效...")
    wait_for_change(change_id)
    print(f"  Change {change_id} 已生效 (INSYNC)")
    return change_id


def build_weighted_record(name, set_id, weight, target, hc_id=None):
    """构建单条加权 CNAME 记录，Health Check 可选。"""
    record = {
        "Name": name,
        "Type": "CNAME",
        "SetIdentifier": set_id,
        "Weight": weight,
        "TTL": 60,
        "ResourceRecords": [{"Value": target}],
    }
    if hc_id:
        record["HealthCheckId"] = hc_id
    return record
```

### 4.1 创建配置

```python
# === 依赖 4.0 的配置参数与工具函数 ===

def create_health_check(fqdn: str, ref_suffix: str) -> str:
    """创建 HTTPS Health Check，返回 Health Check ID。"""
    resp = r53.create_health_check(
        CallerReference=f"{ref_suffix}-{int(time.time())}",
        HealthCheckConfig={
            "Type": "HTTPS",
            "FullyQualifiedDomainName": fqdn,
            "Port": 443,
            "ResourcePath": "/",  # 替换为你的 CDN 上能返回 2xx 的路径
            "RequestInterval": 30,
            "FailureThreshold": 3,
        },
    )
    return resp["HealthCheck"]["Id"]


def create_weighted_records(akamai_hc_id=None, cf_hc_id=None):
    """创建 india-weighted.example.com 的两条加权 CNAME 记录。"""
    apply_change({
        "Changes": [
            {
                "Action": "UPSERT",
                "ResourceRecordSet": build_weighted_record(
                    "india-weighted.example.com", "india-akamai", 1,
                    AKAMAI_DOMAIN, akamai_hc_id,
                ),
            },
            {
                "Action": "UPSERT",
                "ResourceRecordSet": build_weighted_record(
                    "india-weighted.example.com", "india-cloudfront", 1,
                    CF_DOMAIN, cf_hc_id,
                ),
            },
        ]
    })


def create_geolocation_records():
    """创建 www.example.com 的地理位置 CNAME 记录（印度 + 默认）。"""
    apply_change({
        "Changes": [
            {
                "Action": "UPSERT",
                "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "CNAME",
                    "SetIdentifier": "geo-india",
                    "GeoLocation": {"CountryCode": "IN"},
                    "TTL": 60,
                    "ResourceRecords": [{"Value": "india-weighted.example.com"}],
                },
            },
            {
                "Action": "UPSERT",
                "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "CNAME",
                    "SetIdentifier": "geo-default",
                    "GeoLocation": {"CountryCode": "*"},
                    "TTL": 60,
                    "ResourceRecords": [{"Value": DEFAULT_TARGET}],
                },
            },
        ]
    })


def update_weights(akamai_weight: int, cf_weight: int, akamai_hc_id=None, cf_hc_id=None):
    """动态调整权重比例（供外部监控系统调用）。"""
    apply_change({
        "Changes": [
            {
                "Action": "UPSERT",
                "ResourceRecordSet": build_weighted_record(
                    "india-weighted.example.com", "india-akamai",
                    akamai_weight, AKAMAI_DOMAIN, akamai_hc_id,
                ),
            },
            {
                "Action": "UPSERT",
                "ResourceRecordSet": build_weighted_record(
                    "india-weighted.example.com", "india-cloudfront",
                    cf_weight, CF_DOMAIN, cf_hc_id,
                ),
            },
        ]
    })


if __name__ == "__main__":
    akamai_hc, cf_hc = None, None

    # Step 1: 创建 Health Check（可选）
    if ENABLE_HEALTH_CHECKS:
        akamai_hc = create_health_check(AKAMAI_DOMAIN, "akamai")
        cf_hc = create_health_check(CF_DOMAIN, "cloudfront")
        print(f"Health Check 已创建: Akamai={akamai_hc}, CloudFront={cf_hc}")
        print("请保存以上 ID 用于后续管理和回滚。\n")
    else:
        print("跳过 Health Check（使用外部监控系统管理故障转移）\n")

    # Step 2: 创建加权记录（第二层）
    print("创建加权记录 (india-weighted.example.com)...")
    create_weighted_records(akamai_hc, cf_hc)

    # Step 3: 创建地理位置记录（第一层）
    print("创建地理位置记录 (www.example.com)...")
    create_geolocation_records()

    print("\n配置完成！使用以下命令验证:")
    print("  dig @8.8.8.8 www.example.com +subnet=103.21.244.0/24")
```

### 4.2 验证配置

```python
# === 依赖 4.0 的配置参数与工具函数 ===

def get_authoritative_ns() -> str:
    """获取 Hosted Zone 的第一个权威 NS，用于绕过缓存验证加权分布。"""
    resp = r53.get_hosted_zone(Id=HOSTED_ZONE_ID)
    ns = resp["DelegationSet"]["NameServers"][0]
    print(f"权威 NS: {ns}")
    print(f"验证命令: dig @{ns} india-weighted.example.com +short")
    return ns


def list_records(name: str):
    """列出指定域名的所有记录（用于验证配置或回滚前确认）。"""
    resp = r53.list_resource_record_sets(
        HostedZoneId=HOSTED_ZONE_ID,
        StartRecordName=name,
        StartRecordType="CNAME",
        MaxItems="10",
    )
    records = [r for r in resp["ResourceRecordSets"] if r["Name"].rstrip(".") == name]
    for r in records:
        sid = r.get("SetIdentifier", "-")
        w = r.get("Weight", "-")
        geo = r.get("GeoLocation", {})
        vals = [v["Value"] for v in r.get("ResourceRecords", [])]
        hc = r.get("HealthCheckId", "无")
        print(f"  SetId={sid}  Weight={w}  Geo={geo or '-'}  -> {vals}  HC={hc}")
    return records


def get_health_check_status(hc_id: str):
    """查看 Health Check 当前状态。"""
    resp = r53.get_health_check_status(HealthCheckId=hc_id)
    for obs in resp["HealthCheckObservations"]:
        region = obs.get("Region", "?")
        status = obs["StatusReport"]["Status"]
        print(f"  {region}: {status}")


# 使用示例：
# list_records("www.example.com")
# list_records("india-weighted.example.com")
# get_authoritative_ns()
```

### 4.3 回滚/清理

> 如果你使用 CLI 回滚，见第 7 节。以下是等效的 Python 函数。

```python
# === 依赖 4.0 的配置参数与工具函数，以及 4.2 的 list_records ===

def find_health_check_id(fqdn: str) -> "str | None":
    """根据 FQDN 查找 Health Check ID（shell 变量丢失时使用）。"""
    paginator = r53.get_paginator("list_health_checks")
    for page in paginator.paginate():
        for hc in page["HealthChecks"]:
            if hc["HealthCheckConfig"].get("FullyQualifiedDomainName") == fqdn:
                return hc["Id"]
    return None


def delete_geolocation_records():
    """删除 www.example.com 的地理位置记录。"""
    apply_change({
        "Changes": [
            {
                "Action": "DELETE",
                "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "CNAME",
                    "SetIdentifier": "geo-india",
                    "GeoLocation": {"CountryCode": "IN"},
                    "TTL": 60,
                    "ResourceRecords": [{"Value": "india-weighted.example.com"}],
                },
            },
            {
                "Action": "DELETE",
                "ResourceRecordSet": {
                    "Name": "www.example.com",
                    "Type": "CNAME",
                    "SetIdentifier": "geo-default",
                    "GeoLocation": {"CountryCode": "*"},
                    "TTL": 60,
                    "ResourceRecords": [{"Value": DEFAULT_TARGET}],
                },
            },
        ]
    })


def delete_weighted_records():
    """删除 india-weighted.example.com 的加权记录。
    自动读取当前记录的实际 Weight 和 HealthCheckId，避免 InvalidChangeBatch。
    """
    records = list_records("india-weighted.example.com")
    changes = []
    for r in records:
        changes.append({"Action": "DELETE", "ResourceRecordSet": r})
    if changes:
        apply_change({"Changes": changes})


def delete_health_checks():
    """删除 Akamai 和 CloudFront 的 Health Check。"""
    for fqdn in [AKAMAI_DOMAIN, CF_DOMAIN]:
        hc_id = find_health_check_id(fqdn)
        if hc_id:
            r53.delete_health_check(HealthCheckId=hc_id)
            print(f"  已删除 Health Check: {fqdn} ({hc_id})")
        else:
            print(f"  未找到 Health Check: {fqdn}（可能未创建或已删除）")


def rollback():
    """完整回滚：按创建逆序删除所有记录和 Health Check。"""
    print("Step 1: 删除地理位置记录...")
    delete_geolocation_records()

    print("Step 2: 删除加权记录...")
    delete_weighted_records()

    if ENABLE_HEALTH_CHECKS:
        print("Step 3: 删除 Health Check...")
        delete_health_checks()

    print("\n回滚完成。请记得恢复 www.example.com 的原始 DNS 记录。")


# 使用示例：
# rollback()
```

---

## 5. 验证方法

### 5.1 使用 dig + EDNS Client Subnet 模拟印度查询

```bash
# 使用印度 IP 段测试（103.21.244.0 是印度 IP 段）
dig @8.8.8.8 www.example.com +subnet=103.21.244.0/24

# 预期结果：
# www.example.com.  60  IN  CNAME  india-weighted.example.com.
# india-weighted.example.com.  60  IN  CNAME  www.example.com.edgekey.net.
# 或
# india-weighted.example.com.  60  IN  CNAME  d111111abcdef8.cloudfront.net.
```

### 5.2 测试非印度（默认）路由

```bash
# 使用美国 IP 段测试
dig @8.8.8.8 www.example.com +subnet=8.8.8.0/24

# 预期结果：
# www.example.com.  60  IN  CNAME  your-default-endpoint.example.com.
```

### 5.3 验证加权分布

> ⚠️ **不要用公共 DNS（8.8.8.8）测试加权分布**——它会缓存结果，TTL 60 秒内多次查询全部返回同一个答案。应该直接查 Route 53 的权威 NS。

```bash
# 先获取你的 Hosted Zone 的权威 NS
aws route53 get-hosted-zone --id "$HOSTED_ZONE_ID" \
  --query 'DelegationSet.NameServers[0]' --output text
# 假设返回 ns-123.awsdns-45.com

# 直接查权威 NS（不经过缓存），执行 20 次
NS="ns-123.awsdns-45.com"  # 替换为实际 NS
for i in $(seq 1 20); do
  dig @"$NS" india-weighted.example.com +short
done | sort | uniq -c | sort -rn

# 预期：大约各 10 次，因为权威 NS 每次都重新计算权重
```

> Python 等效：`get_authoritative_ns()`（见第 4.2 节）

### 5.4 检查记录配置

```bash
# 查看 www.example.com 的所有记录
aws route53 list-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --query "ResourceRecordSets[?Name=='www.example.com.']"

# 查看 india-weighted.example.com 的加权记录
aws route53 list-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --query "ResourceRecordSets[?Name=='india-weighted.example.com.']"

# 查看 Health Check 状态
aws route53 get-health-check-status --health-check-id "$AKAMAI_HC"
aws route53 get-health-check-status --health-check-id "$CF_HC"
```

> Python 等效：`list_records("www.example.com")` 和 `list_records("india-weighted.example.com")`（见第 4.2 节）

---

## 6. 注意事项与最佳实践

### 架构限制

- **CNAME 不能用于 Zone Apex**：`example.com`（裸域）不能创建 CNAME 记录，本方案仅适用于 `www.example.com` 等子域名
- **必须创建 Default 地理位置记录**：如果没有 `CountryCode: "*"` 的默认记录，非印度用户会收到 NXDOMAIN
- **加权组内所有记录必须同名同类型**：`india-weighted.example.com` 下的两条记录必须都是 CNAME 类型
- **已有记录冲突**：如果 `www.example.com` 已有非 Geolocation 的 CNAME/A 记录，必须先删除旧记录再创建新的 Geolocation 记录，或使用 `UPSERT`

### TTL 建议

- 挂载 Health Check 的记录：**TTL ≤ 60 秒**，确保故障转移快速生效
- 加权组内所有记录的 **TTL 必须一致**，否则 Route 53 会静默覆盖为最后设置的值
- **两层记录的 TTL 建议统一为 60 秒**，避免第一层 TTL 过长导致故障转移延迟（本文档所有记录已统一为 60 秒）
- 初始上线阶段建议用短 TTL（60 秒），稳定后可适当调高以降低查询成本

### Health Check 与故障转移（仅在启用 Health Check 时适用）

- 如果 Akamai 的 Health Check 失败，Route 53 会自动将 100% 流量切到 CloudFront（反之亦然）——无需人工干预
- 建议为加权组内**所有**记录都挂载 Health Check，确保故障转移行为可预测
- Health Check 检查器分布在全球各地，从多个区域发起探测
- ⚠️ **双 CDN 同时故障的风险**：如果启用了 Health Check 但第一层 `geo-india` 记录没有挂载 Health Check，当 Akamai 和 CloudFront 同时不可用时，印度用户**不会**自动 fallback 到 `geo-default`。可以创建一个 [Calculated Health Check](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/health-checks-creating-values.html#health-checks-creating-values-calculated)（聚合两个子 HC，条件为"至少 1 个健康"），挂载到 `geo-india` 记录上来解决
- **不使用 Health Check 的场景**：如果你有自己的监控系统，可以在检测到故障后通过 API 将故障端点的权重设为 0（参见上方"调整权重比例"小节），效果等同于 Health Check 的自动故障转移

### 地理位置精度

- Route 53 使用 **EDNS0 Client Subnet (ECS)** 提高定位精度。如果用户的 DNS 解析器支持 ECS，Route 53 使用截断的客户端 IP 而非解析器 IP 来判断位置
- 部分 IP 地址无法映射到地理位置，这些查询会落到 Default 记录
- 印度的 ISO 3166-1 国家代码为 `IN`

### 权重说明

- 权重范围 0–255，值是相对的。`1:1` 和 `50:50` 效果相同，都是 50/50 分配
- 设置 `Weight=0` 可以停止向该端点发送流量，但不删除记录
- 如果所有加权记录的权重都为 0，Route 53 会以等概率路由到所有记录

### 调整权重比例（灰度切换）

上线初期建议先用不对等权重灰度验证，再逐步调整到 50:50：

```bash
# 示例：先 90% Akamai + 10% CloudFront
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch '{
    "Changes": [
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-akamai",
          "Weight": 9,
          "TTL": 60,
          "ResourceRecords": [{"Value": "www.example.com.edgekey.net"}],
          "HealthCheckId": "'"$AKAMAI_HC"'"
        }
      },
      {
        "Action": "UPSERT",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-cloudfront",
          "Weight": 1,
          "TTL": 60,
          "ResourceRecords": [{"Value": "d111111abcdef8.cloudfront.net"}],
          "HealthCheckId": "'"$CF_HC"'"
        }
      }
    ]
  }'

# 验证无误后，调整为 50:50（Weight 各改为 1）
```

> Python 等效：`update_weights(akamai_weight=9, cf_weight=1)`（见第 4.1 节）

---

## 7. 清理/回滚

如果需要删除配置，按**创建的逆序**操作。

> 如果你使用 Python SDK，可以直接调用 `rollback()` 函数（见第 4.3 节），以下是等效的 CLI 命令。

**先查出需要的 ID（如果 shell 变量已丢失）：**

```bash
# 查出 Health Check ID（如果创建了的话）
aws route53 list-health-checks \
  --query 'HealthChecks[?HealthCheckConfig.FullyQualifiedDomainName==`www.example.com.edgekey.net`].Id' \
  --output text
aws route53 list-health-checks \
  --query 'HealthChecks[?HealthCheckConfig.FullyQualifiedDomainName==`d111111abcdef8.cloudfront.net`].Id' \
  --output text

# 设置变量（如果没有创建 Health Check，跳过这两行，并删除后续 DELETE 命令中的 HealthCheckId 行）
AKAMAI_HC="<从上面输出获取>"
CF_HC="<从上面输出获取>"
```

**Step 1: 删除地理位置记录**

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch '{
    "Changes": [
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "CNAME",
          "SetIdentifier": "geo-india",
          "GeoLocation": {"CountryCode": "IN"},
          "TTL": 60,
          "ResourceRecords": [{"Value": "india-weighted.example.com"}]
        }
      },
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "www.example.com",
          "Type": "CNAME",
          "SetIdentifier": "geo-default",
          "GeoLocation": {"CountryCode": "*"},
          "TTL": 60,
          "ResourceRecords": [{"Value": "'"$DEFAULT_TARGET"'"}]
        }
      }
    ]
  }'
```

**Step 2: 删除加权记录**

> 如果创建时没有挂载 Health Check，删除下面命令中的 `"HealthCheckId"` 行。DELETE 时的记录内容必须与创建时完全一致。
>
> ⚠️ 如果你之前调整过权重（如灰度切换改成了 9:1），DELETE 命令中的 `Weight` 必须与当前实际值一致，否则会报 `InvalidChangeBatch`。先运行以下命令确认当前值：
> ```bash
> aws route53 list-resource-record-sets --hosted-zone-id "$HOSTED_ZONE_ID" \
>   --query "ResourceRecordSets[?Name=='india-weighted.example.com.']"
> ```

```bash
aws route53 change-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --change-batch '{
    "Changes": [
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-akamai",
          "Weight": 1,
          "TTL": 60,
          "ResourceRecords": [{"Value": "www.example.com.edgekey.net"}],
          "HealthCheckId": "'"$AKAMAI_HC"'"
        }
      },
      {
        "Action": "DELETE",
        "ResourceRecordSet": {
          "Name": "india-weighted.example.com",
          "Type": "CNAME",
          "SetIdentifier": "india-cloudfront",
          "Weight": 1,
          "TTL": 60,
          "ResourceRecords": [{"Value": "d111111abcdef8.cloudfront.net"}],
          "HealthCheckId": "'"$CF_HC"'"
        }
      }
    ]
  }'
```

**Step 3: 删除 Health Check（如果创建了的话）**

```bash
# 如果没有创建 Health Check，跳过此步骤
aws route53 delete-health-check --health-check-id "$AKAMAI_HC"
aws route53 delete-health-check --health-check-id "$CF_HC"
```

> 删除后，记得恢复 `www.example.com` 的原始 DNS 记录。

---

## 参考文档

- [Route 53 路由策略概述](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy.html)
- [Geolocation 路由](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-geo.html)
- [Weighted 路由](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-weighted.html)
- [EDNS0 Client Subnet](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/routing-policy-edns0.html)
- [Alias vs Non-Alias 记录](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/resource-record-sets-choosing-alias-non-alias.html)
- [Route 53 定价](https://aws.amazon.com/route53/pricing/)
