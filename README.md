# 58同城账户购买与自动化管理系统

## 项目概述

本项目提供了一个完整的58同城账户购买、管理及自动化操作解决方案。系统采用Python语言开发，结合Selenium、Requests等库实现自动化操作，并提供了完整的API接口供二次开发。

## 功能特性

- 58同城账户批量购买接口
- 账户自动登录与验证系统
- 信息自动发布模块
- 智能防封号策略
- 多账户轮换管理系统
- 数据统计与分析面板

## 技术架构

```
├── core/                # 核心功能模块
│   ├── account.py       # 账户管理
│   ├── automation.py    # 自动化操作
│   ├── anti_ban.py      # 防封策略
├── api/                 # API接口
│   ├── purchase.py      # 购买接口
│   ├── manage.py        # 管理接口
├── utils/               # 工具类
│   ├── logger.py        # 日志系统
│   ├── proxy.py         # 代理管理
├── config/              # 配置文件
│   ├── settings.py      # 主配置
│   ├── paths.py         # 路径配置
├── tests/               # 测试用例
├── docs/                # 文档
└── main.py              # 主入口
```

## 安装与配置

### 环境要求

- Python 3.8+
- Chrome浏览器(最新版)
- ChromeDriver(与浏览器版本匹配)

### 安装步骤

```bash
# 克隆仓库
git clone https://github.com/yourrepo/58account-manager.git
cd 58account-manager

# 创建虚拟环境
python -m venv venv
source venv/bin/activate  # Linux/Mac
venv\Scripts\activate     # Windows

# 安装依赖
pip install -r requirements.txt
```

### 配置文件

编辑`config/settings.py`进行基本配置：

```python
# 基本配置
BASE_CONFIG = {
    'platform': '58同城',
    'version': '2.5.0',
    'debug': False,
    'max_retry': 3,
}

# 账户配置
ACCOUNT_CONFIG = {
    'purchase_url': 'https://api.58.com/account/purchase',
    'api_key': 'your_api_key_here',
    'default_quantity': 10,
    'balance_warning': 100,
}

# 自动化配置
AUTOMATION_CONFIG = {
    'headless': False,
    'timeout': 30,
    'implicit_wait': 10,
    'page_load_timeout': 60,
}
```

## 核心功能实现

### 1. 账户购买模块

```python
import requests
import hashlib
import time
from urllib.parse import urlencode

class AccountPurchaser:
    def __init__(self, api_key, secret_key):
        self.api_key = api_key
        self.secret_key = secret_key
        self.base_url = "https://api.58.com/v3/account/purchase"
    
    def generate_sign(self, params):
        """生成API签名"""
        param_str = urlencode(sorted(params.items()))
        sign_str = f"{param_str}&secret_key={self.secret_key}"
        return hashlib.md5(sign_str.encode()).hexdigest().upper()
    
    def purchase_accounts(self, quantity, account_type='standard'):
        """批量购买58同城账户"""
        params = {
            'api_key': self.api_key,
            'timestamp': int(time.time()),
            'quantity': quantity,
            'type': account_type
        }
        
        params['sign'] = self.generate_sign(params)
        
        try:
            response = requests.post(self.base_url, data=params, timeout=30)
            result = response.json()
            
            if result['code'] == 200:
                return {
                    'success': True,
                    'order_id': result['data']['order_id'],
                    'accounts': result['data']['accounts']
                }
            else:
                return {
                    'success': False,
                    'error_code': result['code'],
                    'message': result['message']
                }
        except Exception as e:
            return {
                'success': False,
                'error_code': 'REQUEST_ERROR',
                'message': str(e)
            }
```

### 2. 自动化登录系统

