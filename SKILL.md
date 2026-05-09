---
name: parking-whitelist
description: 停车场系统管理 - 黑白名单管理、用户管理、月租车开通、车牌凭证管理、删除人员注销车牌、已有用户增加车牌
triggers:
  - 白名单
  - 黑名单
  - 灰名单
  - 车牌
  - 停车场
  - 月租车
  - 新增用户
  - 开通月租
  - 删除用户
  - 注销车牌
  - 删除人员
  - 批量迁出
  - 增加车牌
  - 再加一个车牌
---

# 停车场系统管理功能

停车场系统 API 操作，支持：黑白名单管理、用户增删改查、月租车开通、车牌凭证管理。

## 系统信息

- **地址**: https://10.0.12.1:9091/
- **模块名**: `systemcenter` (用户管理), `parkmanagement` (车场管理)
- **认证**: Bearer Token

## 认证信息

- **账号**: 9990
- **密码**: 88888888
- **登录API**: `/api/systemcenter/auth/login`
- **密码加密**: MD5

---

## 功能一：黑白名单管理

### 名单类型

| type值 | 类型 |
|--------|------|
| 1 | 黑名单 |
| 2 | 灰名单 |
| 3 | 白名单 |

### 新增名单记录

```python
import hashlib
import requests

# 登录获取Token
base_url = "https://10.0.12.1:9091"
password_md5 = hashlib.md5("88888888".encode()).hexdigest()
resp = requests.post(
    f"{base_url}/api/systemcenter/auth/login",
    json={"account": "9990", "password": password_md5},
    verify=False
)
token = resp.json()['data']['token']
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# 新增白名单
new_record = {
    "plate": "粤Sxxxx",           # 车牌号
    "plateColor": "3",            # 车牌颜色: 3=蓝
    "carType": "1",               # 车型: 1=小型
    "type": 3,                    # 名单类型: 1=黑, 2=灰, 3=白
    "ownerName": "",
    "startDate": "2026-04-28T00:00:00",
    "endDate": "2026-04-30T23:59:59",
    "remark": ""
}

resp = requests.post(
    f"{base_url}/api/parkmanagement/BlackWhiteGrey",
    headers=headers,
    json=new_record,
    verify=False
)
```

### 查询名单

```python
params = {"pageIndex": 0, "pageSize": 10, "plate": "粤Sxxxx"}
resp = requests.get(
    f"{base_url}/api/parkmanagement/BlackWhiteGrey",
    headers=headers,
    params=params,
    verify=False
)
```

---

## 功能二：新增用户并开通月租车（完整流程）

### API调用顺序

```
1. 登录获取Token
2. 查询组织ID (GET /api/systemcenter/group)
3. 获取新用户编号 (GET /api/systemcenter/person/newId)
4. 创建用户 (POST /api/systemcenter/person)
5. 添加车牌凭证 (POST /api/systemcenter/credential)
6. 开通月租车 (POST /api/parkmanagement/pmsLeaseStall/openLeaseStall)
7. 同步本地表格 (可选)
```

### 完整代码示例

