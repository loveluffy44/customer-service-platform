# 客服管理平台 - 数据库设计文档

> 版本: v1.0  
> 日期: 2026-04-08  
> 表数量: 25张核心表

---

## 一、设计说明

### 1.1 核心原则
- **无客服工作台**: 用户通过第三方平台反馈，系统通过API/人工录入接入
- **无工单分配**: 不涉及自动分配机制，仅记录投诉和处理状态
- **退款分流**: 自动退款（系统对接）vs 手动退款（人工审批）
- **8大渠道**: 支持API接入、App上报、人工录入三种方式

### 1.2 命名规范
- 表名: 小写下划线，如 `user_info`, `complaint_record`
- 字段: 小写下划线，如 `user_id`, `create_time`
- 主键: 统一使用 `id` (BIGINT UNSIGNED AUTO_INCREMENT)
- 外键: 关联表名 + `_id`，如 `user_id`, `complaint_id`
- 时间字段: `create_time`, `update_time`, `delete_time` (软删除)

### 1.3 通用字段
每张表均包含以下基础字段:
```sql
create_time   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
update_time   DATETIME     NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
creator_id    BIGINT       DEFAULT 0 COMMENT '创建人ID',
updater_id    BIGINT       DEFAULT 0 COMMENT '更新人ID',
delete_time   DATETIME     DEFAULT NULL COMMENT '删除时间(软删除)',
is_deleted    TINYINT      NOT NULL DEFAULT 0 COMMENT '是否删除: 0-否 1-是'
```

---

## 二、用户中心 (5张表)

### 2.1 user_info - 用户基础信息表

存储用户核心身份信息，对接业务系统用户体系。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK, AUTO_INCREMENT | 主键ID |
| user_id | VARCHAR(64) | NOT NULL, UNIQUE | 业务用户ID |
| phone | VARCHAR(20) | INDEX | 手机号 |
| email | VARCHAR(100) | | 邮箱 |
| nickname | VARCHAR(100) | | 用户昵称 |
| real_name | VARCHAR(100) | | 真实姓名 |
| id_card | VARCHAR(20) | | 身份证号(加密存储) |
| register_time | DATETIME | | 注册时间 |
| register_source | VARCHAR(50) | | 注册来源 |
| user_status | TINYINT | NOT NULL DEFAULT 1 | 用户状态: 0-禁用 1-正常 2-注销 |
| user_level | TINYINT | DEFAULT 1 | 用户等级 |
| risk_level | TINYINT | DEFAULT 0 | 风险等级: 0-正常 1-低风险 2-中风险 3-高风险 |
| tags | JSON | | 用户标签JSON |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_user_id (user_id),
KEY idx_phone (phone),
KEY idx_register_time (register_time),
KEY idx_user_status (user_status),
KEY idx_risk_level (risk_level)
```

---

### 2.2 user_membership - 会员信息表

存储用户会员状态，支持多会员类型。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| membership_type | TINYINT | NOT NULL DEFAULT 1 | 会员类型: 1-普通 2-银卡 3-金卡 4-黑卡 |
| membership_status | TINYINT | NOT NULL DEFAULT 0 | 会员状态: 0-非会员 1-有效 2-过期 3-冻结 |
| start_time | DATETIME | | 会员开始时间 |
| end_time | DATETIME | | 会员结束时间 |
| source_order_id | VARCHAR(64) | | 来源订单ID |
| total_amount | DECIMAL(16,2) | DEFAULT 0 | 累计消费金额 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
KEY idx_user_id (user_id),
KEY idx_membership_status (membership_status),
KEY idx_end_time (end_time)
```

---

### 2.3 user_tag_relation - 用户标签关联表

