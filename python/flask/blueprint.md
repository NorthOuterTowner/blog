# Blueprint

`Blueprint`是一种在`Flask`下组织API的形式，通过这种形式来完成API的模块化。如果读者熟悉`Express`的话，这种组织方式与`Express`中的`router`其实是非常相似的。

通过定义`Blueprint`，将API加入到`Blueprint`上，再在app中使用`Blueprint`，即可完成路由的组织。

## 定义子路由

这里首先设置一个`user_db`的mock数据，供用户进行调用。
```py
users_db = {
    "1": {"name": "张三", "role": "Admin"},
    "2": {"name": "李四", "role": "User"}
}
```
在`user/routes.py`中定义Blueprint，将Blueprint的名称和导入模块的名称作为参数，其中导入模块的名称只需要通过`__name__`即可获得。
之后用注解的形式定义路由的内容，即：
```py
from flask import Blueprint, jsonify, request
user_bp = Blueprint('user_bp', __name__)

@user_bp.route('/search/<user_id>', methods=['GET'])
def get_user(user_id):
    user = users_db.get(user_id)
    if user:
        return jsonify({"status": "success", "data": user}), 200
    else:
        return jsonify({"status": "error", "message": "用户不存在"}), 404
```

这里需要注意，不同于其他大多数框架如`express`或`springboot`直接提供`get()`或者`post()`一类的方法，在`flask`的旧版本中必须要通过`@Blueprint.route()`的方法定义路由并在`methods`中指定支持的方法。
当然，如果使用的是`flask 2.0 +`的版本，则`flask`也提供了直接的方法来简化语法。

## 注册Blueprint
导入`user_bp`，在
```py
# app/user/routes.py
from flask import Flask
from user.routes import user_bp

def create_app():
    app = Flask(__name__)

    app.register_blueprint(user_bp, url_prefix='/user')

    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True)
```