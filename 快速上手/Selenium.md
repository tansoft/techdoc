## 安装

### 安装 chrome 浏览器，如果已经安装，查看版本

```bash
# linux 一键安装
sudo curl https://intoli.com/install-google-chrome.sh | bash
sudo mv /usr/bin/google-chrome-stable /usr/bin/google-chrome

# mac 安装
brew install --cask google-chrome
```

### 安装 chromedriver，控制chrome的驱动

```bash
#last version found in https://chromedriver.storage.googleapis.com/LATEST_RELEASE
#all version found in https://chromedriver.storage.googleapis.com/index.html

# mac 安装
brew install --cask chromedriver

# linux or mac 手动安装
# 1. get current version, such as 103.0.5060.53
#  for linux
chrome_version=`google-chrome --version | awk '{print $3}'`
#  for mac
chrome_version=`"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --version | awk '{print $3}'`
# 2. get current chromedriver
mapping_version=`echo $chrome_version | awk -F. '{printf "%s.%s.%s",$1,$2,$3}'`
mapping_version=`curl https://chromedriver.storage.googleapis.com/LATEST_RELEASE_${mapping_version}`
# 3. download the mapping chromedriver
#  for linux
wget "https://chromedriver.storage.googleapis.com/${mapping_version}/chromedriver_linux64.zip"
#  for mac x86
wget "https://chromedriver.storage.googleapis.com/${mapping_version}/chromedriver_mac64.zip"
#  for mac arm
wget "https://chromedriver.storage.googleapis.com/${mapping_version}/chromedriver_mac64_m1.zip"
# 4. install chromedriver
unzip chromedriver_*.zip
sudo mv chromedriver /usr/local/bin/chromedriver
chromedriver --version
rm -f chromedriver_*.zip
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
wait = WebDriverWait(driver, 20)
'''
#各种条件元素定位 driver.find_element(xx,xx)
 By.ID,'xxxid'
 By.NAME,'xxxname'
 By.CLASS_NAME,'xxxclassname'
 By.LINK_TEXT, 'More information...'
 By.PARTIAL_LINK_TEXT, 'More inf'
 By.CSS_SELECTOR, '#fruits .tomatoes'
 By.CSS_SELECTOR, "[name='login']"
 By.XPATH,"//*[@id='u1']/a[8]"
'''
'''
#各种判断条件 EC.xxx
 #元素加载成功
 presence_of_element_located((By.ID,'xxxid'))
 presence_of_all_elements_located()
 #元素可见或不可见
 visibility_of_element_located
 invisibility_of_element_located
 #判断文本是否在elem.text和elem.value中出现
 text_to_be_present_in_element
 text_to_be_present_in_element_value
 #判断标题
 title_is('xxx')
 title_contains('xxx')
 #判断frame是否可切入，可传入locator元组或定位方式：id、name、index或WebElement
 frame_to_be_available_and_switch_to_it
 #判断是否有alert
 alert_is_present
 #判断元素是否可点击
 element_to_be_clickable
 #以下四个条件判断元素是否被选中，第一个条件传入WebElement对象，第二个传入locator元组
 #第三个传入WebElement对象以及状态，相等返回True，否则返回False
 #第四个传入locator以及状态，相等返回True，否则返回False
 element_to_be_selected
 element_located_to_be_selected
 element_selection_state_to_be
 element_located_selection_state_to_be
 #最后一个条件判断一个元素是否仍在DOM中，传入WebElement对象，可以判断页面是否刷新了
 staleness_of
'''
try:
    wait.until(EC.presence_of_element_located((By.ID,'xxxid')))
    wait.until_not(XXX) #等待直到元素消失
    driver.find_element(By.ID,'xxxid').send_keys('123456')
    driver.find_element(By.ID,'xxxbutton').click()
    driver.find_element(By.CSS_SELECTOR, 'h1').text
    driver.find_element(By.CSS_SELECTOR, "[name='login']").is_displayed()
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
WebDriverWait(driver, 10).until(EC.number_of_windows_to_be(2))
# 循环执行，直到找到一个新的窗口句柄
for window_handle in driver.window_handles:
    if window_handle != original_window:
        driver.switch_to.window(window_handle)
        break
# 等待新标签页完成加载内容
WebDriverWait(driver, 10).until(EC.title_is("SeleniumHQ Browser Automation"))
```
