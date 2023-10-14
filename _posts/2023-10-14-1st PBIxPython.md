---
layout: post
title: 2nd 1% - How I created my first PowerBI dashboard with Python script
---

Thank you for visiting my blog! Today I am sharing my project focusing on 3 parts: 
1. Data extraction from Elasticsearch 
2. Data preparation in Python
3. Dashboard creation in PBI

The main idea is to create tabulated data with the necessary columns that could be perfectly represented in a Gantt chart in PBI.

## 1st part: Data extraction from Elasticsearch

### Import the required libraries and encode your credentials
```ruby
import http.client
import json
import pandas as pd
import math
import base64

username = 'YOUR_USERNAME'
password = 'YOUR_PASSWORD'
userpass = username + ':' + password
encoded_u = base64.b64encode(userpass.encode()).decode()
encoded_cred = "Basic %s" % encoded_u
print('Your encoded credential is: ', encoded_cred)
```

### Define search query and connect to Elasticsearch (RESTAPI method to get scroll ID)
```ruby
json_request = {PUT IN YOUR JSON REQUEST HERE}

scroll_request = {  
  "scroll" : "15m", 
  "scroll_id" : "scroll_id1"}

payload = json.dumps(json_request)
authorization = encoded_cred
url= "YOUR ELASTICSEARCH URL"
conn = http.client.HTTPSConnection(url)

#define the header
headers = {'content-type': "application/json",'authorization': ''} 
headers['authorization'] = authorization

#submit connection request
conn.request("GET", "/rest/app.all/_search?scroll=15m", payload, headers) #request data from query
print("Connecting...")

#get response from Elasticsearch
res = conn.getresponse()
print("Received response...") 

#read data to get scroll_ID
data = res.read()
json_data = data.decode('utf-8')
py_obj = json.loads(json_data)

#get first round of data
first_result = list(py_obj['hits']['hits'])
temp=[]
for hit in first_result:
  temp.append({k:v[0] for k,v in hit['fields'].items()})

#pull data using scroll_ID by chunk size
scroll_id = py_obj['_scroll_id']
for x in range(math.ceil(py_obj['hits']['total']['value']/10000)+1):
  print('Pulling batch: ', x)
  scroll_request['scroll_id'] = scroll_id
  conn.request("GET", "/rest/_search/scroll", json.dumps(scroll_request), headers)
  res = conn.getresponse()
  data = res.read()
  json_data = data.decode('utf-8')
  new_py_obj = json.loads(json_data)
  #get new scroll_ID if change
  scroll_id = new_py_obj['_scroll_id']
  new_results = list(new_py_obj['hits']['hits'])
  for hits in new_results:
    temp.append({k:v[0] for k,v in hits['fields'].items()})
print('Data pulling is completed!')
```


## 2nd part: Data preparation in Python

### Transform raw dataset into usable table through string operations such as str.extract, slice, replace

The raw data from my search query are long messages that contains the key strings that we need.
Therefore, I need to extract the key strings from each message and store them in different columns.

![image](https://github.com/annnie04/onepercent/assets/113150580/b186e004-c916-4dc5-a5c1-a3fdf1f15dad)

In high level, I'm creating a dataframe that consist of changes of machine status in the last 24 hours.
This is a sample of raw alarm messages that I obtained and below is how I prepare my dataframe.

![image](https://github.com/annnie04/onepercent/assets/113150580/b8931baf-bf2c-4f1f-9a43-d9318a7d077e)


```ruby
def extract_machine_set_to_down(df):
    set_down_df = raw_df.loc[(raw_df["type"]=='DEBUG') & (raw_df["message"].str.contains('Alarm'))].copy().reset_index(drop=True)
    set_down_df["end_time"] = set_down_df["start_time"] + pd.Timedelta(minutes=10) #Note: The "end_time" is helpful in Gantt chart later

    #Extract string betwween <Message></Message>
    set_down_df['Message'] = set_down_df["message"].str.extract('(Message>.*(?=</Message))',expand=True)
    set_down_df['Message'] = set_down_df['Message'].str[7:]  #number set according to your string index

    #Set alarm activity as Down
    set_down_df['Activity'] = 'Down'

    #Extract machine info
    set_down_df['Bondstage'] = set_down_df["message"].str.extract('(arm>.*(?=</arm))',expand=True)
    set_down_df['Bondstage'] = set_down_df['Bondstage'].str[7:]  #number set according to your string index
    
    return set_down_df

```
### Result:

![image](https://github.com/annnie04/onepercent/assets/113150580/10d01537-24d8-478d-966e-2d000c4960a9)


## 3rd part: Dashboard creation in PBI

### Connect PBI to Python script
As you select to Get Data in PBI, you will be prompted to choose your source of data.
Pick *Python script* and simply paste your Python code into the space provided.
PBI will connect with Python to process and shortly later you will be able to select the dataframes for dashboard building.

![image](https://github.com/annnie04/onepercent/assets/113150580/d1b5f1cc-2fa1-4940-b69d-8dd65b049201)


### Import Gantt 2.2.3 into your PBI and start playing!

![image](https://github.com/annnie04/onepercent/assets/113150580/f34ab597-cda1-49d2-a2ec-93420fae9356)

And this is what I got! With this I am able to know when and what were the status change. 

![image](https://github.com/annnie04/onepercent/assets/113150580/cede2d57-40f4-4885-8b84-ca78f8d497f5)

Hope it helps! :)
