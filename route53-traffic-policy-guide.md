# Route 53 Traffic Policy 配置指南：地理位置 + 加权路由

> 场景：`www.example.com` 仅针对印度用户，50% 流量指向 Akamai CDN，50% 指向 CloudFront CDN。其他地区保持默认行为不变。

---

## 术语速查

| 术语 | 含义 |
|------|------|
| **Traffic Policy** | 路由规则的**模板**（JSON 文档），定义"怎么路由"，但不关联任何域名。可以有多个版本。**免费** |
| **Traffic Policy Version** | 策略的版本号。策略一旦创建不可修改，任何变更（改权重、改端点）都需要创建新版本 |
| **Traffic Policy Instance** | 把模板**应用到具体域名**的实例。一个 Instance = 一个域名 + 一个 Policy 版本 + 一个 Hosted Zone。Route 53 自动创建底层 DNS 记录。**$50/月/个** |
| **Policy Document** | 定义路由规则的 JSON 文档，包含 Endpoints（目标）和 Rules（路由规则） |
| **RuleReference** | 规则嵌套的关键——让一个规则（如 Geo）引用另一个规则（如 Weighted），实现多层路由 |

> 类比：**Policy 是蓝图，Instance 是按蓝图盖的房子。** 一张蓝图可以盖多栋房子（多个域名复用同一个 Policy），每栋房子单独收费 $50/月。蓝图本身免费，画多少版本都不收钱。

---

## 1. 架构原理

Traffic Policy 原生支持规则嵌套，**不需要中间过渡域名**。一个 JSON 文档定义完整的路由逻辑：

```
www.example.com（Traffic Policy Instance）
│
├── [Geo Rule: India (IN)] → [Weighted Rule]
│                              ├── 50% → www.example.com.edgekey.net (Akamai)
│                              └── 50% → d111111abcdef8.cloudfront.net (CloudFront)
│
└── [Geo Rule: Default (*)] → default-origin.example.com
```

Route 53 自动管理底层 DNS 记录，你只需要操作 Traffic Policy 和 Instance。

---

## 2. 费用

| 费用项 | 单价 |
|--------|------|
| **Traffic Policy Instance（策略实例）** | **$50.00/月**（按天按比例计费） |
| Traffic Policy 定义和版本 | 免费 |
| **DNS 查询（Geolocation 档）** | **$0.70/百万次**（前 10 亿次/月） |
| DNS 查询（超出 10 亿次/月） | $0.35/百万次 |

- 策略中包含 Geolocation 规则，所有查询按 Geolocation 档计费（$0.70/百万次），Weighted 子规则不额外收费
- 每个域名关联一个 Instance = $50/月。如果多个子域名需要相同策略，可以创建一个 Instance + 其他域名 CNAME 指向它，只收一份 $50

> 定价参考：https://aws.amazon.com/route53/pricing/

---

## 3. Policy Document JSON 格式

Policy Document 由三部分组成，对应架构图中的三层：

1. **Endpoints**（目标）：定义流量最终去哪里——Akamai、CloudFront、默认源站
2. **Rules**（规则）：定义路由逻辑——先按地理位置分流，印度流量再按权重分配
3. **StartRule**（入口）：指定从哪个规则开始执行

```json
{
  "AWSPolicyFormatVersion": "2015-10-01",
  "RecordType": "CNAME",
  "StartRule": "geo_rule",
  "Endpoints": {
    "ep_akamai": {
      "Type": "value",
      "Value": "www.example.com.edgekey.net"
    },
    "ep_cloudfront": {
      "Type": "value",
      "Value": "d111111abcdef8.cloudfront.net"
    },
    "ep_default": {
      "Type": "value",
      "Value": "default-origin.example.com"
    }
  },
  "Rules": {
    "geo_rule": {
      "RuleType": "geo",
      "Locations": [
        {
          "Country": "IN",
          "RuleReference": "india_weighted"
        },
        {
          "Country": "*",
          "EndpointReference": "ep_default"
        }
      ]
    },
    "india_weighted": {
      "RuleType": "weighted",
      "Items": [
        {
          "EndpointReference": "ep_akamai",
          "Weight": "1"
        },
        {
          "EndpointReference": "ep_cloudfront",
          "Weight": "1"
        }
      ]
    }
  }
}
```

**要点：**

