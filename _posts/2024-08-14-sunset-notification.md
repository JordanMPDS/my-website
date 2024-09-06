---
layout: post
title: "Sunset Notification"
date: 2024-09-04 10:00:00 -0600
categories: process-automation
---

Sunset Notification is a Python-based project that sends you a daily text message with the exact time of sunset. This tool ensures you never miss a sunset. The notification time is intelligently scheduled to align with each day's sunset.

My use case for creating this was to have an alarm for when I need to go outside and lock the door to the chicken coop.  Chickens will go to the coop and onto their roosts just before sunset and changing an alarm every day is annoying.  This tool allows me to have an alarm right at sunset every day without me having to make any changes.

I have this hosted on Databricks because I wanted to learn how to schedule jobs based on when other jobs are completed.  This first task is scheduled to start at 8am every day and it's only purpose is to find the time of the sunset and schedule the subsequent job at the correct time.

Sensitive items such as location and API keys are housed in a separate configuration file.

```python
# Databricks notebook source
import requests
from datetime import datetime
import pytz
import json
from dotenv import dotenv_values
import os

# COMMAND ----------

def datetime_to_cron(dt: datetime) -> str:
    return dt.strftime(f'{dt.second} {dt.minute} {dt.hour} * * ?')

# COMMAND ----------

secrets = dotenv_values("/Workspace/Users/jordan@jordanmartinetti.com/.env")

# COMMAND ----------

url = 'https://api.openweathermap.org/data/2.5/weather?lat='+secrets['LAT']+'&lon='+secrets['LON']+'&appid='+secrets['API_KEY']
response = requests.get(url)
weather = response.json()


# COMMAND ----------

sunset_utc = weather["sys"]["sunset"]
local_timezone = pytz.timezone("US/Central")
sunset_utc_formatted = datetime.fromtimestamp(sunset_utc,local_timezone)

# COMMAND ----------

databricks_instance = secrets['DB_INSTANCE']
api_token = secrets['DB_API_TOKEN'] #apiToken
job_id = secrets['DB_JOB_ID'] #Your notification job id which is created separately.
cron_expression = datetime_to_cron(sunset_utc_formatted)

# COMMAND ----------

payload = {
    "job_id":job_id,
    "new_settings":{
        "schedule": {
        "quartz_cron_expression": cron_expression,
        "timezone_id": "US/Central",  # Set the timezone
        "pause_status": "UNPAUSED"
    }
    }
}

headers = {
    "Authorization": f"Bearer {api_token}",
    "Content-Type": "application/json"
}

# COMMAND ----------

url = f"{databricks_instance}/api/2.1/jobs/update"
response = requests.post(url, headers=headers, data=json.dumps(payload))

# COMMAND ----------

# Check the response
if response.status_code == 200:
    print(f"Job scheduled at {sunset_utc_formatted.strftime('%Y-%m-%d %H:%M:%S')}.")
else:
    print(f"Failed to update job schedule: {response.status_code}")
    print(response.json())
```

This is the second script where the purpose is to send out the text notification to my phone

```python
# Databricks notebook source
import smtplib
from email.mime.text import MIMEText
from email.mime.multipart import MIMEMultipart
from dotenv import dotenv_values
import os
import sys

# COMMAND ----------

secrets = dotenv_values("/Workspace/Users/jordan@jordanmartinetti.com/.env")

# COMMAND ----------

email = secrets['EMAIL']
password = secrets['EMAIL_PASSWORD']
recipient = secrets['RECIPIENT']

# COMMAND ----------

msg = MIMEMultipart()
msg["Subject"] = "Chicken Alarm"
msg["From"] = email
msg["To"] = recipient

# COMMAND ----------

message = "Lock up the chickens"
msg.attach(MIMEText(message))

# COMMAND ----------

msg.as_string()

# COMMAND ----------

print("Setting server")
server = smtplib.SMTP("smtp.gmail.com", 587)
print("Sending hello")
server.ehlo()
# Check the response to EHLO
if server.ehlo_resp:
    print("EHLO response is positive.")
else:
    print("EHLO response is not positive, check connection and try again.")
    sys.exit()

# Proceed with STARTTLS to secure the connection
server.starttls()
# After STARTTLS, it's a good practice to call ehlo() again
server.ehlo()

# Now you can proceed with login and sending the email
server.login(email, password)
text = msg.as_string()
server.sendmail(email, recipient, text)
server.quit()

# COMMAND ----------

print("Message sent, script ending")
```