用户与标签的多对多关系表。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| tag_code | VARCHAR(50) | NOT NULL | 标签编码 |
| tag_name | VARCHAR(100) | | 标签名称(冗余) |
| tag_source | TINYINT | DEFAULT 1 | 标签来源: 1-系统 2-人工 3-规则计算 |
| effective_time | DATETIME | | 生效时间 |
| expire_time | DATETIME | | 过期时间 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_user_tag (user_id, tag_code),
KEY idx_tag_code (tag_code),
KEY idx_expire_time (expire_time)
```

---

### 2.4 user_behavior_record - 用户行为记录表

记录用户关键行为，用于风控和投诉分析。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| behavior_type | VARCHAR(50) | NOT NULL, INDEX | 行为类型: LOGIN/ORDER/REFUND/COMPLAINT等 |
| behavior_desc | VARCHAR(500) | | 行为描述 |
| related_id | VARCHAR(64) | | 关联业务ID |
| device_info | VARCHAR(500) | | 设备信息 |
| ip_address | VARCHAR(50) | | IP地址 |
| behavior_time | DATETIME | NOT NULL, INDEX | 行为发生时间 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
KEY idx_user_behavior_time (user_id, behavior_time),
KEY idx_behavior_type (behavior_type),
KEY idx_behavior_time (behavior_time)
```

---

### 2.5 user_call_record - 用户电话记录表

400电话通话记录，用于投诉关联分析。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| user_id | VARCHAR(64) | INDEX | 业务用户ID(可能未识别) |
| phone_number | VARCHAR(20) | NOT NULL, INDEX | 来电号码 |
| call_id | VARCHAR(64) | UNIQUE | 通话唯一ID(呼叫中心) |
| call_type | TINYINT | DEFAULT 1 | 通话类型: 1-呼入 2-呼出 |
| call_status | TINYINT | | 通话状态: 1-接通 2-未接 3-留言 |
| start_time | DATETIME | NOT NULL | 通话开始时间 |
| end_time | DATETIME | | 通话结束时间 |
| duration | INT | DEFAULT 0 | 通话时长(秒) |
| recording_url | VARCHAR(500) | | 录音文件URL |
| satisfaction_score | TINYINT | | 满意度评分 |
| complaint_id | BIGINT | INDEX | 关联投诉ID |
| call_content | TEXT | | 通话内容摘要 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_call_id (call_id),
KEY idx_user_id (user_id),
KEY idx_phone_number (phone_number),
KEY idx_start_time (start_time),
KEY idx_complaint_id (complaint_id)
```

---

## 三、投诉中心 (3张表)

### 3.1 complaint_record - 投诉记录主表

核心投诉数据表，对接8大渠道。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID(投诉ID) |
| complaint_no | VARCHAR(64) | NOT NULL, UNIQUE | 投诉单号: CP+年月日+6位序号 |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| channel_type | TINYINT | NOT NULL, INDEX | 渠道类型: 1-支付宝官方 2-支付宝小程序 3-微信官方 4-App内 5-400电话 6-在线客服 7-意见反馈 8-监管 |
| channel_source_id | BIGINT | INDEX | 渠道来源记录ID |
| external_complaint_id | VARCHAR(128) | INDEX | 外部投诉ID(支付宝/微信等) |
| complaint_type | TINYINT | NOT NULL | 投诉类型: 1-退款 2-服务质量 3-产品问题 4-账户问题 5-其他 |
| complaint_status | TINYINT | NOT NULL DEFAULT 10 | 投诉状态: 10-待处理 20-处理中 30-已解决 40-已关闭 50-已驳回 |
| priority | TINYINT | DEFAULT 2 | 优先级: 1-紧急 2-普通 3-低 |
| refund_type | TINYINT | NOT NULL | 退款类型: 1-自动退款 2-手动退款 |
| refund_amount | DECIMAL(16,2) | DEFAULT 0 | 退款金额 |
| refund_status | TINYINT | DEFAULT 0 | 退款状态: 0-无 10-处理中 20-已退款 30-已拒绝 |
| order_id | VARCHAR(64) | INDEX | 关联订单ID |
| product_id | VARCHAR(64) | | 关联产品ID |
| complaint_title | VARCHAR(200) | | 投诉标题 |
| complaint_content | TEXT | | 投诉内容 |
| user_contact | VARCHAR(100) | | 用户联系方式 |
| attachments | JSON | | 附件列表JSON |
| handler_id | BIGINT | INDEX | 处理人ID |
| handler_name | VARCHAR(100) | | 处理人姓名(冗余) |
| handle_result | TEXT | | 处理结果说明 |
| handle_time | DATETIME | | 处理时间 |
| close_time | DATETIME | | 关闭时间 |
| close_reason | TINYINT | | 关闭原因: 1-已解决 2-用户取消 3-重复投诉 4-无效投诉 |
| sla_deadline | DATETIME | INDEX | SLA截止时间 |
| sla_status | TINYINT | DEFAULT 0 | SLA状态: 0-正常 1-预警 2-超时 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_complaint_no (complaint_no),
KEY idx_user_id (user_id),
KEY idx_channel_type (channel_type),
KEY idx_complaint_status (complaint_status),
KEY idx_refund_type (refund_type),
KEY idx_refund_status (refund_status),
KEY idx_order_id (order_id),
KEY idx_handler_id (handler_id),
KEY idx_create_time (create_time),
KEY idx_sla_deadline (sla_deadline),
KEY idx_external_id (external_complaint_id)
```