- `StartRule` 指向顶层规则（Geo Rule）
- Geo Rule 的印度条目用 `RuleReference` 引用 Weighted Rule——这就是嵌套的实现方式
- `Country: "*"` 是默认兜底，匹配所有未命中的地区和无法映射地理位置的 IP
- `Weight` 是字符串，范围 `"0"`–`"255"`，值是相对的。`"1":"1"` = 50/50
- `RecordType` 必须是 `CNAME`（因为目标是外部域名）
- `AWSPolicyFormatVersion` 支持 `"2015-10-01"` 和 `"2023-05-09"` 两个版本，本场景用哪个都行

---

## 4. AWS CLI 完整操作

### 4.1 前置准备

```bash
HOSTED_ZONE_ID="Z1234567890ABC"  # 替换为实际值
DOMAIN="www.example.com"

# 查询 Hosted Zone ID（如果不知道）
aws route53 list-hosted-zones \
  --query 'HostedZones[?Name==`example.com.`].Id' --output text
# 输出类似：/hostedzone/Z1234567890ABC，取 Z1234567890ABC 部分
```

> Python 等效：

```python
resp = r53.list_hosted_zones()
for zone in resp["HostedZones"]:
    if zone["Name"] == "example.com.":
        print(zone["Id"].split("/")[-1])
```

### 4.2 创建 Traffic Policy

```bash
# 将 Policy Document 保存为文件
cat > policy.json << 'EOF'
{
  "AWSPolicyFormatVersion": "2015-10-01",
  "RecordType": "CNAME",
  "StartRule": "geo_rule",
  "Endpoints": {
    "ep_akamai":     { "Type": "value", "Value": "www.example.com.edgekey.net" },
    "ep_cloudfront": { "Type": "value", "Value": "d111111abcdef8.cloudfront.net" },
    "ep_default":    { "Type": "value", "Value": "default-origin.example.com" }
  },
  "Rules": {
    "geo_rule": {
      "RuleType": "geo",
      "Locations": [
        { "Country": "IN", "RuleReference": "india_weighted" },
        { "Country": "*", "EndpointReference": "ep_default" }
      ]
    },
    "india_weighted": {
      "RuleType": "weighted",
      "Items": [
        { "EndpointReference": "ep_akamai",     "Weight": "1" },
        { "EndpointReference": "ep_cloudfront", "Weight": "1" }
      ]
    }
  }
}
EOF

# 创建策略
aws route53 create-traffic-policy \
  --name "geo-weighted-india" \
  --document file://policy.json \
  --comment "India 50/50 Akamai+CloudFront, default elsewhere"
# 记录输出中的 TrafficPolicy.Id（POLICY_ID）和 Version（1）
```

### 4.3 创建 Traffic Policy Instance（关联域名）

```bash
POLICY_ID="<从上一步输出获取>"

aws route53 create-traffic-policy-instance \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --name "$DOMAIN" \
  --ttl 60 \
  --traffic-policy-id "$POLICY_ID" \
  --traffic-policy-version 1
# 记录输出中的 TrafficPolicyInstance.Id（INSTANCE_ID）
```

### 4.4 查看 Instance 状态

```bash
# 列出所有 Instance
aws route53 list-traffic-policy-instances \
  --output table \
  --query 'TrafficPolicyInstances[].{Name:Name,State:State,TTL:TTL,PolicyVersion:TrafficPolicyVersion}'

# 查看特定策略的 Instance
aws route53 list-traffic-policy-instances-by-policy \
  --traffic-policy-id "$POLICY_ID" \
  --traffic-policy-version 1
```

### 4.5 修改权重（创建新版本 + 更新 Instance）

Traffic Policy 版本不可修改，修改需创建新版本：

```bash
# 复制 policy.json 并修改权重（如改为 90:10）
cp policy.json policy_v2.json
# 将 policy_v2.json 中 india_weighted 的 Weight 从 "1":"1" 改为 "9":"1"
# 修改后的 india_weighted 部分应为：
#   "india_weighted": {
#     "RuleType": "weighted",
#     "Items": [
#       { "EndpointReference": "ep_akamai",     "Weight": "9" },
#       { "EndpointReference": "ep_cloudfront", "Weight": "1" }
#     ]
#   }

# 创建新版本
aws route53 create-traffic-policy-version \
  --id "$POLICY_ID" \
  --document file://policy_v2.json \
  --comment "Changed India weights to 90:10"
# 返回新的 Version（如 2）

# 更新 Instance 指向新版本（零停机，原子切换）
INSTANCE_ID="<从 4.3 获取>"
aws route53 update-traffic-policy-instance \
  --id "$INSTANCE_ID" \
  --ttl 60 \
  --traffic-policy-id "$POLICY_ID" \
  --traffic-policy-version 2
```

