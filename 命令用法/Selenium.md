## 安装

### 安装 chrome 浏览器，如果已经安装，查看版本

```bash
# linux 一键安装
sudo curl https://intoli.com/install-google-chrome.sh | bash
sudo mv /usr/bin/google-chrome-stable /usr/bin/google-chrome
```

### 安装 chromedriver，控制chrome的驱动

```bash
version=`google-chrome --version | awk '{print $3}'`
#current version is 103.0.5060.53
#last_version=`curl https://chromedriver.storage.googleapis.com/LATEST_RELEASE`
#for linux，for mac is chromedriver_mac64.zip，for mac m1 is chromedriver_mac64_m1.zip
wget "https://chromedriver.storage.googleapis.com/${version}/chromedriver_linux64.zip"
unzip chromedriver_linux64.zip
sudo mv chromedriver /usr/bin/chromedriver
chromedriver --version
rm -f chromedriver_linux64.zip
```

### 安装 selenium

```bash
pip3 install selenium
```

### 测试

```bash
#注意，linux需要进入显示模式，chrome才可以启动
screen
#如果ssh还不行，需要 -X 带x11连接，且 /etc/ssh/sshd_config 需要 X11Forwarding yes，修改完需要重启sshd

# 测试代码
cat>test.py<<EOF
from selenium.webdriver.chrome.options import Options
from selenium import webdriver
from selenium.webdriver.common.by import By
options = Options()
options.add_argument("--headless")
options.add_argument("window-size=1920,1080")
options.add_argument("--disable-gpu")
options.add_argument("--no-sandbox")
options.add_argument("start-maximized")
options.add_argument("enable-automation")
options.add_argument("--disable-infobars")
options.add_argument("--disable-dev-shm-usage")
driver = webdriver.Chrome(options=options)
driver.get("https://aws.amazon.com/")
text = driver.find_element(By.CLASS_NAME,'lb-btn-p-primary').text
print(text)
EOF

#这个测试会返回aws首页上的创建账号按钮的文本：如：“创建 AWS 账户”
python3 test.py
```

## 浏览器启动参数

```python
from selenium.webdriver.chrome.options import Options
from selenium import webdriver

options = Options()
if not args.debug:
    options.add_argument("--headless")
    options.add_argument("window-size=1920,1080")
    options.add_argument("--disable-gpu")
    options.add_argument("start-maximized")
    options.add_argument("--disable-infobars")
    options.add_argument("--disable-dev-shm-usage")
options.add_argument("--no-sandbox")
options.add_argument("enable-automation")
# chrome 权限允许，1:allow, 2:block
options.add_experimental_option("prefs", {
    "profile.default_content_setting_values.media_stream_mic": 1,
    "profile.default_content_setting_values.media_stream_camera": 1,
    "profile.default_content_setting_values.geolocation": 1,
    "profile.default_content_setting_values.notifications": 1
})

driver = webdriver.Chrome(options=options)
```
## 页面访问，元素定位

```python
from selenium.webdriver.common.by import By
from selenium.webdriver.support.wait import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

#页面加载等待最多 20 秒
driver.implicitly_wait(20)
driver.get('https://xxx.com/')
try:
    WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.ID,'xxxid')))
    WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.NAME,'xxxname')))
    WebDriverWait(driver, 20).until(EC.presence_of_element_located((By.CLASS_NAME,'xxxclassname')))
    driver.find_element(By.ID,'xxxid').send_keys('123456')
    driver.find_element(By.ID,'xxxbutton').click()
except Exception as e:
    print(e)
    driver.close()
driver.quit()
```

## 常用代码

```python
#执行脚本
driver.execute_script('return arguments[0].innerText', elem)

#屏幕截图
driver.save_screenshot('./image.png')

#元素截图
ele = driver.find_element(By.CSS_SELECTOR, 'h1')
ele.screenshot('./image.png')

# 切换页面
handles = driver.window_handles() #获取当前浏览器的所有标签页
driver.switch_to_window(handles[0]) #定位到第二个标签页

# 等待新窗口完成并切换
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC
original_window = driver.current_window_handle
elem.click()
wait.until(EC.number_of_windows_to_be(2))
# 循环执行，直到找到一个新的窗口句柄
for window_handle in driver.window_handles:
    if window_handle != original_window:
        driver.switch_to.window(window_handle)
        break
# 等待新标签页完成加载内容
wait.until(EC.title_is("SeleniumHQ Browser Automation"))
```