---

### 3.2 complaint_channel_source - 投诉渠道来源记录表

记录各渠道的原始数据，用于溯源和对账。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| complaint_id | BIGINT | INDEX | 关联投诉ID |
| channel_type | TINYINT | NOT NULL | 渠道类型(同complaint_record) |
| source_type | TINYINT | NOT NULL | 接入方式: 1-API 2-App上报 3-人工录入 |
| external_id | VARCHAR(128) | INDEX | 外部系统ID |
| external_url | VARCHAR(500) | | 外部链接(支付宝/微信投诉页) |
| raw_data | JSON | | 原始数据JSON |
| sync_time | DATETIME | | 同步时间 |
| sync_status | TINYINT | DEFAULT 1 | 同步状态: 1-成功 2-失败 3-重试中 |
| error_msg | VARCHAR(500) | | 同步错误信息 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
KEY idx_complaint_id (complaint_id),
KEY idx_channel_type (channel_type),
KEY idx_external_id (external_id),
KEY idx_sync_time (sync_time)
```

---

### 3.3 complaint_operation_log - 投诉操作日志表

记录投诉处理全流程操作。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| complaint_id | BIGINT | NOT NULL, INDEX | 投诉ID |
| operation_type | TINYINT | NOT NULL | 操作类型: 1-创建 2-分配 3-处理 4-退款 5-关闭 6-重开 7-备注 |
| operation_desc | VARCHAR(500) | | 操作描述 |
| before_status | TINYINT | | 操作前状态 |
| after_status | TINYINT | | 操作后状态 |
| operator_id | BIGINT | INDEX | 操作人ID |
| operator_name | VARCHAR(100) | | 操作人姓名 |
| operator_role | VARCHAR(50) | | 操作人角色 |
| operation_data | JSON | | 操作数据JSON |
| client_ip | VARCHAR(50) | | 操作IP |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
KEY idx_complaint_id (complaint_id),
KEY idx_operation_type (operation_type),
KEY idx_operator_id (operator_id),
KEY idx_create_time (create_time)
```

---

## 四、售后处理中心 (5张表)

### 4.1 refund_record - 退款记录表