```python
import hashlib
import requests
import urllib3
from datetime import datetime

urllib3.disable_warnings()

base_url = "https://10.0.12.1:9091"
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# ========== 用户信息 ==========
name = "陈座娣"
plate = "粤BF7889"
phone = "13480980000"
unit = "卫生健康局"       # 组织名称关键字
seal_name = "地面月卡"    # 套餐名称
seal_id = "p220447822150" # 套餐ID
end_time = "2030-12-30T23:59:59"
remark = "借调"

# ========== 步骤1: 查询组织ID（分页遍历获取全部） ==========
all_groups = []
page = 0
while True:
    r = requests.get(f"{base_url}/api/systemcenter/group",
        headers=headers, params={"pageIndex": page, "pageSize": 100}, verify=False)
    rows = r.json()['data']['rows']
    all_groups.extend(rows)
    total = r.json()['data'].get('total', 0)
    if len(all_groups) >= total or not rows:
        break
    page += 1

# 匹配组织：精确匹配 → 用户输入包含在组织名中 → 组织名包含在用户输入中
group_id = None
for match_fn in [lambda g: g['name'] == unit, lambda g: unit in g['name'], lambda g: g['name'] in unit]:
    found = next((g for g in all_groups if match_fn(g)), None)
    if found:
        group_id = found['id']
        unit = found['name']  # 使用API返回的完整组织名
        break

# ========== 步骤2: 获取新用户编号 ==========
r = requests.get(f"{base_url}/api/systemcenter/person/newId", headers=headers, verify=False)
person_no = r.json()['data']

# ========== 步骤3: 创建用户 ==========
person_data = {
    "personNo": person_no,
    "name": name,
    "mobile": phone,
    "groupId": group_id,
    "groupName": unit,
    "type": 2,                          # 2=业主
    "gender": "F",                      # M/F
    "relationship": 0,
    "enterTime": datetime.now().strftime("%Y-%m-%dT00:00:00"),
    "status": 1,
    "remark": remark
}
r = requests.post(f"{base_url}/api/systemcenter/person", headers=headers, json=person_data, verify=False)
person_id = r.json()['data']['id']

# ========== 步骤4: 添加车牌凭证 ==========
# 车牌颜色: 3=蓝牌, 5=绿牌(新能源)
plate_color = 5 if (len(plate) == 8 and plate[:2] in ["粤B", "粤D", "粤F"]) else 3

credential_data = {
    "personId": person_id,
    "personNo": person_no,
    "personName": name,
    "credentialNo": plate,
    "credentialType": 163,              # 163=车牌号码
    "plate": plate,
    "plateColor": plate_color,
    "vehicleType": 0,                   # 0=小型车
    "status": 1
}
r = requests.post(f"{base_url}/api/systemcenter/credential", headers=headers, json=credential_data, verify=False)
credential_id = r.json()['data']['id']

# ========== 步骤5: 开通月租车 ==========
open_data = {
    "personId": person_id,
    "personNo": person_no,
    "personName": name,                  # 注意：用 personName
    "mobile": phone,
    "gender": "F",
    "groupId": group_id,
    "groupName": unit,
    "type": 2,
    "typeName": "业主",
    
    "sealId": seal_id,
    "sealName": seal_name,
    "userType": 1,
    
    "startTime": datetime.now().strftime("%Y-%m-%dT00:00:00"),
    "endTime": end_time,
    
    "delayMoney": 0,
    "payTypeID": "XJ",
    
    "carNumber": 1,
    "spaceNumbers": "1",
    "zyPlace": 1,                        # 租用车位数量
    "cqPlace": 0,
    "gxPlace": 0,
    "fjdPlace": 0,
    
    "spaceTypeInfoList": [{"spaceType": 2, "spaceNumber": 1}],
    
    "credentialNo": plate,
    "credentiallList": [{
        "credentialId": credential_id,
        "credentialNo": plate,
        "credentialType": 163,
        "plateColor": plate_color,
        "vechicleType": "1"
    }]
}

r = requests.post(f"{base_url}/api/parkmanagement/pmsLeaseStall/openLeaseStall",
    headers=headers, json=open_data, verify=False)
print("开通结果:", r.json())
```

### 套餐ID参考表

| 套餐名称 | sealId |
|----------|--------|
| 地面月卡 | p220447822150 |
| 地面临时车 | p220447822154 |
| 地库临时车 | p220447822155 |
| 地库月卡 | p220447822159 |
| 免费用户A | p220447822365 |
| 免费用户B | p220447822362 |

### 车牌颜色判断

```python
def get_plate_color(plate: str) -> int:
    """根据车牌号自动判断颜色"""
    if len(plate) == 8:  # 新能源车牌8位
        if plate[:2] in ["粤B", "粤D", "粤F"]:  # 深圳新能源常见
            return 5  # 绿牌
    return 3  # 默认蓝牌
```

---

## 功能三：车牌凭证管理

### 添加车牌凭证

```python
credential_data = {
    "personId": person_id,
    "personNo": person_no,
    "personName": "用户姓名",
    "credentialNo": "粤A11111",     # 车牌号
    "credentialType": 163,          # 163=车牌号码
    "plate": "粤A11111",
    "plateColor": 3,                # 3=蓝牌
    "vehicleType": 0,               # 0=小型车
    "status": 1
}

resp = requests.post(
    f"{base_url}/api/systemcenter/credential",
    headers=headers,
    json=credential_data,
    verify=False
)
credential_id = resp.json()['data']['id']
```

