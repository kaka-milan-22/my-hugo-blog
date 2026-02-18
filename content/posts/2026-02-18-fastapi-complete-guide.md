---
title: "FastAPI 完整指南：从入门到 Docker 部署"
date: 2026-02-18T17:58:00+08:00
draft: false
tags: ["FastAPI", "Python", "WebDevelopment", "Docker", "RESTAPI", "CRUD", "JWT"]
categories: ["Python", "DevOps"]
author: "Kaka"
description: "一份完整的 FastAPI 教程，涵盖核心概念、CRUD 操作、MySQL 数据库集成、JWT 认证授权，以及 Docker 容器化部署。适合想要快速掌握现代 Python Web API 开发的开发者。"
---

## 引言

在 Python Web 开发领域，FastAPI 正以其卓越的性能和开发体验迅速成为首选框架。作为一个现代、快速的 Web 框架，FastAPI 不仅提供了自动化的 API 文档生成，还具备类型提示、异步支持等特性，让开发者能够高效地构建生产级别的 API 服务。

本文基于一个完整的 5 小时实战教程，将带你从零开始掌握 FastAPI 的核心概念，并通过实际项目演练 CRUD 操作、数据库集成、用户认证以及最终的 Docker 容器化部署。

## FastAPI 核心概念

### 为什么选择 FastAPI？

FastAPI 相比传统框架（如 Flask、Django）有显著优势：

- **高性能**：基于 Starlette 和 Pydantic，性能与 Node.js 和 Go 相当
- **自动文档**：内置 Swagger UI 和 ReDoc，无需额外配置
- **类型提示**：利用 Python 类型注解进行数据验证和序列化
- **异步支持**：原生支持 async/await，适合高并发场景
- **开发效率**：减少 40% 的人为错误，提升开发速度

### 环境搭建

首先创建虚拟环境并安装依赖：

```bash
# 创建虚拟环境
python -m venv venv

# 激活虚拟环境 (Linux/macOS)
source venv/bin/activate

# 激活虚拟环境 (Windows)
venv\Scripts\activate

# 安装 FastAPI 和 Uvicorn
pip install fastapi uvicorn[standard]
```

### 第一个 FastAPI 应用

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello World"}

@app.get("/items/{item_id}")
async def read_item(item_id: int):
    return {"item_id": item_id}
```

运行应用：

```bash
uvicorn main:app --reload
```

访问 `http://127.0.0.1:8000/docs` 即可看到自动生成的 Swagger UI 文档。

## HTTP 方法与路由

### GET 请求

GET 请求用于获取资源数据：

```python
from typing import Optional
from pydantic import BaseModel

class Item(BaseModel):
    id: int
    name: str
    description: Optional[str] = None
    price: float
    tax: Optional[float] = None

# 模拟数据库
items_db = {}

@app.get("/items/", response_model=list[Item])
async def get_items():
    return list(items_db.values())

@app.get("/items/{item_id}", response_model=Item)
async def get_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    return items_db[item_id]
```

### POST 请求

POST 请求用于创建新资源：

```python
from fastapi import HTTPException, status

@app.post("/items/", response_model=Item, status_code=status.HTTP_201_CREATED)
async def create_item(item: Item):
    if item.id in items_db:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail="Item already exists"
        )
    items_db[item.id] = item
    return item
```

## 完整 CRUD 操作

### PUT 请求 - 更新资源

```python
@app.put("/items/{item_id}", response_model=Item)
async def update_item(item_id: int, item: Item):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    
    item.id = item_id  # 确保 ID 一致
    items_db[item_id] = item
    return item
```

### DELETE 请求 - 删除资源

```python
@app.delete("/items/{item_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_item(item_id: int):
    if item_id not in items_db:
        raise HTTPException(status_code=404, detail="Item not found")
    
    del items_db[item_id]
    return None
```

## SQLAlchemy 数据库集成

### 数据库配置

使用 SQLAlchemy 进行异步数据库操作：

```python
# database.py
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import declarative_base, sessionmaker

DATABASE_URL = "mysql+aiomysql://user:password@localhost/dbname"

engine = create_async_engine(DATABASE_URL, echo=True)
async_session = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)

Base = declarative_base()

async def get_db():
    async with async_session() as session:
        try:
            yield session
        finally:
            await session.close()
```

### 定义模型

```python
# models.py
from sqlalchemy import Column, Integer, String, Float
from database import Base

class Item(Base):
    __tablename__ = "items"
    
    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(100), nullable=False)
    description = Column(String(500))
    price = Column(Float, nullable=False)
    tax = Column(Float)
```

### 异步 CRUD 操作

```python
# crud.py
from sqlalchemy.ext.asyncio import AsyncSession
from sqlalchemy import select
from models import Item
from schemas import ItemCreate, ItemUpdate

async def create_item(db: AsyncSession, item: ItemCreate):
    db_item = Item(**item.dict())
    db.add(db_item)
    await db.commit()
    await db.refresh(db_item)
    return db_item

async def get_items(db: AsyncSession, skip: int = 0, limit: int = 100):
    result = await db.execute(select(Item).offset(skip).limit(limit))
    return result.scalars().all()

async def get_item(db: AsyncSession, item_id: int):
    result = await db.execute(select(Item).where(Item.id == item_id))
    return result.scalar_one_or_none()

async def update_item(db: AsyncSession, item_id: int, item: ItemUpdate):
    db_item = await get_item(db, item_id)
    if db_item:
        for key, value in item.dict(exclude_unset=True).items():
            setattr(db_item, key, value)
        await db.commit()
        await db.refresh(db_item)
    return db_item

async def delete_item(db: AsyncSession, item_id: int):
    db_item = await get_item(db, item_id)
    if db_item:
        await db.delete(db_item)
        await db.commit()
    return db_item
```

