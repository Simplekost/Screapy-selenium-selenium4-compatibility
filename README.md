# scrapy-selenium: Selenium 4+ compatibility patch

> **Problem:** Selenium 4 removed the `executable_path` argument from WebDriver constructors. Older `scrapy-selenium` middleware (or user code) that calls `WebDriver(executable_path=...)` crashes with:
>
> ```text
> TypeError: WebDriver.__init__() got an unexpected keyword argument 'executable_path'
> ```

---

## Summary

This document describes a robust patch for `scrapy-selenium` middleware so it works with **Selenium 4+**. The patch:

* Instantiates Chrome using `selenium.webdriver.chrome.service.Service` and passes it to the driver as `service=Service(...)` (Selenium 4 pattern).
* Keeps **Remote** (Grid) initialization and a **legacy fallback** for other drivers to preserve backwards compatibility.
* Hardened middleware: defends against `None` values for arguments, missing cookies, and uses the correct `spider_closed(self, spider)` signature.

---

## Where to patch (precise local paths)

**Make a backup of the original `middlewares.py` before editing.** Prefer maintaining a fork of `scrapy-selenium` and applying changes under version control rather than editing site-packages directly.

Typical locations depending on how Python/Anaconda is installed:

* **Standard Windows Python installation** (example):

  ```text
  C:\Users\<YOUR_USER>\AppData\Local\Programs\Python\Python311\Lib\site-packages\scrapy_selenium\middlewares.py
  ```

* **Anaconda / Miniconda (Windows)** (example):

  ```text
  C:\Users\<YOUR_USER>\anaconda3\envs\<YOUR_ENV>\Lib\site-packages\scrapy_selenium\middlewares.py
  ```

* **Anaconda / Miniconda (macOS / Linux)** (example):

  ```text
  /home/<YOUR_USER>/anaconda3/envs/<YOUR_ENV>/lib/python3.10/site-packages/scrapy_selenium/middlewares.py
  ```

**How to find the exact location in an activated conda env:**

```bash
conda activate <env>
python -c "import scrapy_selenium, os; print(os.path.join(os.path.dirname(scrapy_selenium.__file__), 'middlewares.py'))"
```

Or list package files:

```bash
pip show -f scrapy-selenium
```

---

## Patch 

**Concept:**

* If driver is Chrome and `SELENIUM_DRIVER_EXECUTABLE_PATH` is provided → use `Service(...)` and pass `service=...` and `options=...` to the WebDriver constructor.
* If `SELENIUM_COMMAND_EXECUTOR` (remote) is provided → use `webdriver.Remote(...)` and supply options/capabilities.
* Otherwise → fallback to the old-style constructor (keeps compatibility for drivers that still need `executable_path`).

**Note:** for grids (Selenium Grid 4+) you may need to adapt capabilities to W3C style (`{"alwaysMatch": ...}`) depending on your Grid configuration.

A hardened, copy/paste-ready middleware implementation is included in this document. Use it to replace `middlewares.py` in your site-packages or keep it as a local project middleware and point `DOWNLOADER_MIDDLEWARES` at it.

---
Drop this in your Scrapy-Selenium Middleware.py

---python

from importlib import import_module
from scrapy import signals
from scrapy.exceptions import NotConfigured
from scrapy.http import HtmlResponse
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.chrome.service import Service
from .http import SeleniumRequest