### 车牌颜色代码

| plateColor | 颜色 |
|------------|------|
| 1 | 黄牌 |
| 2 | 白牌 |
| 3 | 蓝牌 |
| 4 | 黑牌 |
| 5 | 绿牌 |

---

## 功能四：开通月租车

### 查询套餐列表

```python
resp = requests.get(
    f"{base_url}/api/parkmanagement/SetMealType",
    headers=headers,
    params={"pageIndex": 0, "pageSize": 20},
    verify=False
)
# 返回套餐列表，选择 sealId
```

### 开通月租车

```python
open_data = {
    "personId": person_id,
    "personNo": person_no,
    "personName": "用户姓名",       # 注意：用 personName 不是 name
    "mobile": "13800138000",
    "gender": "M",
    "groupId": "组织ID",
    "groupName": "组织名称",
    "type": 2,
    "typeName": "业主",
    
    "sealId": "套餐ID",
    "sealName": "套餐名称",
    "userType": 1,
    
    "startTime": datetime.now().strftime("%Y-%m-%dT00:00:00"),
    "endTime": "2030-12-30T23:59:59",
    
    "delayMoney": 0,
    "payTypeID": "XJ",
    
    "carNumber": 1,
    "spaceNumbers": "1",
    "zyPlace": 1,                    # 租用车位数量
    "cqPlace": 0,
    "gxPlace": 0,
    "fjdPlace": 0,
    
    "spaceTypeInfoList": [
        {"spaceType": 2, "spaceNumber": 1}  # spaceType=2 租用车位
    ],
    
    "credentialNo": "粤A11111",
    "credentiallList": [{
        "credentialId": credential_id,
        "credentialNo": "粤A11111",
        "credentialType": 163,
        "plateColor": 3,
        "vechicleType": "1"
    }]
}

resp = requests.post(
    f"{base_url}/api/parkmanagement/pmsLeaseStall/openLeaseStall",
    headers=headers,
    json=open_data,
    verify=False
)
```

---

## 功能五：查询月租车记录

```python
resp = requests.get(
    f"{base_url}/api/parkmanagement/pmsLeaseStall/pmsLeaseStallList",
    headers=headers,
    params={"pageIndex": 0, "pageSize": 10, "personName": "用户姓名"},
    verify=False
)
```

---

## 功能六：修改车牌信息

### 方法一：API自动修改（推荐）

通过 PUT `/api/parkmanagement/pmsLeaseStall/{leaseId}` 更新月租凭证。系统自动创建新车牌凭证 + 注销旧凭证。

```python
import hashlib, requests, urllib3
urllib3.disable_warnings()

base_url = "https://10.0.12.1:9091"
pw = hashlib.md5("88888888".encode()).hexdigest()
r = requests.post(f"{base_url}/api/systemcenter/auth/login",
    json={"account": "9990", "password": pw}, verify=False, timeout=10)
token = r.json()['data']['token']
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# 查月租
r1 = requests.get(f"{base_url}/api/parkmanagement/pmsLeaseStall/pmsLeaseStallList",
    headers=headers, params={"pageIndex": 0, "pageSize": 10, "personName": "用户姓名"}, verify=False, timeout=10)
row = r1.json()['data']['rows'][0]
lease_id = row['id']

# PUT更新（credentialId为空串则自动创建新凭证）
body = {k: row[k] for k in ["id","personNo","personName","mobile","sealId","sealName",
    "startTime","endTime","carNumber","spaceNumbers","spaceTypeInfoList"]}
body["userType"] = 1
body["status"] = 0
body["credentialNo"] = "粤Sxxxxx"  # 新车牌
body["credentiallList"] = [{"credentialId": "", "credentialNo": "粤Sxxxxx",
    "credentialType": 163, "plateColor": 3, "vechicleType": "1"}]

r2 = requests.put(f"{base_url}/api/parkmanagement/pmsLeaseStall/{lease_id}",
    headers=headers, json=body, verify=False, timeout=15)
```

### 方法二：前端界面操作