```python
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from selenium.common.exceptions import TimeoutException

class AutoLoginSystem:
    def __init__(self, driver_path, proxy=None):
        self.options = webdriver.ChromeOptions()
        if proxy:
            self.options.add_argument(f'--proxy-server={proxy}')
        
        # 防止被识别为自动化工具
        self.options.add_argument("--disable-blink-features=AutomationControlled")
        self.options.add_experimental_option("excludeSwitches", ["enable-automation"])
        self.options.add_experimental_option('useAutomationExtension', False)
        
        self.driver = webdriver.Chrome(
            executable_path=driver_path,
            options=self.options
        )
    
    def login_58_account(self, username, password):
        """58同城账户自动登录"""
        try:
            self.driver.get("https://passport.58.com/login")
            
            # 等待并点击账号密码登录
            WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.XPATH, "//div[@class='login-tab']/span[2]"))
            ).click()
            
            # 输入用户名密码
            username_field = WebDriverWait(self.driver, 20).until(
                EC.presence_of_element_located((By.XPATH, "//input[@id='username']"))
            )
            username_field.send_keys(username)
            
            password_field = self.driver.find_element(By.XPATH, "//input[@id='password']")
            password_field.send_keys(password)
            
            # 点击登录按钮
            login_btn = self.driver.find_element(By.XPATH, "//button[@id='submit-btn']")
            login_btn.click()
            
            # 验证是否登录成功
            WebDriverWait(self.driver, 30).until(
                EC.presence_of_element_located((By.XPATH, "//div[contains(@class, 'login-success')]"))
            )
            return True
        except TimeoutException:
            # 检查是否有验证码
            if self._handle_captcha():
                return self.login_58_account(username, password)
            return False
        except Exception as e:
            print(f"登录过程中发生错误: {str(e)}")
            return False
    
    def _handle_captcha(self):
        """处理验证码"""
        try:
            captcha = WebDriverWait(self.driver, 10).until(
                EC.presence_of_element_located((By.XPATH, "//div[@class='captcha-container']"))
            )
            # 这里可以集成第三方验证码识别服务
            print("需要人工处理验证码")
            return False
        except:
            return False
```

### 3. 信息发布自动化

```python
class PostAutomation:
    def __init__(self, driver):
        self.driver = driver
    
    def post_info(self, title, content, category, images=None, price=None):
        """自动发布信息到58同城"""
        try:
            # 进入发布页面
            self.driver.get("https://post.58.com")
            
            # 选择分类
            self._select_category(category)
            
            # 填写基本信息
            self._fill_basic_info(title, content, price)
            
            # 上传图片
            if images:
                self._upload_images(images)
            
            # 提交发布
            submit_btn = WebDriverWait(self.driver, 20).until(
                EC.element_to_be_clickable((By.XPATH, "//button[@class='submit-btn']"))
            )
            submit_btn.click()
            
            # 验证发布成功
            WebDriverWait(self.driver, 30).until(
                EC.presence_of_element_located((By.XPATH, "//div[contains(text(), '发布成功')]"))
            )
            return True
        except Exception as e:
            print(f"发布过程中发生错误: {str(e)}")
            return False
    
    def _select_category(self, category_path):
        """选择分类"""
        categories = category_path.split('>')
        for i, category in enumerate(categories):
            xpath = f"//span[contains(text(), '{category.strip()}')]"
            WebDriverWait(self.driver, 10).until(
                EC.element_to_be_clickable((By.XPATH, xpath))
            ).click()
    
    def _fill_basic_info(self, title, content, price):
        """填写基本信息"""
        title_field = WebDriverWait(self.driver, 10).until(
            EC.presence_of_element_located((By.XPATH, "//input[@placeholder='请输入标题']"))
        )
        title_field.send_keys(title)
        
        content_field = self.driver.find_element(By.XPATH, "//textarea[@placeholder='请输入详细描述']")
        content_field.send_keys(content)
        
        if price:
            price_field = self.driver.find_element(By.XPATH, "//input[@placeholder='请输入价格']")
            price_field.send_keys(str(price))
    
    def _upload_images(self, image_paths):
        """上传图片"""
        upload_input = WebDriverWait(self.driver, 10).until(
            EC.presence_of_element_located((By.XPATH, "//input[@type='file']"))
        )
        
        for img_path in image_paths:
            upload_input.send_keys(img_path)
            time.sleep(1)  # 等待上传完成
```

