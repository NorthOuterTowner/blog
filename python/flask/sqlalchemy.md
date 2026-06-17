# SQLAlchemy

## 介绍SQLAlchemy

SQLAlchemy是一款在Python中十分常用的ORM工具，作为一款ORM，他是我认知中极少数的不剥夺用户使用原生SQL权利的ORM工具。在我之前的很多项目中，如MyBatis等ORM，以及像Django直接集成了ORM，对于刚开始上手项目，不熟悉ORM自己设计的方法的人，我认为极不友好，因此有时宁愿放弃ORM提供的些许便捷性，不如直接写原生SQL，尤其是对于可以自己主导的项目来说，只需要确保自己的SQL参数化查询，不被SQL注入攻击等等，就可以简单的使用connector来完成数据库操作，因此有时ORM反倒常常引入更加繁琐的内容，但SQL alchemy是我了解到的第一个仍然可以让用户使用SQL的ORM，因此我对其颇有好感。

作为一款ORM工具，他具备基础的功能，如将Python映射到SQL，以及进行对象关系映射（也就是ORM）。

其核心组件主要包括以下几个：
- Engine：负责管理数据库连接和方言转换
- Declarative Base：ORM 的起点，所有的模型类都要继承它。
- Session：你的“工作区”，对对象的操作都先在会话中进行，最后统一 commit 到数据库。（所以这里很容易知道一道常问的面试问题的答案——“进行数据库操作时，在会话中完成操作，但是没有commit的情况下，会发生什么”）
Model：代表数据库表的 Python 类。

防止SQL注入不必多说，但SQLalchemy提供的强大的关系管理提供了很便捷的一对多or多对多查询（当然要编码n+1查询，个人认为在n+1查询的情况下不妨直接手写join SQL）。

## 快速上手：Flask + SQLAlchemy + MySQL

为了保证项目的可扩展性和维护性，我们将采用工厂模式，并将数据库配置、模型定义和路由逻辑分离到不同的文件中。本示例使用 MySQL 作为底层数据库。

### 1. 项目目录结构

这种结构可以有效避免代码耦合和循环导入问题：

```text
my_project/
├── app.py
├── config.py
├── extensions.py
├── models.py
└── routes.py
```

### 2. 环境依赖

在使用 MySQL 之前，除了安装 Flask 和 Flask-SQLAlchemy，还需要安装 MySQL 的 Python 驱动：

```bash
pip install flask flask-sqlalchemy pymysql
```

### 3. 代码实现

#### 配置文件 (`config.py`)

我们将数据库的连接信息独立出来。这里使用 `mysql+pymysql` 作为连接协议，同时关闭追踪修改功能以节省系统资源。

```python
class Config:
    SQLALCHEMY_DATABASE_URI = 'mysql+pymysql://root:your_password@localhost:3306/library_db'
    SQLALCHEMY_TRACK_MODIFICATIONS = False
```

#### 扩展初始化 (`extensions.py`)

创建一个独立的扩展文件，只负责实例化 SQLAlchemy 对象，这是解决循环导入的关键。

```python
from flask_sqlalchemy import SQLAlchemy

db = SQLAlchemy()
```

#### 数据模型与关系 (`models.py`)

在此处定义表结构。`Author` 和 `Book` 构成了一对多的关系。通过 `ForeignKey` 建立物理层面约束，通过 `relationship` 建立逻辑层面联系，并设置级联删除。
db.Model 作为参数是 SQLAlchemy 的声明式基类，它提供了 ORM 功能的基础，因此需要db.Model作为基类。
定义表结构时，最好显示的声明`__tablename__`，否则会以类名的小写寻找表，以及各个列最好与变量一致，或者需要指定`name`参数，如：
`id = db.Column("id", primary_key = True), autoincrement = True`