1. 登录系统 → 车行系统 → 车场服务管理 → 套餐服务管理
2. 通过用户姓名搜索找到目标用户
3. 点击操作列的"变更"按钮 → 选择"车牌变更"
4. 输入新车牌号码 → 确定保存

---

## 功能七：删除人员并注销车牌

### 删除规则（重要）

| 操作方式 | 对应系统模块 | 效果 |
|----------|-------------|------|
| **删除姓名** | 用户管理 → 搜索姓名 → 批量迁出 | 删除用户信息 **+** 同时删除关联车牌凭证 |
| **删除车牌** | 凭证管理 → 搜索车牌 → 注销 | **仅注销车牌凭证**，用户信息保留 |
| **按车牌完全删除** | 先凭证管理注销车牌 → 再用户管理批量迁出 | 车牌注销 + 用户信息删除（完整清理） |

### 判断逻辑

- **用户说「删除姓名：xxx」** → 直接去用户管理搜索姓名，执行批量迁出（会同时删除用户和车牌）
- **用户说「删除车牌：粤Sxxxxx」** → 先去凭证管理注销车牌，再去用户管理删除用户信息

### API流程

#### 方式一：按姓名删除（批量迁出）

```
1. 登录获取Token
2. 查询用户 (GET /api/systemcenter/person?name=姓名)
3. 批量迁出 (PATCH /api/systemcenter/person/moveOut)
   → 同时删除用户信息和关联车牌凭证
4. 同步删除本地Excel记录（可选）
```

```python
import hashlib, requests, urllib3
urllib3.disable_warnings()

base_url = "https://10.0.12.1:9091"
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# 查询用户
params = {"pageIndex": 0, "pageSize": 20, "name": "张一"}
resp = requests.get(f"{base_url}/api/systemcenter/person",
    headers=headers, params=params, verify=False)
persons = resp.json()['data']['rows']

# 批量迁出（同时删除用户 + 关联凭证）
person_ids = [p["id"] for p in persons]
resp = requests.patch(f"{base_url}/api/systemcenter/person/moveOut",
    headers=headers, json={"ids": person_ids, "isServicesCancelled": True}, verify=False)
```

#### 方式二：按车牌删除（先注销再迁出）

```
1. 登录获取Token
2. 查询凭证 (GET /api/systemcenter/credential?credentialNo=车牌号)
3. 注销凭证 (PATCH /api/systemcenter/credential) → 设置 status=4
4. 查询用户 (GET /api/systemcenter/person?name=车主姓名)
5. 批量迁出 (PATCH /api/systemcenter/person/moveOut)
6. 同步删除本地Excel记录（可选）
```

```python
# 步骤1: 查询凭证
params = {"pageIndex": 0, "pageSize": 20, "credentialNo": "粤S22222"}
resp = requests.get(f"{base_url}/api/systemcenter/credential",
    headers=headers, params=params, verify=False)
creds = resp.json()['data']['rows']

# 步骤2: 注销凭证（PATCH status=4，批量注销）
active_cred_ids = [c["id"] for c in creds if c.get("status") == 1]
if active_cred_ids:
    resp = requests.patch(f"{base_url}/api/systemcenter/credential",
        headers=headers, json={"ids": active_cred_ids, "status": 4}, verify=False)

# 步骤3: 获取车主姓名后查询用户
person_name = creds[0].get("personName", "")
params = {"pageIndex": 0, "pageSize": 20, "name": person_name}
resp = requests.get(f"{base_url}/api/systemcenter/person",
    headers=headers, params=params, verify=False)
persons = resp.json()['data']['rows']

# 步骤4: 批量迁出
person_ids = [p["id"] for p in persons]
resp = requests.patch(f"{base_url}/api/systemcenter/person/moveOut",
    headers=headers, json={"ids": person_ids, "isServicesCancelled": True}, verify=False)
```

### 命令行工具

```bash
# 按姓名删除（批量迁出，同时删除用户和车牌）
python parking_helper.py delete-name 张一

# 按车牌删除（先注销车牌，再批量迁出删除用户）
python parking_helper.py delete-plate 粤S22222
```

---

## 功能八：已有用户增加车牌

### 适用场景

用户已开通月租车，需要为同一用户**增加一个额外车牌**（保留原有车牌）。

