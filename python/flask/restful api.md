
# 通过 Flask 实现 GET 和 POST 的不同形式 RESTful API

*这篇博客有Gemini完成*

在现代 Web 开发中，**RESTful API** 是前后端分离架构的核心。Flask 作为一个轻量级、灵活的 Python Web 框架，提供了非常直观的方式来处理 HTTP 请求。本文将详细介绍如何使用 Flask 接收不同形式的 GET 和 POST 数据。

---

## 1. 环境准备

首先，确保你的环境中安装了 Flask：

```bash
pip install flask
```

创建一个基础的 `app.py` 结构：

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

# 后续代码将在此处添加

if __name__ == '__main__':
    app.run(debug=True)
```

---

## 2. GET 请求：获取与查询

GET 请求通常用于从服务器检索数据。在 Flask 中，数据通常通过 **路径参数** 或 **查询字符串** 传递。

### 2.1 路径参数 (Path Parameters)
这种形式常用于定位特定资源，例如通过 ID 获取用户信息。

```python
@app.route('/api/users/<int:user_id>', methods=['GET'])
def get_user(user_id):
    # 模拟从数据库获取数据
    user_data = {
        "id": user_id,
        "name": "李瑞泽",
        "profession": "Software Engineer"
    }
    return jsonify(user_data), 200
```
* **调用示例**：`GET /api/users/1024`

### 2.2 查询字符串 (Query Strings)
用于过滤、排序或分页。通过 `request.args` 获取。

```python
@app.route('/api/search', methods=['GET'])
def search():
    keyword = request.args.get('q', '')
    page = request.args.get('page', default=1, type=int)
    
    return jsonify({
        "query": keyword,
        "page": page,
        "results": ["Result 1", "Result 2"]
    }), 200
```
* **调用示例**：`GET /api/search?q=flask&page=2`

---

## 3. POST 请求：提交与创建

POST 请求用于向服务器发送数据以创建新资源。数据通常存储在 **请求体 (Request Body)** 中。

### 3.1 接收 JSON 数据 (推荐)
这是目前移动端和单页应用（Vue/React）最常用的方式。

```python
@app.route('/api/users', methods=['POST'])
def create_user():
    # 自动解析 JSON 负载
    data = request.get_json()
    
    if not data or 'username' not in data:
        return jsonify({"error": "Bad Request", "message": "Missing username"}), 400
        
    username = data.get('username')
    email = data.get('email')
    
    return jsonify({
        "status": "success",
        "created_user": {"username": username, "email": email}
    }), 201
```

### 3.2 接收表单数据 (Form Data)
常用于传统的 HTML 表单提交。

```python
@app.route('/api/login', methods=['POST'])
def login():
    username = request.form.get('username')
    password = request.form.get('password')
    
    # 业务逻辑处理...
    return jsonify({"status": "logged in", "user": username}), 200
```

---

## 4. 核心组件对比

为了方便记忆，我们可以通过下表对比这几种数据获取方式：

| 获取方式 | Flask 调用 | 适用场景 |
| :--- | :--- | :--- |
| **路径参数** | 路由定义中的 `<var>` | 定位唯一资源 (ID) |
| **查询字符串** | `request.args.get()` | 搜索、过滤、分页 |
| **JSON 数据** | `request.get_json()` | 复杂结构数据、前后端分离 |
| **表单数据** | `request.form.get()` | HTML 表单、简单键值对 |

---

## 5. 总结

Flask 的 `request` 对象是处理 API 输入的万能钥匙。在设计 RESTful API 时：
* 使用 **GET** 来“读”数据。
* 使用 **POST** 来“写”数据。
* 始终返回 **JSON** 格式响应，并配合正确的 **HTTP 状态码**（如 200 OK, 201 Created, 400 Bad Request）。
