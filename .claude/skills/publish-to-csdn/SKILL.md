---
name: publish-to-csdn
description: 将 Markdown 文章发布到 CSDN 博客，支持自动上传图片和设置标签
argument-hint: "<markdown文件路径>"
---

# 发布 Markdown 到 CSDN

将本地 Markdown 文件发布到 CSDN 博客平台。

## 前置条件

### 1. 安装依赖

```bash
pip install selenium webdriver-manager pyperclip markdown
```

### 2. 安装 Chrome 浏览器

确保系统已安装 Chrome 浏览器。

### 3. 获取 CSDN Cookie

1. 登录 CSDN (https://www.csdn.net)
2. F12 打开开发者工具 → Application → Cookies
3. 复制所有 cookie 值，保存到 `~/.csdn_cookie.json`

```json
{
  "cookie": "你的完整cookie字符串"
}
```

## 执行步骤

### 1. 读取 Markdown 文件

读取用户指定的 markdown 文件内容。

### 2. 生成发布脚本

在项目目录创建 `csdn_publisher.py`:

```python
import os
import sys
import json
import time
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.chrome.service import Service
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
from webdriver_manager.chrome import ChromeDriverManager
import pyperclip

def load_cookie():
    cookie_path = os.path.expanduser("~/.csdn_cookie.json")
    if os.path.exists(cookie_path):
        with open(cookie_path, 'r') as f:
            return json.load(f).get('cookie', '')
    return None

def publish_to_csdn(md_file, title=None):
    # 读取文件
    with open(md_file, 'r', encoding='utf-8') as f:
        content = f.read()

    # 提取标题（如果未指定）
    if not title:
        lines = content.split('\n')
        for line in lines:
            if line.startswith('# '):
                title = line[2:].strip()
                break
        if not title:
            title = os.path.basename(md_file).replace('.md', '')

    # 配置浏览器
    options = Options()
    options.add_argument('--start-maximized')
    # options.add_argument('--headless')  # 无头模式，取消注释可后台运行

    driver = webdriver.Chrome(
        service=Service(ChromeDriverManager().install()),
        options=options
    )

    try:
        # 打开 CSDN 编辑器
        driver.get('https://editor.csdn.net/md/')
        time.sleep(2)

        # 注入 cookie
        cookie = load_cookie()
        if cookie:
            for item in cookie.split(';'):
                item = item.strip()
                if '=' in item:
                    name, value = item.split('=', 1)
                    driver.add_cookie({'name': name.strip(), 'value': value.strip()})
            driver.refresh()
            time.sleep(2)

        # 等待编辑器加载
        wait = WebDriverWait(driver, 20)

        # 输入标题
        title_input = wait.until(
            EC.presence_of_element_located((By.CLASS_NAME, 'article-bar__title'))
        )
        title_input.clear()
        title_input.send_keys(title)

        # 输入内容（使用剪贴板）
        pyperclip.copy(content)
        editor = wait.until(
            EC.presence_of_element_located((By.CLASS_NAME, 'editor'))
        )
        editor.click()
        editor.send_keys(Keys.CONTROL, 'a')
        editor.send_keys(Keys.CONTROL, 'v')

        time.sleep(2)

        # 点击发布按钮
        publish_btn = wait.until(
            EC.element_to_be_clickable((By.CLASS_NAME, 'btn-publish'))
        )
        publish_btn.click()

        time.sleep(2)

        # 选择文章类型（原创）
        try:
            original_radio = driver.find_element(By.XPATH, "//label[contains(text(),'原创')]")
            original_radio.click()
        except:
            pass

        # 最终发布
        try:
            final_publish = wait.until(
                EC.element_to_be_clickable((By.XPATH, "//button[contains(text(),'发布文章')]"))
            )
            final_publish.click()
            print(f"文章 '{title}' 发布成功!")
        except Exception as e:
            print(f"请手动完成发布: {e}")
            input("按回车键退出...")

        time.sleep(3)

    finally:
        driver.quit()

if __name__ == '__main__':
    if len(sys.argv) < 2:
        print("用法: python csdn_publisher.py <markdown文件>")
        sys.exit(1)

    md_file = sys.argv[1]
    title = sys.argv[2] if len(sys.argv) > 2 else None
    publish_to_csdn(md_file, title)
```

### 3. 执行发布

```bash
python csdn_publisher.py "文章.md"
```

## 使用方式

### 方式一：直接调用 Skill

```
/publish-to-csdn blog/我的文章.md
```

### 方式二：使用已有工具

推荐使用开源工具 [blog-auto-publishing-tools](https://github.com/ddean2009/blog-auto-publishing-tools):

```bash
git clone https://github.com/ddean2009/blog-auto-publishing-tools.git
cd blog-auto-publishing-tools
pip install -r requirements.txt
python main.py --platform csdn --file "文章.md"
```

## 注意事项

1. **Cookie 有效期**：CSDN cookie 会过期，需要定期更新
2. **图片处理**：CSDN 会自动转存文章中的图片
3. **反爬限制**：频繁发布可能触发验证码
4. **手动确认**：建议发布前预览，确认格式正确

## 常见问题

### Q: 提示未登录？
A: 更新 `~/.csdn_cookie.json` 中的 cookie

### Q: 编辑器加载失败？
A: 检查网络，或增加 `time.sleep()` 等待时间

### Q: 图片上传失败？
A: CSDN 会自动转存外链图片，本地图片需要先上传到图床