**注意**：此功能是「增加」车牌，不是「修改」车牌。原车牌继续有效，新车牌同时生效。

### 实现方式：通过月租变更 API

月租变更 API（PUT `/api/parkmanagement/pmsLeaseStall/{leaseId}`）支持 `credentiallList` 数组，在列表中追加新车牌即可同时生效。

### API 流程

```
1. 登录获取 Token
2. 查询用户月租记录 (GET pmsLeaseStallList?personName=姓名)
3. 从月租记录中获取现有车牌列表 (credentiallList)
4. 创建新车牌凭证 (POST /api/systemcenter/credential)
5. 合并车牌列表（旧 + 新）
6. 更新月租记录 (PUT pmsLeaseStall/{leaseId})
   → carNumber 更新为车牌总数
   → credentialNo 更新为新车牌（主车牌）
   → credentiallList 包含全部车牌
```

### 完整代码示例

```python
import hashlib, requests, urllib3
from datetime import datetime

urllib3.disable_warnings()

base_url = "https://10.0.12.1:9091"
pw = hashlib.md5("88888888".encode()).hexdigest()
r = requests.post(f"{base_url}/api/systemcenter/auth/login",
    json={"account": "9990", "password": pw}, verify=False, timeout=10)
token = r.json()['data']['token']
headers = {"Authorization": f"Bearer {token}", "Content-Type": "application/json"}

# ========== 参数 ==========
user_name = "陈座娣"
new_plate = "粤B88888"

# ========== 步骤1: 查找月租记录 ==========
r = requests.get(f"{base_url}/api/parkmanagement/pmsLeaseStall/pmsLeaseStallList",
    headers=headers, params={"pageIndex": 0, "pageSize": 10, "personName": user_name},
    verify=False, timeout=10)
rows = r.json()['data']['rows']
if not rows:
    raise Exception(f"未找到 {user_name} 的月租记录")
lease = rows[0]

# 提取用户信息
person_id = lease['personId']
person_no = lease['personNo']
lease_id = lease['id']

# ========== 步骤2: 获取现有车牌列表 ==========
existing_creds = lease.get('credentiallList', [])
existing_plates = [c['credentialNo'] for c in existing_creds]
print(f"现有车牌: {existing_plates}")

# 检查是否重复
if new_plate in existing_plates:
    raise Exception(f"车牌 {new_plate} 已存在")

# ========== 步骤3: 创建新车牌凭证 ==========
def get_plate_color(plate):
    if len(plate) == 8 and plate[:2] in ["粤B", "粤D", "粤F"]:
        return 5  # 绿牌
    return 3  # 蓝牌

plate_color = get_plate_color(new_plate)
cred_data = {
    "personId": person_id,
    "personNo": person_no,
    "personName": user_name,
    "credentialNo": new_plate,
    "credentialType": 163,
    "plate": new_plate,
    "plateColor": plate_color,
    "vehicleType": 0,
    "status": 1
}
r = requests.post(f"{base_url}/api/systemcenter/credential",
    headers=headers, json=cred_data, verify=False, timeout=10)
new_cred_id = r.json()['data']['id']
print(f"新车牌凭证ID: {new_cred_id}")

# ========== 步骤4: 合并车牌列表（旧 + 新）==========
existing_creds.append({
    "credentialId": new_cred_id,
    "credentialNo": new_plate,
    "credentialType": 163,
    "plateColor": plate_color,
    "vechicleType": "1"
})

total_cars = len(existing_creds)
print(f"合并后车牌数: {total_cars}")

# ========== 步骤5: 更新月租记录 ==========
body = {
    "id": lease_id,
    "personNo": person_no,
    "personName": user_name,
    "mobile": lease['mobile'],
    "credentialNo": new_plate if total_cars == 1 else lease.get('credentialNo', new_plate),
    "credentiallList": existing_creds,
    "sealId": lease['sealId'],
    "sealName": lease['sealName'],
    "userType": 1,
    "startTime": lease['startTime'],
    "endTime": lease['endTime'],
    "carNumber": total_cars,
    "spaceNumbers": lease.get('spaceNumbers', '1'),
    "spaceTypeInfoList": lease.get('spaceTypeInfoList', [{"spaceType": 2, "spaceNumber": 1}]),
    "status": 0
}

r = requests.put(f"{base_url}/api/parkmanagement/pmsLeaseStall/{lease_id}",
    headers=headers, json=body, verify=False, timeout=15)

result = r.json()
if result.get('code') == 200:
    print(f"✅ 成功为 {user_name} 增加车牌 {new_plate}")
    print(f"   当前所有车牌: {[c['credentialNo'] for c in existing_creds]}")
else:
    print(f"❌ 失败: {result}")
```

