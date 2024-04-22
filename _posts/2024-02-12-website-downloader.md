---
layout: post
title:  "Website Downloader"
date:   2024-02-12 14:25:00 -0600
categories: process-automation
---

This year when I was faced with downloading 20K+ files from a website I knew I needed an automated solution.  I worked some new tools like selenium, requests, tqdm, and sqlite.

In the end I was able to put together a fantastic script which would download the files, show progress of the current download, and track downloads in a sqlite database to avoid duplicates.  The full script is below, or [click here][github-downloader] to view it on Github.

```python
import requests
import random
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.chrome.options import Options
from selenium.webdriver.common.keys import Keys
from selenium.webdriver.common.action_chains import ActionChains
from datetime import datetime
from tqdm import tqdm
import sqlite3

# Setting initial paths
documents = "<path-to-save-games>"
con = sqlite3.connect("...\\sqlite3\\newHomeWindows.db")
print("Connected to database")
cur = con.cursor()
excludes = cur.execute("SELECT DISTINCT * FROM old_games").fetchall()
excludesList = [item for t in excludes for item in t]
excludesLen = len(excludes)
print("Got initial list")

# Creating file download progress function
def download(url: str, fname: str, chunk_size=1024):
    resp = requests.get(url, stream=True)
    total = int(resp.headers.get('content-length', 0))
    with open(fname, 'wb') as file, tqdm(
        desc=fname,
        total=total,
        unit='iB',
        unit_scale=True,
        unit_divisor=1024,
    ) as bar:
        for data in resp.iter_content(chunk_size=chunk_size):
            size = file.write(data)
            bar.update(size)

# Creating selenium function for website navigation
def selenium_test(test_url):
    edgeOptions = Options()
    edgeOptions.add_argument("--enable-chrome-browser-cloud-management")
    edgeOptions.add_argument("--headless=new")
    edgeOptions.add_argument('--log-level=3')
    edgeOptions.add_experimental_option('excludeSwitches', ['enable-logging'])
    driver = webdriver.Edge(edgeOptions)
    driver.get("<http://website-here>"+str(test_url))
    print("Opened url")
    textBox = driver.find_element(By.NAME, "acc")
    textBox.send_keys("<api-key>")
    actions = ActionChains(driver)
    actions.send_keys(Keys.TAB).perform()
    actions.send_keys(Keys.ENTER).perform()
    print("Logged in")
    download_link = driver.find_element(By.PARTIAL_LINK_TEXT, "DOWNLOAD")
    downloadText = download_link.get_attribute("href")
    driver.quit()
    now = datetime.now()
    print("Quit driver and started download at: ", now.time())
    downloadTextSlash = downloadText.rfind("/")
    downloadTextLen = len(downloadText)
    downloadTextLenSub = downloadText[downloadTextSlash+1:downloadTextLen]
    fileName = str(test_url)+"-"+downloadTextLenSub
    download(downloadText, documents+fileName)
    print("Saving file " + fileName)

# For loop to go through the entire list of possible websites
for x in range(1, 21426 - excludesLen):
    for i in random.sample(list(set([x for x in range(1,21426)]) - set(excludesList)),1):
        print("Working on file number {:0}".format(i))
        cur.execute("INSERT INTO old_games VALUES ({:0})".format(i))
        con.commit()
        selenium_test(i)
    print("Updating exclusion list")
    excludes = cur.execute("SELECT DISTINCT * FROM old_games").fetchall()
    excludesList = [item for t in excludes for item in t]
    excludesLen = len(excludes)

```

[github-downloader]: https://github.com/JordanMPDS/website-downloader