### 4.6 删除（按顺序）

```bash
# Step 1: 删除 Instance（同时删除底层 DNS 记录）
aws route53 delete-traffic-policy-instance --id "$INSTANCE_ID"

# ⚠️ Instance 删除是异步的，需要等待完全删除后才能删策略版本
# 检查 Instance 是否已完全删除：
aws route53 list-traffic-policy-instances \
  --query "TrafficPolicyInstances[?Id=='$INSTANCE_ID'].State" --output text
# 返回空值表示已删除；返回 "Deleting" 表示还在进行中，等几秒再查

# Step 2: 删除策略版本（每个版本单独删除，Instance 必须已完全删除）
aws route53 delete-traffic-policy --id "$POLICY_ID" --traffic-policy-version 2
aws route53 delete-traffic-policy --id "$POLICY_ID" --traffic-policy-version 1
```

---

## 5. Python SDK (boto3) 完整代码

### 5.0 配置参数

```python
import json
import boto3

r53 = boto3.client("route53")

HOSTED_ZONE_ID = "Z1234567890ABC"  # 替换为实际值
DOMAIN = "www.example.com"

POLICY_DOC = {
    "AWSPolicyFormatVersion": "2015-10-01",
    "RecordType": "CNAME",
    "StartRule": "geo_rule",
    "Endpoints": {
        "ep_akamai":     {"Type": "value", "Value": "www.example.com.edgekey.net"},
        "ep_cloudfront": {"Type": "value", "Value": "d111111abcdef8.cloudfront.net"},
        "ep_default":    {"Type": "value", "Value": "default-origin.example.com"},
    },
    "Rules": {
        "geo_rule": {
            "RuleType": "geo",
            "Locations": [
                {"Country": "IN", "RuleReference": "india_weighted"},
                {"Country": "*", "EndpointReference": "ep_default"},
            ],
        },
        "india_weighted": {
            "RuleType": "weighted",
            "Items": [
                {"EndpointReference": "ep_akamai",     "Weight": "1"},
                {"EndpointReference": "ep_cloudfront", "Weight": "1"},
            ],
        },
    },
}
```

### 5.1 创建策略 + 关联域名

```python
# === 依赖 5.0 ===

# 创建 Traffic Policy
resp = r53.create_traffic_policy(
    Name="geo-weighted-india",
    Document=json.dumps(POLICY_DOC),
    Comment="India 50/50 Akamai+CloudFront",
)
policy_id = resp["TrafficPolicy"]["Id"]
policy_version = resp["TrafficPolicy"]["Version"]  # 1
print(f"策略已创建: {policy_id} v{policy_version}")

# 创建 Instance（关联域名）
resp = r53.create_traffic_policy_instance(
    HostedZoneId=HOSTED_ZONE_ID,
    Name=DOMAIN,
    TTL=60,
    TrafficPolicyId=policy_id,
    TrafficPolicyVersion=policy_version,
)
instance_id = resp["TrafficPolicyInstance"]["Id"]
print(f"Instance 已创建: {instance_id}")
print("请保存 policy_id 和 instance_id 用于后续操作。")
```

### 5.2 查看 + 修改权重