记录所有退款申请和处理结果。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| refund_no | VARCHAR(64) | NOT NULL, UNIQUE | 退款单号: RF+年月日+6位序号 |
| complaint_id | BIGINT | NOT NULL, INDEX | 关联投诉ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| order_id | VARCHAR(64) | NOT NULL, INDEX | 关联订单ID |
| refund_type | TINYINT | NOT NULL | 退款类型: 1-自动 2-手动 |
| refund_channel | TINYINT | NOT NULL | 退款渠道: 1-原路返回 2-支付宝 3-微信 4-银行卡 |
| refund_amount | DECIMAL(16,2) | NOT NULL | 退款金额 |
| refund_reason | TINYINT | | 退款原因: 1-用户要求 2-产品问题 3-服务问题 4-其他 |
| refund_reason_desc | VARCHAR(500) | | 退款原因描述 |
| refund_status | TINYINT | NOT NULL DEFAULT 10 | 退款状态: 10-待处理 20-审批中 30-审批通过 40-审批拒绝 50-退款中 60-退款成功 70-退款失败 |
| apply_time | DATETIME | NOT NULL | 申请时间 |
| apply_operator_id | BIGINT | | 申请人ID |
| approve_time | DATETIME | | 审批时间 |
| approver_id | BIGINT | | 审批人ID |
| approve_remark | VARCHAR(500) | | 审批备注 |
| refund_time | DATETIME | | 实际退款时间 |
| third_refund_no | VARCHAR(128) | | 第三方支付退款单号 |
| fail_reason | VARCHAR(500) | | 失败原因 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_refund_no (refund_no),
KEY idx_complaint_id (complaint_id),
KEY idx_user_id (user_id),
KEY idx_order_id (order_id),
KEY idx_refund_status (refund_status),
KEY idx_refund_type (refund_type),
KEY idx_apply_time (apply_time),
KEY idx_third_refund_no (third_refund_no)
```

---

### 4.2 compensation_record - 补偿记录表

记录非退款类补偿（优惠券、积分、会员等）。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| compensation_no | VARCHAR(64) | NOT NULL, UNIQUE | 补偿单号: CM+年月日+6位序号 |
| complaint_id | BIGINT | NOT NULL, INDEX | 关联投诉ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| compensation_type | TINYINT | NOT NULL | 补偿类型: 1-优惠券 2-积分 3-会员延期 4-现金 5-其他 |
| compensation_value | VARCHAR(100) | | 补偿值(如优惠券码、积分数) |
| compensation_amount | DECIMAL(16,2) | DEFAULT 0 | 补偿金额(现金类) |
| compensation_status | TINYINT | NOT NULL DEFAULT 10 | 补偿状态: 10-待审批 20-审批通过 30-审批拒绝 40-已发放 50-发放失败 |
| apply_time | DATETIME | NOT NULL | 申请时间 |
| apply_operator_id | BIGINT | | 申请人ID |
| approve_time | DATETIME | | 审批时间 |
| approver_id | BIGINT | | 审批人ID |
| issue_time | DATETIME | | 发放时间 |
| issue_result | VARCHAR(500) | | 发放结果 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_compensation_no (compensation_no),
KEY idx_complaint_id (complaint_id),
KEY idx_user_id (user_id),
KEY idx_compensation_status (compensation_status),
KEY idx_apply_time (apply_time)
```

---

### 4.3 approval_record - 审批记录表