## 高级功能实现

### 1. 智能防封策略

```python
import random
import time
from fake_useragent import UserAgent

class AntiBanSystem:
    def __init__(self):
        self.ua = UserAgent()
        self.last_operation_time = 0
        self.operation_count = 0
    
    def get_random_delay(self, min_delay=2, max_delay=10):
        """获取随机延迟时间"""
        return random.uniform(min_delay, max_delay)
    
    def human_like_delay(self):
        """模拟人类操作延迟"""
        current_time = time.time()
        if current_time - self.last_operation_time < 1:
            self.operation_count += 1
        else:
            self.operation_count = 0
        
        # 操作越频繁，延迟越长
        base_delay = self.operation_count * 0.5
        delay = self.get_random_delay(1 + base_delay, 3 + base_delay)
        time.sleep(delay)
        self.last_operation_time = time.time()
    
    def get_random_headers(self):
        """获取随机请求头"""
        return {
            'User-Agent': self.ua.random,
            'Accept': 'text/html,application/xhtml+xml,application/xml;q=0.9,image/webp,*/*;q=0.8',
            'Accept-Language': 'zh-CN,zh;q=0.8,zh-TW;q=0.7,zh-HK;q=0.5,en-US;q=0.3,en;q=0.2',
            'Connection': 'keep-alive',
            'Upgrade-Insecure-Requests': '1'
        }
    
    def random_mouse_movement(self, driver, element=None):
        """模拟随机鼠标移动"""
        if element:
            action = webdriver.ActionChains(driver)
            
            # 随机移动路径
            for _ in range(random.randint(2, 5)):
                x_offset = random.randint(-50, 50)
                y_offset = random.randint(-50, 50)
                action.move_by_offset(x_offset, y_offset).perform()
                time.sleep(random.uniform(0.1, 0.3))
            
            # 最终移动到目标元素
            action.move_to_element(element).perform()
            time.sleep(random.uniform(0.2, 0.5))
```

### 2. 多账户轮换系统

```python
import json
from queue import Queue
from threading import Lock

class AccountPool:
    def __init__(self, account_file='accounts.json'):
        self.account_file = account_file
        self.accounts = Queue()
        self.lock = Lock()
        self._load_accounts()
    
    def _load_accounts(self):
        """从文件加载账户"""
        try:
            with open(self.account_file, 'r', encoding='utf-8') as f:
                account_list = json.load(f)
                for account in account_list:
                    self.accounts.put(account)
        except FileNotFoundError:
            print(f"账户文件 {self.account_file} 不存在")
        except json.JSONDecodeError:
            print(f"账户文件 {self.account_file} 格式错误")
    
    def get_account(self):
        """获取一个可用账户"""
        with self.lock:
            if self.accounts.empty():
                return None
            account = self.accounts.get()
            # 将账户放回队列尾部实现轮换
            self.accounts.put(account)
            return account
    
    def add_account(self, username, password, cookies=None):
        """添加新账户"""
        with self.lock:
            new_account = {
                'username': username,
                'password': password,
                'cookies': cookies or {},
                'last_used': None,
                'usage_count': 0
            }
            self.accounts.put(new_account)
            self._save_accounts()
    
    def update_account(self, username, **kwargs):
        """更新账户信息"""
        with self.lock:
            temp_queue = Queue()
            found = False
            
            while not self.accounts.empty():
                account = self.accounts.get()
                if account['username'] == username:
                    account.update(kwargs)
                    found = True
                temp_queue.put(account)
            
            # 将账户重新放回主队列
            while not temp_queue.empty():
                self.accounts.put(temp_queue.get())
            
            if found:
                self._save_accounts()
            return found
    
    def _save_accounts(self):
        """保存账户到文件"""
        accounts = []
        temp_queue = Queue()
        
        while not self.accounts.empty():
            account = self.accounts.get()
            accounts.append(account)
            temp_queue.put(account)
        
        # 恢复队列
        while not temp_queue.empty():
            self.accounts.put(temp_queue.get())
        
        with open(self.account_file, 'w', encoding='utf-8') as f:
            json.dump(accounts, f, ensure_ascii=False, indent=2)
```