以下是`db.Column`提供的全部参数：
| 参数 | 类型 | 说明 | 示例 |
|------|------|------|------|
| `type_` | 类型类 | 列的数据类型 | `db.String(50)`, `db.Integer`, `db.DateTime` |
| `primary_key` | bool | 是否为主键 | `primary_key=True` |
| `nullable` | bool | 是否允许NULL值 | `nullable=False` |
| `default` | 值或函数 | 列的默认值 | `default='active'`, `default=datetime.now` |
| `server_default` | str | 数据库端的默认值 | `server_default='CURRENT_TIMESTAMP'` |
| `unique` | bool | 是否唯一约束 | `unique=True` |
| `index` | bool | 是否创建索引 | `index=True` |
| `autoincrement` | bool/str | 是否自动增长 | `autoincrement=True` |
| `name` | str | 数据库中的列名 | `name='user_id'` |
| `nullable` | bool | 是否允许为空 | `nullable=True` |
| `comment` | str | 列的注释 | `comment='用户年龄'` |
| `key` | str | 字典访问时的键名 | `key='userAge'` |
| `info` | dict | 附加信息字典 | `info={'description': '...'}` |
| `onupdate` | 值或函数 | 更新时自动调用 | `onupdate=datetime.now` |
| `server_onupdate` | str | 数据库端更新触发器 | `server_onupdate=FetchedValue()` |
| `foreign_key` | str | 外键约束 | `db.ForeignKey('users.id')` |
| `unique_constraint` | list | 复合唯一约束 | 通常通过 `UniqueConstraint` 类定义 |
| `check` | str | 检查约束 | `db.CheckConstraint('age >= 0')` |
| `doc` | str | 列文档字符串 | `doc='用户邮箱地址'` |
| `quote` | bool | 是否添加引号 | `quote=True` |
| `deferred` | bool | 是否延迟加载 | `deferred=True` |
| `group` | str | 分组名称 | `group='basic_info'` |

```python
from extensions import db
from datetime import datetime

class Author(db.Model):
    __tablename__ = 'authors'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(50), nullable=False)
    books = db.relationship('Book', backref='author', lazy=True, cascade="all, delete-orphan")

class Book(db.Model):
    __tablename__ = 'books'
    id = db.Column(db.Integer, primary_key=True)
    title = db.Column(db.String(100), nullable=False)
    pub_date = db.Column(db.DateTime, default=datetime.utcnow)
    author_id = db.Column(db.Integer, db.ForeignKey('authors.id'), nullable=False)
```

#### 路由与业务逻辑 (`routes.py`)

使用 Flask 的蓝图来管理 API 路由。展示了如何添加关联数据、如何进行关联查询，以及如何使用 `text()` 执行原生 SQL 语句。

```python
from flask import Blueprint, request, jsonify
from extensions import db
from models import Author, Book
from sqlalchemy import text

api_bp = Blueprint('api', __name__)

@api_bp.route('/authors', methods=['POST'])
def create_author():
    data = request.get_json()
    new_author = Author(name=data['name'])
    db.session.add(new_author)
    db.session.commit()
    return jsonify({"message": "Author created", "id": new_author.id}), 201

@api_bp.route('/authors/<int:author_id>/books', methods=['POST'])
def add_book(author_id):
    data = request.get_json()
    author = Author.query.get_or_404(author_id)
    
    new_book = Book(title=data['title'], author=author)
    db.session.add(new_book)
    db.session.commit()
    return jsonify({"message": f"Book added to {author.name}"}), 201

@api_bp.route('/authors/<int:author_id>', methods=['GET'])
def get_author(author_id):
    author = Author.query.get_or_404(author_id)
    books = [{"id": b.id, "title": b.title} for b in author.books]
    
    return jsonify({
        "id": author.id,
        "name": author.name,
        "books": books
    })

@api_bp.route('/stats', methods=['GET'])
def get_stats():
    sql = text("SELECT count(*) as count FROM books")
    result = db.session.execute(sql).fetchone()
    return jsonify({"total_books": result.count})
```

#### 程序入口 (`app.py`)

采用应用工厂模式。负责创建 Flask 实例、加载配置、初始化扩展对象并注册蓝图。在应用上下文内部调用 `create_all()` 自动创建数据库表。

```python
from flask import Flask
from config import Config
from extensions import db
from routes import api_bp
import pymysql

def create_app():
    # 创建数据库（如果不存在）
    conn = pymysql.connect(host='localhost', user='root', password='root')
    conn.cursor().execute("CREATE DATABASE IF NOT EXISTS library_db")
    conn.commit()
    conn.close()
    
    app = Flask(__name__)
    app.config.from_object(Config)
    
    db.init_app(app)
    app.register_blueprint(api_bp)
    
    with app.app_context():
        db.create_all()
        
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```