class SeleniumMiddleware:
    """Scrapy middleware handling the requests using selenium"""

    def __init__(self, driver_name, driver_executable_path,
        browser_executable_path, command_executor, driver_arguments):
        """Initialize the selenium webdriver"""

        webdriver_base_path = f'selenium.webdriver.{driver_name}'
        driver_klass_module = import_module(f'{webdriver_base_path}.webdriver')
        driver_klass = getattr(driver_klass_module, 'WebDriver')
        driver_options_module = import_module(f'{webdriver_base_path}.options')
        driver_options_klass = getattr(driver_options_module, 'Options')
        driver_options = driver_options_klass()

        if browser_executable_path:
            driver_options.binary_location = browser_executable_path
        for argument in driver_arguments:
            driver_options.add_argument(argument)

        # Chrome/Selenium 4.x style
        if driver_name and driver_name.lower() == 'chrome':
            service = Service(driver_executable_path)
            self.driver = driver_klass(service=service, options=driver_options)
        # Remote driver
        elif command_executor is not None:
            from selenium import webdriver
            capabilities = driver_options.to_capabilities()
            self.driver = webdriver.Remote(command_executor=command_executor,
                                           desired_capabilities=capabilities)
        # Other drivers (fallback to old style)
        else:
            driver_kwargs = {
                'executable_path': driver_executable_path,
                f'{driver_name}_options': driver_options
            }
            self.driver = driver_klass(**driver_kwargs)

    @classmethod
    def from_crawler(cls, crawler):
        driver_name = crawler.settings.get('SELENIUM_DRIVER_NAME')
        driver_executable_path = crawler.settings.get('SELENIUM_DRIVER_EXECUTABLE_PATH')
        browser_executable_path = crawler.settings.get('SELENIUM_BROWSER_EXECUTABLE_PATH')
        command_executor = crawler.settings.get('SELENIUM_COMMAND_EXECUTOR')
        driver_arguments = crawler.settings.get('SELENIUM_DRIVER_ARGUMENTS')

        if driver_name is None:
            raise NotConfigured('SELENIUM_DRIVER_NAME must be set')

        if (driver_name.lower() != 'chrome') and (driver_executable_path is None and command_executor is None):
            raise NotConfigured('Either SELENIUM_DRIVER_EXECUTABLE_PATH or SELENIUM_COMMAND_EXECUTOR must be set')

        middleware = cls(
            driver_name=driver_name,
            driver_executable_path=driver_executable_path,
            browser_executable_path=browser_executable_path,
            command_executor=command_executor,
            driver_arguments=driver_arguments
        )

        crawler.signals.connect(middleware.spider_closed, signals.spider_closed)
        return middleware

    def process_request(self, request, spider):
        if not isinstance(request, SeleniumRequest):
            return None

        self.driver.get(request.url)

        for cookie_name, cookie_value in request.cookies.items():
            self.driver.add_cookie({
                'name': cookie_name,
                'value': cookie_value
            })

        if request.wait_until:
            WebDriverWait(self.driver, request.wait_time).until(request.wait_until)

        if request.screenshot:
            request.meta['screenshot'] = self.driver.get_screenshot_as_png()

        if request.script:
            self.driver.execute_script(request.script)

        body = str.encode(self.driver.page_source)
        request.meta.update({'driver': self.driver})

        return HtmlResponse(
            self.driver.current_url,
            body=body,
            encoding='utf-8',
            request=request
        )

    def spider_closed(self):
        self.driver.quit()
---

## Recommended settings (`settings.py`)

```python
import os

# path to chromedriver (example: repository root or absolute)
CHROMEDRIVER_PATH = os.path.join(os.getcwd(), "chromedriver.exe")  # windows example

SELENIUM_DRIVER_NAME = 'chrome'
SELENIUM_DRIVER_EXECUTABLE_PATH = CHROMEDRIVER_PATH
SELENIUM_BROWSER_EXECUTABLE_PATH = None  # optional: path to Chrome/Chromium binary
SELENIUM_COMMAND_EXECUTOR = None  # set if you use a remote Grid
SELENIUM_DRIVER_ARGUMENTS = [
    '--start-maximized',
    '--disable-blink-features=AutomationControlled',
    '--disable-infobars',
    '--disable-dev-shm-usage',
    '--no-sandbox',
    '--headless=new'  # optional: remove if you need a visible browser
]

DOWNLOADER_MIDDLEWARES = {
    # if you patched site-packages, this points to the package middleware
    'scrapy_selenium.SeleniumMiddleware': 800,
    # OR if you keep it inside your project, point to your module:
    # 'your_project.selenium_middleware.SeleniumMiddleware': 800,
}
```

---

## Optional: stealth driver factory

If you use **selenium-stealth** to make the browser appear more human, include a factory in your settings (or a helper module). Example using Selenium 4 `Service` pattern:

```python
from selenium import webdriver
from selenium.webdriver.chrome.service import Service
from selenium_stealth import stealth

def get_stealth_driver(chromedriver_path=CHROMEDRIVER_PATH):
    options = webdriver.ChromeOptions()
    for arg in SELENIUM_DRIVER_ARGUMENTS:
        options.add_argument(arg)

    # Service accepts the executable path (Selenium 4)
    service = Service(chromedriver_path)
    driver = webdriver.Chrome(service=service, options=options)

    stealth(driver,
        languages=["en-US", "en"],
        vendor="Google Inc.",
        platform="Win32",
        webgl_vendor="Intel Inc.",
        renderer="Intel Iris OpenGL Engine",
        fix_hairline=True,
    )
    return driver
```

If you prefer the keyword form: `Service(executable_path=chromedriver_path)` also works with Selenium 4.

**Usage:** import `get_stealth_driver` from your settings or factory module inside a custom spider.

---

## Spiders (example files in repo)

* `silkdeal/spiders/compt_deal.py` — Slickdeals Computer Deals spider (uses `SeleniumRequest` and driver.page\_source; paginates via clicking and waits)
* `silkdeal/spiders/silkdeal_spy.py` — Dev/stealth spider demonstrating `get_stealth_driver()` and other stealth options. Saves screenshots and demonstrates human-like delays.

Open `silkdeal/spiders/` to inspect those files for real examples you can adapt.

---

## How to run the sample spiders

From repository root (where `scrapy.cfg` lives):

```bash
# Slickdeals spider
scrapy crawl compt_deal -o deals.json

# Dev stealth spider
scrapy crawl silkdeal_spy -o duck_links.json
```

Files `deals.json` and `duck_links.json` will be written with scraped items.

---

## Troubleshooting — common errors & fixes

* **Error:** `TypeError: WebDriver.__init__() got an unexpected keyword argument 'executable_path'`

  * **Cause:** Unpatched `scrapy-selenium` middleware or outdated local code still calling `executable_path=`.
  * **Fix:** Apply the middleware patch (use `Service(...)` pattern) or replace the package file with the patched version. Prefer maintaining a fork and applying the change under version control.

* **Error:** `ModuleNotFoundError: No module named 'scrapy_selenium'`

  * **Cause:** Package not installed in the active environment.
  * **Fix:** Activate the correct conda env and `pip install scrapy-selenium` or `pip install -e <path-to-fork>`.

* **Chrome not found / binary errors**

  * Ensure `SELENIUM_BROWSER_EXECUTABLE_PATH` is set if Chrome is not in PATH.
  * For headless servers, include `--no-sandbox` and `--disable-dev-shm-usage` in arguments.

* **Remote / Grid issues**

  * For Selenium Grid 4 or cloud grids (BrowserStack, SauceLabs), adapt remote capabilities to W3C style if needed. Example: wrap options in `{"alwaysMatch": options.to_capabilities()}` or pass `options=...` depending on the Grid.

---

## Test checklist (concrete steps)

1. Confirm Selenium version in your active env: `python -c "import selenium; print(selenium.__version__)"` (ensure `>= 4`).
2. Backup original `middlewares.py`.
3. Apply the patched middleware to either:

   * Edit `site-packages/scrapy_selenium/middlewares.py` (quick patch), or
   * Maintain a fork and `pip install -e .` and point `DOWNLOADER_MIDDLEWARES` at the package, or
   * Drop the patched `selenium_middleware.py` into your project and point `DOWNLOADER_MIDDLEWARES` at it.
4. Set `SELENIUM_DRIVER_NAME` and `SELENIUM_DRIVER_EXECUTABLE_PATH` in `settings.py`.
5. Run a simple spider that issues a `SeleniumRequest`; verify you receive an `HtmlResponse` and Chrome launches.
6. Verify you no longer see the `TypeError: WebDriver.__init__() got an unexpected keyword argument 'executable_path'` error.
7. Test remote usage if applicable by setting `SELENIUM_COMMAND_EXECUTOR` to the grid URL.
8. Confirm `spider_closed` quits the browser and no dangling chromedriver processes remain.