通用审批流程记录，支持退款、补偿等审批。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| approval_no | VARCHAR(64) | NOT NULL, UNIQUE | 审批单号: AP+年月日+6位序号 |
| business_type | TINYINT | NOT NULL | 业务类型: 1-退款 2-补偿 3-会员调整 |
| business_id | BIGINT | NOT NULL, INDEX | 业务单据ID |
| approval_type | TINYINT | NOT NULL | 审批类型: 1-自动 2-一级审批 3-二级审批 |
| current_level | TINYINT | DEFAULT 1 | 当前审批层级 |
| approval_status | TINYINT | NOT NULL DEFAULT 10 | 审批状态: 10-待审批 20-审批中 30-审批通过 40-审批拒绝 50-已撤回 |
| submitter_id | BIGINT | NOT NULL | 提交人ID |
| submitter_name | VARCHAR(100) | | 提交人姓名 |
| submit_time | DATETIME | NOT NULL | 提交时间 |
| submit_remark | VARCHAR(500) | | 提交备注 |
| approver_id | BIGINT | | 当前审批人ID |
| approver_name | VARCHAR(100) | | 当前审批人姓名 |
| approve_time | DATETIME | | 审批时间 |
| approve_result | TINYINT | | 审批结果: 1-通过 2-拒绝 |
| approve_remark | VARCHAR(500) | | 审批备注 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_approval_no (approval_no),
KEY idx_business (business_type, business_id),
KEY idx_approval_status (approval_status),
KEY idx_submitter_id (submitter_id),
KEY idx_approver_id (approver_id),
KEY idx_submit_time (submit_time)
```

---

### 4.4 approval_flow_detail - 审批流程明细表

记录多级审批的每一步详情。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| approval_id | BIGINT | NOT NULL, INDEX | 审批记录ID |
| level | TINYINT | NOT NULL | 审批层级 |
| approver_id | BIGINT | NOT NULL | 审批人ID |
| approver_name | VARCHAR(100) | | 审批人姓名 |
| approve_status | TINYINT | NOT NULL DEFAULT 10 | 审批状态: 10-待审批 20-已通过 30-已拒绝 |
| receive_time | DATETIME | | 收到审批时间 |
| approve_time | DATETIME | | 审批时间 |
| approve_result | TINYINT | | 审批结果: 1-通过 2-拒绝 |
| approve_remark | VARCHAR(500) | | 审批备注 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
KEY idx_approval_id (approval_id),
KEY idx_approver_id (approver_id)
```

---

### 4.5 membership_adjust_record - 会员调整记录表

记录会员补发、延期等调整。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| adjust_no | VARCHAR(64) | NOT NULL, UNIQUE | 调整单号: MA+年月日+6位序号 |
| complaint_id | BIGINT | INDEX | 关联投诉ID |
| user_id | VARCHAR(64) | NOT NULL, INDEX | 业务用户ID |
| adjust_type | TINYINT | NOT NULL | 调整类型: 1-补发会员 2-延期 3-升级 4-降级 |
| membership_type | TINYINT | | 目标会员类型 |
| extend_days | INT | | 延期的天数 |
| adjust_status | TINYINT | NOT NULL DEFAULT 10 | 调整状态: 10-待审批 20-审批通过 30-审批拒绝 40-已执行 |
| apply_operator_id | BIGINT | | 申请人ID |
| execute_time | DATETIME | | 执行时间 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_adjust_no (adjust_no),
KEY idx_complaint_id (complaint_id),
KEY idx_user_id (user_id),
KEY idx_adjust_status (adjust_status)
```

---

## 五、权限管理 (3张表)

### 5.1 cs_staff - 客服人员表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| staff_no | VARCHAR(50) | NOT NULL, UNIQUE | 工号 |
| staff_name | VARCHAR(100) | NOT NULL | 姓名 |
| login_name | VARCHAR(50) | NOT NULL, UNIQUE | 登录账号 |
| password | VARCHAR(256) | NOT NULL | 密码(加密) |
| phone | VARCHAR(20) | | 手机号 |
| email | VARCHAR(100) | | 邮箱 |
| department_id | BIGINT | INDEX | 部门ID |
| department_name | VARCHAR(100) | | 部门名称(冗余) |
| staff_status | TINYINT | NOT NULL DEFAULT 1 | 状态: 0-禁用 1-在职 2-离职 |
| staff_type | TINYINT | DEFAULT 1 | 人员类型: 1-普通客服 2-客服主管 3-管理员 |
| last_login_time | DATETIME | | 最后登录时间 |
| last_login_ip | VARCHAR(50) | | 最后登录IP |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_staff_no (staff_no),
UNIQUE KEY uk_login_name (login_name),
KEY idx_department_id (department_id),
KEY idx_staff_status (staff_status)
```

---

### 5.2 cs_role - 角色表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| role_code | VARCHAR(50) | NOT NULL, UNIQUE | 角色编码 |
| role_name | VARCHAR(100) | NOT NULL | 角色名称 |
| role_desc | VARCHAR(500) | | 角色描述 |
| role_status | TINYINT | NOT NULL DEFAULT 1 | 状态: 0-禁用 1-启用 |
| permissions | JSON | | 权限列表JSON |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_role_code (role_code),
KEY idx_role_status (role_status)
```

---

### 5.3 cs_staff_role_relation - 人员角色关联表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| staff_id | BIGINT | NOT NULL | 人员ID |
| role_id | BIGINT | NOT NULL | 角色ID |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_staff_role (staff_id, role_id),
KEY idx_role_id (role_id)
```

