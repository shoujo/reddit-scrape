```python
import os
import sys
import json
import requests
import hashlib
import configparser
import pandas as pd
from pathlib import Path
from datetime import datetime
```

```python
class RedditPost:
    def __init__(self, post, **kwargs):
        self.url = post['data']['url']
        self.sub = post['data']['subreddit_name_prefixed']
        self.author = post['data']['author']
        self.title = post['data']['title']
        self.text = post['data']['selftext']
        self.score = post['data']['score']
        self.date = convert_timestamp("date", post['data']['created_utc'])
        self.time = convert_timestamp("time", post['data']['created_utc'])
        
        
    def _asdict(self):
        post_dict = {
            'sub': self.sub,
            'author': self.author,
            'title': self.title,
            'text': self.text[0:50],
            'score': self.score,
            'date': self.date,
            'time': self.time
        }
        return post_dict
```

```python
def config_section_map(section):
    options_dict = {}
    Config = configparser.ConfigParser()
    Config.read("un.ini")
    options = Config.options(section)
    for option in options:
        try:
            options_dict[option] = Config.get(section, option)
            if options_dict[option] == -1:
                DebugPrint("skip: %s" % option)
        except:
            print("exception on %s!" % option)
            options_dict[option] = None
    return options_dict
```

```python
def get_username():
    # read from db (!)
        # eventually..
    return config_section_map('1')['1']
```

```python
def convert_timestamp(action, timestamp):
    def get_date(timestamp):
        return datetime.utcfromtimestamp(timestamp).strftime('%Y-%m-%d')

    def get_time(timestamp):
        return datetime.utcfromtimestamp(timestamp).strftime('%H:%M')
    
    def invalid_timestamp(timestamp):
        return "invalid timestamp argument!"
        
    switcher = {
        "date": get_date(timestamp),
        "time": get_time(timestamp)
    }
    return switcher.get(action, invalid_timestamp)
```

```python
def csv_exists(username):
    user_csv = Path(username + '.csv')
    if user_csv.is_file():
        return True
    else:
        return False
```

```python
def data_to_dataframe(posts):
    all_posts = pd.DataFrame()
    for post in posts:
        post_val = post._asdict().values()
        all_posts=all_posts.append(pd.Series(post_val).to_frame().T)
    return all_posts
```

```python
def sort_reddit_data(request):
    post_list = []
    posts = request['data']['children']
    for post in posts:
        post_list.append(RedditPost(post))
    return post_list
```

```python
def scrape_reddit(username):
    def generate_url(username):
        base_url = 'https://www.reddit.com/user/'
        end_url = '/submitted.json'
        return base_url + username + end_url
    def generate_out_file(username):
        if csv_exists(username):
            pass
        else:
            try:
                df = pd.DataFrame(columns=['sub', 'author', 'title', 'text', 'score', 'date', 'time'])
                df.to_csv(username + '.csv', mode='w')
            except:
                return "unable to generate outfile!"
    def req_reddit_data(url, headers):
        if url:
            try:
                request = requests.get(url, headers=headers).json()
            except:
                return "unable to make request!"
        return request
    
    generate_out_file(username)
    url = generate_url(username)
    headers = {'User-agent': 'Broad Arrow Scraper v0.1'}
    return req_reddit_data(url, headers)
```

```python
def main():
    target = get_username()
    posts = sort_reddit_data(scrape_reddit(target))
    post_dataframe = data_to_dataframe(posts)
    post_dataframe.to_csv(target + '.csv', mode="a", header=False)
```

```python
if __name__ == "__main__":
    main()
```
