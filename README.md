# org-management-miniprogram

请帮我开发一个微信小程序前端项目，用于"多租户机构管理系统"。后端 API 已开发完成并部署在腾讯云服务器上。

## 一、项目概述
- 项目名称：机构管理微信小程序
- 技术栈：微信小程序原生开发（WXML + WXSS + JS）
- 后端基础地址：https://你的域名（或服务器IP）  （开发阶段可先用本地调试）
- 统一响应格式：{"code": 200, "data": {...}, "msg": ""}
- 认证方式：JWT Token，登录成功后存储到本地，后续请求通过 Header: Authorization: Bearer <token> 携带

## 二、业务流程
1. 管理员首次使用：调用 wx.login() → 拿 code 调 /api/login → 如果未注册（404）→ 联系超级管理员添加
2. 管理员已注册：登录 → 创建机构 → 创建班级 → 添加教师（录入教师的 openid）→ 教师可以登录了
3. 教师使用：登录 → 绑定班级 → 使用系统

## 三、需要开发的页面清单

### 页面1：登录页 pages/login/login
- 进入小程序自动调用 wx.login() 获取 code
- 发送 POST /api/login {code: "xxx"}
- 成功：保存 token 和用户信息（openid, org_id, role），跳转到首页
- 失败（404）：提示"请联系管理员添加"
- 失败其他：显示错误信息

### 页面2：管理员 - 首页/仪表盘 pages/admin/index
- 显示当前机构信息（调用 GET /api/org/info）
- 快捷入口：班级管理、教师管理、创建新机构
- 根据 role 判断是否显示管理入口（role=admin 才显示管理功能）

### 页面3：管理员 - 机构管理 pages/admin/org-manage
- 创建机构表单：org_name（输入框） → POST /api/org {org_name}
- 查看/编辑当前机构信息 → GET /api/org/info

### 页面4：管理员 - 班级管理 pages/admin/class-manage
- 创建班级表单：class_name + teacher_name → POST /api/class {class_name, teacher_name}
- 班级列表展示 → GET /api/class/list
- 列表项显示：class_name, teacher_name, created_at

### 页面5：管理员 - 教师管理 pages/admin/teacher-manage
- 添加教师表单：openid + phone + role(选择器) → POST /api/admin/teacher/add {openid, phone, role}
- 教师列表 → GET /api/admin/teacher/list
- 列表项显示：openid, phone, role, class_name(关联), created_at

### 页面6：教师 - 个人中心 pages/teacher/profile
- 显示个人信息 → GET /api/teacher/info
- 编辑手机号 → PUT /api/teacher/profile {phone}

### 页面7：教师 - 绑定班级 pages/teacher/bind-class
- 先获取可用班级列表 → GET /api/class/list
- 选择班级 → PUT /api/teacher/bind-class {class_id}

## 四、完整的后端 API 接口定义

### 基础信息
- 所有接口前缀：/api
- 认证接口（除 /login 外所有接口都需要 Authorization header）

### 1. 微信登录
POST /api/login
请求体：{ "code": "微信wx.login返回的code" }
成功响应(200)：{ "code": 200, "data": {"token": "jwt-token", "openid": "xxx", "org_id": 1, "role": "admin"}, "msg": "登录成功" }
未注册(404)：{ "code": 404, "msg": "您还未被添加到系统中，请联系机构管理员", "data": {"openid": "xxx"} }

### 2. 创建机构（需token）
POST /api/org
Header: Authorization: Bearer <token>
请求体：{ "org_name": "机构名称" }
响应：{ "code": 200, "data": {"id": 1, "org_name": "xxx"}, "msg": "机构创建成功" }

### 3. 获取机构信息（需token）
GET /api/org/info
Header: Authorization: Bearer <token>
响应：{ "code": 200, "data": {"id": 1, "org_name": "xxx", "status": 1, "created_at": "..."}, "msg": "" }

