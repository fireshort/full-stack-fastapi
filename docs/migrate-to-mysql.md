# 从 PostgreSQL 迁移到 MySQL

本文档记录了将 Full Stack FastAPI 项目从 PostgreSQL 迁移到 MySQL 的步骤和注意事项。

## 1. 依赖更改

在 `backend/pyproject.toml` 中：

1. 移除 PostgreSQL 相关依赖：

```toml
- "psycopg[binary]>=3.1.12,<4.0"
```

2. 添加 MySQL 相关依赖：

```toml
"pymysql[rsa]>=1.0.2",
"cryptography>=41.0.0",
```

## 2. 环境变量配置

在 `.env` 文件中：

1. 移除 PostgreSQL 配置：

```env
POSTGRES_SERVER=db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=changethis
POSTGRES_DB=app
```

2. 添加 MySQL 配置：

```env
MYSQL_SERVER=localhost
MYSQL_PORT=3306
MYSQL_DB=full_stack_fastapi
MYSQL_USER=root
MYSQL_PASSWORD=your_password
```

## 3. 数据库配置

在 `backend/app/core/config.py` 中：

1. 修改导入：

```python
from pydantic import MySQLDsn  # 替换 PostgresDsn
```

2. 修改配置类：

```python
class Settings(BaseSettings):
    # 移除 PostgreSQL 配置
    # POSTGRES_SERVER: str
    # POSTGRES_USER: str
    # POSTGRES_PASSWORD: str
    # POSTGRES_DB: str

    # 添加 MySQL 配置
    MYSQL_SERVER: str
    MYSQL_PORT: int = 3306
    MYSQL_USER: str
    MYSQL_PASSWORD: str = ""
    MYSQL_DB: str = ""

    @computed_field
    @property
    def SQLALCHEMY_DATABASE_URI(self) -> MySQLDsn:
        return MultiHostUrl.build(
            scheme="mysql+pymysql",
            username=self.MYSQL_USER,
            password=self.MYSQL_PASSWORD,
            host=self.MYSQL_SERVER,
            port=self.MYSQL_PORT,
            path=self.MYSQL_DB,
        )
```

## 4. 模型修改

在 `backend/app/models.py` 中：

1. 使用 VARCHAR(36) 存储 UUID：
   - MySQL 不支持原生 UUID 类型
   - 使用 VARCHAR(36) 存储 UUID
   - 在字段定义中明确指定数据库类型和配置

2. 在模型中定义 UUID 字段：
```python
class User(UserBase, table=True):
    id: uuid.UUID = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        sa_type=String(36),
    )
    hashed_password: str
    items: list["Item"] = Relationship(back_populates="owner", cascade_delete=True)

class Item(ItemBase, table=True):
    id: uuid.UUID = Field(
        default_factory=uuid.uuid4,
        primary_key=True,
        sa_type=String(36),
    )
    title: str = Field(max_length=255)
    owner_id: uuid.UUID = Field(
        sa_type=String(36),
        foreign_key="user.id",
        nullable=False,
        ondelete="CASCADE"
    )
    owner: User | None = Relationship(back_populates="items")
```

这种方式的优点：
- 明确定义每个字段的数据库类型和配置
- 确保主键和外键的类型一致
- 代码清晰直观，易于理解和维护
- 避免不必要的抽象

## 5. Docker 配置

在 `docker-compose.yml` 中：

1. 修改数据库服务：
```yaml
services:
  db:
    image: mysql:8.0
    restart: always
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u${MYSQL_USER}", "-p${MYSQL_PASSWORD}"]
      interval: 10s
      retries: 5
      start_period: 30s
      timeout: 10s
    volumes:
      - app-db-data:/var/lib/mysql
    env_file:
      - .env
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_PASSWORD?Variable not set}
      - MYSQL_DATABASE=${MYSQL_DB?Variable not set}
      - MYSQL_USER=${MYSQL_USER?Variable not set}
      - MYSQL_PASSWORD=${MYSQL_PASSWORD?Variable not set}
```

## 6. 数据迁移

1. 删除旧的迁移文件：
```bash
rm -rf backend/app/alembic/versions/*
```

2. 删除数据库中的迁移记录：
```sql
DROP TABLE IF EXISTS alembic_version;
```

3. 重新创建数据库（确保使用正确的字符集）：
```sql
DROP DATABASE IF EXISTS full_stack_fastapi;
CREATE DATABASE full_stack_fastapi CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

4. 生成新的迁移文件：
```bash
cd backend
python -m alembic revision --autogenerate -m "initialize_mysql_models"
```

5. 应用迁移：
```bash
python -m alembic upgrade head
```

6. 创建初始数据：
```bash
python app/initial_data.py
```

## 7. 注意事项

1. 字符集和排序规则
   - 使用 `utf8mb4` 字符集和 `utf8mb4_unicode_ci` 排序规则
   - 这样可以支持完整的 Unicode 字符集，包括表情符号

2. UUID 字段处理
   - MySQL 不支持原生 UUID 类型
   - 使用 VARCHAR(36) 存储 UUID
   - 需要在模型中正确配置 sa_type

3. 主键和外键
   - 确保主键和外键使用相同的数据类型和长度
   - 明确指定 ondelete 行为（如 CASCADE）

4. 字段长度限制
   - 合理设置 VARCHAR 字段的长度
   - 常用长度如 255 适用于大多数场景

5. 迁移问题处理
   - 如果迁移失败，检查 alembic_version 表
   - 必要时删除旧的迁移记录和文件重新开始

## 8. 验证

完成迁移后，需要验证以下功能：

1. 用户认证和授权
2. 数据 CRUD 操作
3. 外键关联和级联删除
4. 索引查询性能
5. 事务完整性

## 9. 回滚计划

如果迁移过程中出现问题，准备以下回滚步骤：

1. 备份 MySQL 数据
2. 恢复 PostgreSQL 配置
3. 重新应用原有迁移文件
4. 恢复数据

## 10. 参考资料

- [SQLModel 文档](https://sqlmodel.tiangolo.com/)
- [MySQL 文档](https://dev.mysql.com/doc/)
- [FastAPI 文档](https://fastapi.tiangolo.com/)
- [Alembic 文档](https://alembic.sqlalchemy.org/) 