---

## 六、分析主题表 - ADS层 (5张表)

### 6.1 ads_complaint_theme - 投诉主题表

按日汇总投诉数据，用于经营分析。

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| stat_date | DATE | NOT NULL, INDEX | 统计日期 |
| channel_type | TINYINT | NOT NULL | 渠道类型 |
| complaint_type | TINYINT | NOT NULL | 投诉类型 |
| new_count | INT | DEFAULT 0 | 新增投诉数 |
| resolved_count | INT | DEFAULT 0 | 已解决数 |
| closed_count | INT | DEFAULT 0 | 已关闭数 |
| avg_handle_time | INT | DEFAULT 0 | 平均处理时长(分钟) |
| sla_pass_rate | DECIMAL(5,2) | DEFAULT 100.00 | SLA达标率 |
| refund_amount | DECIMAL(16,2) | DEFAULT 0 | 退款金额 |
| refund_count | INT | DEFAULT 0 | 退款笔数 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_stat_date_channel (stat_date, channel_type, complaint_type),
KEY idx_stat_date (stat_date)
```

---

### 6.2 ads_user_theme - 用户主题表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| stat_date | DATE | NOT NULL, INDEX | 统计日期 |
| user_id | VARCHAR(64) | NOT NULL | 用户ID |
| complaint_count | INT | DEFAULT 0 | 累计投诉次数 |
| refund_count | INT | DEFAULT 0 | 累计退款次数 |
| refund_amount | DECIMAL(16,2) | DEFAULT 0 | 累计退款金额 |
| last_complaint_time | DATETIME | | 最近投诉时间 |
| risk_level | TINYINT | DEFAULT 0 | 风险等级 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_stat_date_user (stat_date, user_id),
KEY idx_stat_date (stat_date),
KEY idx_user_id (user_id)
```

---

### 6.3 ads_channel_theme - 渠道主题表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| stat_date | DATE | NOT NULL, INDEX | 统计日期 |
| channel_type | TINYINT | NOT NULL | 渠道类型 |
| total_count | INT | DEFAULT 0 | 总投诉数 |
| auto_refund_count | INT | DEFAULT 0 | 自动退款数 |
| manual_refund_count | INT | DEFAULT 0 | 手动退款数 |
| api_count | INT | DEFAULT 0 | API接入数 |
| manual_input_count | INT | DEFAULT 0 | 人工录入数 |
| avg_response_time | INT | DEFAULT 0 | 平均响应时长(分钟) |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_stat_date_channel (stat_date, channel_type),
KEY idx_stat_date (stat_date)
```

---

### 6.4 ads_staff_performance_theme - 客服绩效主题表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| stat_date | DATE | NOT NULL, INDEX | 统计日期 |
| staff_id | BIGINT | NOT NULL | 人员ID |
| handle_count | INT | DEFAULT 0 | 处理投诉数 |
| resolved_count | INT | DEFAULT 0 | 解决数 |
| avg_handle_time | INT | DEFAULT 0 | 平均处理时长(分钟) |
| satisfaction_score | DECIMAL(3,2) | DEFAULT 5.00 | 满意度评分 |
| refund_amount | DECIMAL(16,2) | DEFAULT 0 | 处理退款金额 |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_stat_date_staff (stat_date, staff_id),
KEY idx_stat_date (stat_date),
KEY idx_staff_id (staff_id)
```

---

### 6.5 ads_refund_theme - 退款主题表