```python
# === 依赖 5.0 ===

def list_instances(policy_id=None, policy_version=None):
    """列出 Traffic Policy Instance。"""
    if policy_id and policy_version:
        resp = r53.list_traffic_policy_instances_by_policy(
            TrafficPolicyId=policy_id,
            TrafficPolicyVersion=policy_version,
        )
    else:
        resp = r53.list_traffic_policy_instances()
    for inst in resp["TrafficPolicyInstances"]:
        print(f"  {inst['Name']}  State={inst['State']}  "
              f"TTL={inst['TTL']}  Version={inst['TrafficPolicyVersion']}")
    return resp["TrafficPolicyInstances"]


def update_weights(policy_id, akamai_weight, cf_weight, comment=""):
    """创建新策略版本（修改权重），返回新版本号。"""
    doc = json.loads(json.dumps(POLICY_DOC))  # 深拷贝，避免修改原始模板
    doc["Rules"]["india_weighted"] = {
        "RuleType": "weighted",
        "Items": [
            {"EndpointReference": "ep_akamai",     "Weight": str(akamai_weight)},
            {"EndpointReference": "ep_cloudfront", "Weight": str(cf_weight)},
        ],
    }
    resp = r53.create_traffic_policy_version(
        Id=policy_id,
        Document=json.dumps(doc),
        Comment=comment or f"Weights: Akamai={akamai_weight}, CF={cf_weight}",
    )
    new_version = resp["TrafficPolicy"]["Version"]
    print(f"新版本已创建: v{new_version}")
    return new_version


def apply_version(instance_id, policy_id, version, ttl=60):
    """将 Instance 切换到指定策略版本（零停机）。"""
    r53.update_traffic_policy_instance(
        Id=instance_id,
        TTL=ttl,
        TrafficPolicyId=policy_id,
        TrafficPolicyVersion=version,
    )
    print(f"Instance 已更新到 v{version}")


# 使用示例：
# list_instances()
# v2 = update_weights(policy_id, 9, 1, "灰度 90:10")
# apply_version(instance_id, policy_id, v2)
```

### 5.3 删除/回滚

```python
# === 依赖 5.0 ===

def rollback(instance_id, policy_id, old_version, ttl=60):
    """回滚到旧版本（零停机）。"""
    apply_version(instance_id, policy_id, old_version, ttl)
    print(f"已回滚到 v{old_version}")


def delete_all(instance_id, policy_id, versions):
    """完整删除：先删 Instance，等待删除完成，再逐个删策略版本。"""
    import time

    print("Step 1: 删除 Instance...")
    r53.delete_traffic_policy_instance(Id=instance_id)

    # Instance 删除是异步的，需要等待完全删除后才能删策略版本
    print("  等待 Instance 删除完成...")
    for _ in range(60):
        resp = r53.list_traffic_policy_instances()
        if not any(i["Id"] == instance_id for i in resp["TrafficPolicyInstances"]):
            break
        time.sleep(3)
    print("  Instance 已删除")

    print("Step 2: 删除策略版本...")
    for v in sorted(versions, reverse=True):
        r53.delete_traffic_policy(Id=policy_id, Version=v)
        print(f"  已删除 v{v}")

    print("清理完成。")


# 使用示例：
# rollback(instance_id, policy_id, old_version=1)
# delete_all(instance_id, policy_id, versions=[1, 2])
```

---

## 6. 验证方法

### 6.1 查看 Route 53 自动创建的 DNS 记录

Traffic Policy Instance 创建后，Route 53 会自动在 Hosted Zone 中生成记录。

```bash
aws route53 list-resource-record-sets \
  --hosted-zone-id "$HOSTED_ZONE_ID" \
  --query "ResourceRecordSets[?contains(Name, 'www.example.com')]"
```

> Python 等效：

```python
resp = r53.list_resource_record_sets(
    HostedZoneId=HOSTED_ZONE_ID,
    StartRecordName=DOMAIN,
    StartRecordType="CNAME",
    MaxItems="20",
)
for r in resp["ResourceRecordSets"]:
    if DOMAIN in r["Name"]:
        vals = [v["Value"] for v in r.get("ResourceRecords", [])]
        print(f"  {r['Name']}  {r['Type']}  -> {vals}")
```

### 6.2 dig 验证

```bash
# 默认路由
dig www.example.com CNAME +short

# 多次查询观察加权分布
for i in $(seq 1 10); do
  dig www.example.com CNAME +short
  sleep 1
done | sort | uniq -c | sort -rn
```

---

## 7. 操作流程总结

```
创建策略 ──→ 创建 Instance ──→ DNS 自动生效
                                    │
                              需要改权重？
                                    │
                              创建新版本 ──→ 更新 Instance ──→ 零停机切换
                                    │
                              需要回滚？
                                    │
                              更新 Instance 指向旧版本
                                    │
                              需要删除？
                                    │
                              删除 Instance ──→ 逐个删除策略版本
```

---

## 8. 配额（Quota）