## API接口文档

### 账户购买接口

**Endpoint**: `/api/purchase`

**Method**: POST

**请求参数**:

| 参数名 | 类型 | 必填 | 描述 |
|-------|------|------|------|
| quantity | int | 是 | 购买数量 |
| account_type | string | 否 | 账户类型(standard/vip) |

**响应示例**:

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "order_id": "ORD20250620123456",
        "accounts": [
            {
                "username": "user123456",
                "password": "pass123456",
                "initial_cookies": {...}
            },
            ...
        ]
    }
}
```

### 账户管理接口

**Endpoint**: `/api/account/manage`

**Method**: POST

**请求参数**:

| 参数名 | 类型 | 必填 | 描述 |
|-------|------|------|------|
| action | string | 是 | 操作类型(list/get/update) |
| username | string | 否 | 要操作的用户名 |
| data | object | 否 | 更新数据 |

**响应示例**:

```json
{
    "code": 200,
    "message": "success",
    "data": {
        "accounts": [
            {
                "username": "user123456",
                "last_used": "2025-06-20T12:00:00",
                "usage_count": 5
            },
            ...
        ]
    }
}
```

## 部署方案

### Docker部署

```dockerfile
FROM python:3.8-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY . .

# 安装Chrome
RUN apt-get update && apt-get install -y wget gnupg \
    && wget -q -O - https://dl-ssl.google.com/linux/linux_signing_key.pub | apt-key add - \
    && echo "deb [arch=amd64] http://dl.google.com/linux/chrome/deb/ stable main" >> /etc/apt/sources.list.d/google.list \
    && apt-get update \
    && apt-get install -y google-chrome-stable \
    && rm -rf /var/lib/apt/lists/*

# 安装ChromeDriver
RUN CHROME_DRIVER_VERSION=`curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE` \
    && wget -O /tmp/chromedriver.zip https://chromedriver.storage.googleapis.com/$CHROME_DRIVER_VERSION/chromedriver_linux64.zip \
    && unzip /tmp/chromedriver.zip -d /usr/bin/ \
    && chmod +x /usr/bin/chromedriver \
    && rm /tmp/chromedriver.zip

CMD ["python", "main.py"]
```

### Kubernetes部署

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: 58account-manager
spec:
  replicas: 3
  selector:
    matchLabels:
      app: 58account-manager
  template:
    metadata:
      labels:
        app: 58account-manager
    spec:
      containers:
      - name: manager
        image: yourrepo/58account-manager:latest
        ports:
        - containerPort: 8000
        env:
        - name: API_KEY
          valueFrom:
            secretKeyRef:
              name: 58-secrets
              key: api-key
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: "0.5"
            memory: 512Mi
---
apiVersion: v1
kind: Service
metadata:
  name: 58account-service
spec:
  selector:
    app: 58account-manager
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8000
```

## 贡献指南

欢迎贡献代码！请遵循以下步骤：

1. Fork本项目
2. 创建您的特性分支 (`git checkout -b feature/AmazingFeature`)
3. 提交您的更改 (`git commit -m 'Add some AmazingFeature'`)
4. 推送到分支 (`git push origin feature/AmazingFeature`)
5. 提交Pull Request

## 许可证

本项目采用MIT许可证 - 详情请参阅[LICENSE](LICENSE)文件

## 联系方式

如有任何问题或建议，请联系：
- 邮箱: support@58accountmanager.com
- 问题跟踪: [GitHub Issues](https://github.com/yourrepo/58account-manager/issues)