### 命令行工具

```bash
# 为已有用户增加车牌
python parking_helper.py add-plate 陈座娣 粤B88888
```

---

## 本地表格同步

系统会自动将人员和车牌信息同步到本地Excel表格。

**表格路径**: `F:\ShareCache\智能化系统\门禁系统\2024政府年门禁、车牌审核汇总表.xlsx`

**同步规则**:
| 操作 | 是否同步到本地表格 |
|------|-------------------|
| 修改车牌信息 | ✅ 同步修改 |
| 新增人员和车牌 | ✅ 同步添加 |
| 删除人员（按姓名/按车牌） | ✅ 同步删除 |
| 添加黑/白/灰名单 | ❌ 不同步 |

**表格列结构**: 单位名称、类型、工作证编号、姓名、职务、车牌号码、联系电话、备注

### 同步代码示例

技能目录中提供了完整的助手脚本：`scripts/parking_helper.py`

使用方法：
```bash
# 修改车牌（自动同步API + 本地表格）
python parking_helper.py modify 张一 粤S22222

# 新增人员到本地表格
python parking_helper.py add 专班 张一 粤S22222 18099996666
```

---

## 注意事项

1. 系统使用自签名SSL证书，需设置 `verify=False`
2. 需要禁用SSL警告：`urllib3.disable_warnings()`
3. 开通月租车时必须使用 `personName` 字段（不是 `name`）
4. 车位数量参数（zyPlace等）必须与套餐配置匹配
5. 有效期格式：`YYYY-MM-DDTHH:MM:SS`
6. 操作完成后应验证结果
7. 本地表格需要 `openpyxl` 库：`pip install openpyxl`

---

## 使用示例

用户可能的请求格式：
- "帮我把粤S1234添加到白名单，有效期3天"
- "添加用户张三，手机138xxxxxxxx，组织专班，车牌粤A12345，有效期到2030-12-30"
- "查询粤S9012的名单状态"
- "修改用户朱xx的车牌为粤Sxxxx"
- "删除姓名：张三" → 用户管理批量迁出，同时删除用户和车牌
- "删除车牌：粤S22222" → 先注销车牌凭证，再批量迁出删除用户
- "给陈座娣再加一个粤B88888"或"为xxx增加一个车牌粤xxx" → 已有用户增加车牌

### 命令行工具示例

```bash
# 修改车牌（API + 本地表格同步）
python parking_helper.py modify 张一 粤S22222

# 新增人员（API创建用户+凭证+月租 + 本地表格同步）
python parking_helper.py add 卫生健康局 陈座娣 粤BF7889 13480980000 地面月卡 2030-12-30 借调

# 已有用户增加车牌（月租变更追加）
python parking_helper.py add-plate 陈座娣 粤B88888

# 按姓名删除（批量迁出，同时删除用户和关联车牌）
python parking_helper.py delete-name 张一

# 按车牌删除（先注销车牌凭证，再批量迁出删除用户）
python parking_helper.py delete-plate 粤S22222
```

## 日期计算

当用户说"有效期N天"时：
- 开始日期：当天 00:00:00
- 结束日期：当天 + N-1 天 23:59:59

格式：`YYYY-MM-DDTHH:MM:SS`

## 组织代码参考

组织列表通过 `/api/systemcenter/group` 获取，**API限制pageSize最大100**，必须分页遍历（pageSize=100）直至取完全部记录，否则可能遗漏组织导致"找不到组织"错误。

组织匹配逻辑（按优先级）：
1. 精确匹配：用户输入 == 组织名
2. 子串匹配：用户输入包含在组织名中（如"三联村委"匹配"三联村民委员会"）
3. 反向子串匹配：组织名包含在用户输入中