## JWT 认证与授权

### 用户模型与密码哈希

```python
# auth.py
from datetime import datetime, timedelta
from typing import Optional
from jose import JWTError, jwt
from passlib.context import CryptContext
from fastapi import Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer

SECRET_KEY = "your-secret-key-keep-it-secret"
ALGORITHM = "HS256"
ACCESS_TOKEN_EXPIRE_MINUTES = 30

pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="token")

def verify_password(plain_password: str, hashed_password: str) -> bool:
    return pwd_context.verify(plain_password, hashed_password)

def get_password_hash(password: str) -> str:
    return pwd_context.hash(password)

def create_access_token(data: dict, expires_delta: Optional[timedelta] = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=15))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

### 登录与令牌生成

```python
from pydantic import BaseModel
from fastapi.security import OAuth2PasswordRequestForm

class Token(BaseModel):
    access_token: str
    token_type: str

@app.post("/token", response_model=Token)
async def login(
    form_data: OAuth2PasswordRequestForm = Depends(),
    db: AsyncSession = Depends(get_db)
):
    user = await authenticate_user(db, form_data.username, form_data.password)
    if not user:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token = create_access_token(
        data={"sub": user.username},
        expires_delta=timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES)
    )
    return {"access_token": access_token, "token_type": "bearer"}
```

### 依赖注入获取当前用户

```python
async def get_current_user(
    token: str = Depends(oauth2_scheme),
    db: AsyncSession = Depends(get_db)
):
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username: str = payload.get("sub")
        if username is None:
            raise credentials_exception
    except JWTError:
        raise credentials_exception
    
    user = await get_user_by_username(db, username)
    if user is None:
        raise credentials_exception
    return user

# 使用认证保护路由
@app.get("/users/me")
async def read_users_me(current_user: User = Depends(get_current_user)):
    return current_user
```

## Docker 容器化部署

### Dockerfile

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# 安装系统依赖
RUN apt-get update && apt-get install -y \
    gcc \
    default-libmysqlclient-dev \
    pkg-config \
    && rm -rf /var/lib/apt/lists/*

# 复制依赖文件
COPY requirements.txt .

# 安装 Python 依赖
RUN pip install --no-cache-dir -r requirements.txt

# 复制应用代码
COPY . .

# 暴露端口
EXPOSE 8000

# 启动命令
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Docker Compose

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=mysql+aiomysql://root:password@db:3306/fastapi_db
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
    volumes:
      - .:/app
    
  db:
    image: mysql:8.0
    environment:
      - MYSQL_ROOT_PASSWORD=password
      - MYSQL_DATABASE=fastapi_db
    ports:
      - "3306:3306"
    volumes:
      - mysql_data:/var/lib/mysql

volumes:
  mysql_data:
```

### 部署命令

```bash
# 构建镜像
docker-compose build

# 启动服务
docker-compose up -d

# 查看日志
docker-compose logs -f

# 停止服务
docker-compose down
```

## 最佳实践

### 1. 项目结构

```
fastapi-project/
├── app/
│   ├── __init__.py
│   ├── main.py
│   ├── models.py
│   ├── schemas.py
│   ├── crud.py
│   ├── database.py
│   ├── auth.py
│   └── routers/
│       ├── __init__.py
│       ├── items.py
│       └── users.py
├── tests/
├── requirements.txt
├── Dockerfile
└── docker-compose.yml
```

### 2. 配置管理

使用 Pydantic Settings 管理环境变量：

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    app_name: str = "FastAPI Application"
    database_url: str
    secret_key: str
    access_token_expire_minutes: int = 30
    
    class Config:
        env_file = ".env"

settings = Settings()
```

### 3. 错误处理

```python
from fastapi import Request
from fastapi.responses import JSONResponse

@app.exception_handler(HTTPException)
async def http_exception_handler(request: Request, exc: HTTPException):
    return JSONResponse(
        status_code=exc.status_code,
        content={"error": exc.detail}
    )
```

### 4. 日志记录

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s"
)

logger = logging.getLogger(__name__)

@app.middleware("http")
async def log_requests(request: Request, call_next):
    logger.info(f"Request: {request.method} {request.url}")
    response = await call_next(request)
    logger.info(f"Response status: {response.status_code}")
    return response
```

## 总结

FastAPI 凭借其现代化的设计理念，为 Python Web 开发带来了全新的体验。通过本文的学习，你应该掌握了：

- **FastAPI 核心概念**：路由、请求处理、响应模型
- **完整的 CRUD 操作**：创建、读取、更新、删除
- **数据库集成**：使用 SQLAlchemy 进行异步数据库操作
- **用户认证**：JWT 令牌认证和授权机制
- **容器化部署**：使用 Docker 和 Docker Compose 部署应用

FastAPI 的学习曲线平缓，但要真正掌握其精髓，还需要在实际项目中不断实践。建议从简单的 API 开始，逐步添加数据库、认证等功能，最终构建完整的微服务架构。

## 参考资料

- [FastAPI 官方文档](https://fastapi.tiangolo.com/)
- [SQLAlchemy 文档](https://docs.sqlalchemy.org/)
- [JWT.io - JSON Web Tokens](https://jwt.io/)
- [Docker 官方文档](https://docs.docker.com/)
- [视频教程 GitHub 仓库](https://bit.ly/3MWnTcZ)