### 4. 创建班级（需token）
POST /api/class
Header: Authorization: Bearer <token>
请求体：{ "class_name": "一年级一班", "teacher_name": "张老师" }  （teacher_name可选）
响应：{ "code": 200, "data": {"id": 1}, "msg": "班级创建成功" }

### 5. 获取班级列表（需token）
GET /api/class/list
Header: Authorization: Bearer <token>
响应：{ "code": 200, "data": {"total": 10, "list": [{"id": 1, "org_id": 1, "class_name": "xxx", "teacher_name": "xxx", "created_at": "..."}]}, "msg": "" }

### 6. 获取当前教师信息（需token）
GET /api/teacher/info
Header: Authorization: Bearer <token>
响应：{ "code": 200, "data": {"id": 1, "openid": "xxx", "org_id": 1, "class_id": null, "phone": "", "role": "teacher", "created_at": "..."}, "msg": "" }

### 7. 教师绑定班级（需token）
PUT /api/teacher/bind-class
Header: Authorization: Bearer <token>
请求体：{ "class_id": 1 }
响应：{ "code": 200, "msg": "已绑定到班级 ID=1" }

### 8. 更新教师资料（需token）
PUT /api/teacher/profile
Header: Authorization: Bearer <token>
请求体：{ "phone": "13800138000" } （phone和class_id都是可选，至少传一个）
响应：{ "code": 200, "msg": "资料更新成功" }

### 9. 管理员添加教师（需token）
POST /api/admin/teacher/add
Header: Authorization: Bearer <token>
请求体：{ "openid": "教师的微信openid", "phone": "13800138000", "role": "teacher" } （role为"admin"或"teacher"，默认teacher）
响应：{ "code": 200, "data": {"id": 1, "openid": "xxx"}, "msg": "教师已添加（role=teacher）" }
重复添加(409)：{ "code": 409, "msg": "该微信用户已被添加" }

### 10. 管理员查看教师列表（需token）
GET /api/admin/teacher/list
Header: Authorization: Bearer <token>
响应：{ "code": 200, "data": {"total": 5, "list": [{"id":1,"openid":"xxx","org_id":1,"class_id":1,"phone":"138xxx","role":"teacher","created_at":"...","class_name":"一年级一班"}]}, "msg": "" }

## 五、项目文件结构要求
miniprogram/ ├── app.js # 小程序入口，含全局数据和登录检查逻辑 ├── app.json # 小程序配置（页面路由、tabBar等） ├── app.wxss # 全局样式 ├── project.config.json # 项目配置文件 ├── sitemap.json # 站点地图 ├── utils/ │ ├── api.js # 封装 wx.request，统一处理 baseURL、token注入、错误处理 │ ├── auth.js # 登录、token存储/读取/清除 │ └── config.js # 配置文件（API基础URL等） ├── pages/ │ ├── login/ # 登录页 │ │ ├── login.wxml │ │ ├── login.wxss │ │ ├── login.js │ │ └── login.json │ ├── admin/ # 管理员模块 │ │ ├── index/ # 管理员首页/仪表盘 │ │ ├── org-manage/ # 机构管理页 │ │ ├── class-manage/ # 班级管理页 │ │ └── teacher-manage/# 教师管理页（添加+列表） │ └── teacher/ # 教师模块 │ ├── profile/ # 个人中心 │ └── bind-class/ # 绑定班级页 └── components/ # 可复用组件（如有需要）


## 六、技术要点
1. 使用 wx.request 封装统一的请求方法，自动携带 token，统一处理错误码
2. token 存储在 wx.setStorageSync 中，key 为 'token'
3. 登录态检查：app.js 的 onLaunch 中检查 token 是否存在且未过期，不存在则跳转登录页
4. 角色判断：根据登录返回的 role 字段决定显示管理员页面还是教师页面
5. UI 风格：简洁专业，使用微信官方设计规范，色调以蓝色为主
6. 表单验证：必填项校验、长度校验等
7. 加载状态：请求时显示 loading，完成后隐藏
8. 错误提示：使用 wx.showToast 显示操作结果

请生成以上所有文件的完整代码。