| 字段名 | 类型 | 约束 | 说明 |
|--------|------|------|------|
| id | BIGINT UNSIGNED | PK | 主键ID |
| stat_date | DATE | NOT NULL, INDEX | 统计日期 |
| refund_type | TINYINT | NOT NULL | 退款类型: 1-自动 2-手动 |
| refund_channel | TINYINT | | 退款渠道 |
| total_count | INT | DEFAULT 0 | 总退款笔数 |
| total_amount | DECIMAL(16,2) | DEFAULT 0 | 总退款金额 |
| success_count | INT | DEFAULT 0 | 成功笔数 |
| success_amount | DECIMAL(16,2) | DEFAULT 0 | 成功金额 |
| fail_count | INT | DEFAULT 0 | 失败笔数 |
| avg_approve_time | INT | DEFAULT 0 | 平均审批时长(分钟) |
| ext_info | JSON | | 扩展信息 |

**索引设计:**
```sql
PRIMARY KEY (id),
UNIQUE KEY uk_stat_date_type (stat_date, refund_type, refund_channel),
KEY idx_stat_date (stat_date)
```

---

## 七、表关系图

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│   user_info     │◄────┤ complaint_record │────►│ complaint_      │
│   (用户基础)     │     │   (投诉主表)      │     │ channel_source  │
└────────┬────────┘     └────────┬─────────┘     │ (渠道来源)       │
         │                       │               └─────────────────┘
         │                       │
         ▼                       ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ user_membership │     │ refund_record    │◄────┤ approval_record │
│   (会员信息)     │     │   (退款记录)      │     │   (审批记录)     │
└─────────────────┘     └────────┬─────────┘     └─────────────────┘
                                 │
         ┌───────────────────────┼───────────────────────┐
         ▼                       ▼                       ▼
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│compensation_    │     │membership_adjust_│     │ approval_flow_  │
│record (补偿)    │     │record (会员调整)  │     │ detail (审批明细)│
└─────────────────┘     └──────────────────┘     └─────────────────┘

┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│    cs_staff     │◄────┤cs_staff_role_    │────►│    cs_role      │
│   (客服人员)     │     │  relation        │     │   (角色)        │
└─────────────────┘     │ (人员角色关联)    │     └─────────────────┘
                        └──────────────────┘
```

---

## 八、核心业务流程时序

### 8.1 自动退款流程
```
1. 第三方平台投诉 ──► 2. API拉取创建complaint_record
                         (refund_type=1自动退款)
                              │
                              ▼
3. 系统自动创建refund_record ◄── 4. 调用支付接口退款
   (refund_status=60成功)         │
                              5. 更新complaint_record状态
                                 (complaint_status=30已解决)
```

### 8.2 手动退款流程
```
1. 投诉接入 ──► 2. 创建complaint_record
                    (refund_type=2手动退款)
                         │
                         ▼
3. 运营人员申请退款 ──► 4. 创建refund_record
   (refund_status=10待处理)   (refund_status=20审批中)
                                    │
                                    ▼
5. 创建approval_record ◄── 6. 主管审批
   (approval_status=20审批中)      │
                              7. 更新refund_record
                                 (refund_status=30审批通过)
                                    │
                                    ▼
8. 执行退款 ◄── 9. 更新complaint_record
   (refund_status=60成功)
```

---

## 九、数据量估算

| 表名 | 预估日增量 | 保留周期 | 预估总量 |
|------|-----------|---------|---------|
| complaint_record | 1000-5000条 | 3年 | 500万-1000万 |
| complaint_operation_log | 3000-15000条 | 1年 | 500万-1000万 |
| refund_record | 500-2000条 | 3年 | 200万-500万 |
| user_behavior_record | 10万-50万条 | 90天 | 按需清理 |
| ADS主题表 | 按日汇总 | 2年 | 数据量小 |

---

## 十、扩展建议

1. **分库分表**: complaint_record、refund_record 可按user_id取模分表
2. **冷热分离**: 历史数据(>1年)迁移至归档库
3. **读写分离**: 查询类接口走从库
4. **缓存策略**: user_info、user_membership 可缓存至Redis
5. **ES搜索**: 投诉内容全文检索使用Elasticsearch