Traffic Flow 的默认配额**非常低**，尤其是 Instance 数量。规划前务必确认：

| 资源 | 默认配额 | 可否提升 |
|------|---------|---------|
| **Traffic Policy Instance（策略实例）** | **5 个/账号** | ✅ 可提升 |
| Traffic Policy（策略定义） | 50 个/账号 | ✅ 可提升 |
| Traffic Policy Version（策略版本） | 1,000 个/策略 | ❌ 不可提升 |

> ⚠️ **Instance 默认只有 5 个**，意味着整个账号最多只能将 5 个域名关联到 Traffic Policy。如果你有多个域名需要使用 Traffic Policy，必须提前申请提升配额。

### 如何自助申请提升配额

1. 打开 [Service Quotas 控制台（Route 53）](https://console.aws.amazon.com/servicequotas/home?region=us-east-1#!/services/route53/quotas)

   > ⚠️ Route 53 的配额管理**必须在 US East (N. Virginia) 区域**操作，即使你的资源在其他区域。

2. 在列表中找到要提升的配额项（如 `Traffic policy records`）
3. 点击配额名称进入详情页
4. 点击 **Request quota increase**
5. 输入期望的新配额值，点击 **Request**
6. 等待审批（通常几小时到几天）

也可以用 CLI 查看当前配额和申请提升：

```bash
# 查看当前 Traffic Policy 相关配额
aws service-quotas list-service-quotas \
  --service-code route53 \
  --region us-east-1 \
  --query "Quotas[?contains(QuotaName, 'raffic')].[QuotaCode,QuotaName,Value]" \
  --output table

# 查看 Instance 配额
aws service-quotas get-service-quota \
  --service-code route53 \
  --quota-code L-628D5A56 \
  --region us-east-1
# L-628D5A56 = Traffic flow policy records (instances)
# L-FC688E7C = Traffic flow policies

# 申请提升 Instance 配额
aws service-quotas request-service-quota-increase \
  --service-code route53 \
  --quota-code L-628D5A56 \
  --desired-value 20 \
  --region us-east-1
```

> Python 等效：

```python
sq = boto3.client("service-quotas", region_name="us-east-1")

# 列出所有 Traffic 相关配额
resp = sq.list_service_quotas(ServiceCode="route53")
for q in resp["Quotas"]:
    if "raffic" in q["QuotaName"]:
        print(f"  {q['QuotaCode']}  {q['QuotaName']}  = {int(q['Value'])}")

# 查看特定配额
for code, name in [("L-628D5A56", "Instances"), ("L-FC688E7C", "Policies")]:
    resp = sq.get_service_quota(ServiceCode="route53", QuotaCode=code)
    print(f"Traffic Policy {name}: {int(resp['Quota']['Value'])}")

# 申请提升 Instance 配额到 20
sq.request_service_quota_increase(
    ServiceCode="route53",
    QuotaCode="L-628D5A56",
    DesiredValue=20.0,
)
```

---

## 9. 注意事项

- **策略版本不可修改**：任何变更（权重、端点、规则）都需要创建新版本，然后更新 Instance
- **Instance 切换是原子操作**：更新 Instance 版本时，Route 53 原子替换底层记录，DNS 持续响应
- **必须先删 Instance 再删策略**：策略有关联 Instance 时无法删除。且 Instance 删除是**异步**的（状态从 `Applied` → `Deleting` → 消失），需要等完全删除后才能删策略版本，否则会报 `TrafficPolicyInUse` 错误
- **$50/月/Instance**：每个关联域名收费 $50/月。策略定义和版本本身免费
- **配额**：每账号最多 50 个 Traffic Policy，每个策略最多 1,000 个版本（软限制，可申请提升）
- **Weight 是字符串**：`"1"` 不是 `1`，CLI 和 boto3 都要传字符串
- **CNAME 限制**：和普通 CNAME 一样，不能用于 Zone Apex（裸域）

---

## 参考文档

- [Traffic Flow 概述](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-flow.html)
- [Traffic Policy Document 格式](https://docs.aws.amazon.com/Route53/latest/APIReference/api-policies-traffic-policy-document-format.html)
- [Traffic Policy 管理](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-policies.html)
- [Traffic Policy Instance](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/traffic-policy-records.html)
- [Route 53 定价](https://aws.amazon.com/route53/pricing/)
