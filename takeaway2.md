# ntopconf 2023 - takeaway #2

### Creating a csv file containing a dataset with legit and suspicious domains

```python
import os
import json

from ntopng.ntopng import Ntopng

username     = "SET_YOUR_USERNAME"
password     = "SET_YOUR_PASSWORD"
ntopng_url   = "SET_URL" # http://localhost:3000

try:
	my_ntopng = Ntopng(username, password, None, ntopng_url)
	interfaces_available = my_ntopng.get_interfaces_list()
	for interface in interfaces_available:
		print (interface)

	# {'ifid': 1, 'name': 'ens33', 'ifname': 'ens33'}
	# {'ifid': 2, 'name': 'lo', 'ifname': 'lo'}

	my_interface = my_ntopng.get_interface(1)
	traffic = my_interface.get_active_flows_paginated(1,500)
	traffic_b = json.dumps(traffic, indent=4)
	print (traffic_b)

	unique_info = []
	if 'data' in traffic:
		for entry in traffic['data']:
			if ('DNS' in entry['protocol']['l7'] and entry['info'] not in unique_info):
				unique_info.append(entry['info'])
				print (entry['info'])

except ValueError as e:
	print(e)
	os._exit(-1)
``` 


```shell
ap@nt:~/ntopng/python/examples$ mkdir domains
ap@nt:~/ntopng/python/examples$ python3 ex1.py  > domains/suspicious_domains.txt
ap@nt:~/ntopng/python/examples$ cat domains/suspicious_domains.txt
y2hly2sg.exfiltration.test
zg5z.exfiltration.test
...
```

#### Add an extra column to each line in the csv containing the value "1" (we will see the reason in the next exercise)
```shell
ap@nt:~/ntopng/python/examples$ sed -i 's/$/,1/' domains/suspicious_domains.txt
ap@nt:~/ntopng/python/examples$ cat domains/suspicious_domains.txt
y2hly2sg.exfiltration.test,1
zg5z.exfiltration.test,1
...
```

[Link to suspicious_domains.txt](https://github.com/PMorphin/ntop2023/tree/main/resources/suspicious_domains.txt)


#### Genereting legit domains

```python
file_subdomains = open('most-common-subs.txt', 'r')
file_domains = open('alexa-top1000.txt', 'r')
file_legit_domains = open ('legit_domains.txt', 'w')

subdomains = file_subdomains.readlines()
domains = file_domains.readlines()

for s1 in subdomains:
    for s2 in domains:
        file_legit_domains.write(s1.strip() + '.' + s2)
```
```
www.google.com
www.youtube.com
www.tmall.com
www.qq.com
www.baidu.com
www.facebook.com
www.sohu.com
www.login.tmall.com
www.taobao.com
www.yahoo.com
www.jd.com
www.wikipedia.org
www.360.cn
www.amazon.com
www.sina.com.cn
www.pages.tmall.com
www.weibo.com
www.live.com
www.reddit.com
www.vk.com
www.okezone.com
www.xinhuanet.com
www.netflix.com
www.blogspot.com
www.csdn.net
www.office.com
www.alipay.com
www.yahoo.co.jp
www.instagram.com
www.zhanqi.tv
www.bongacams.com
www.microsoftonline.com
www.naver.com
www.panda.tv
www.bing.com
www.google.com.hk
www.microsoft.com
www.stackoverflow.com
www.tribunnews.com
www.livejasmin.com
www.china.com.cn
www.twitch.tv
www.google.co.in
www.aliexpress.com
www.ebay.com
www.amazon.co.jp
www.mama.cn
www.twitter.com
www.myshopify.com
www.msn.com
...
```
#### Add an extra column to each line in the csv containing the value "0" (we will see the reason in the next exercise)
```shell
ap@nt:~/ntopng/python/examples$ sed -i 's/$/,0/' legit_domains.txt
ap@nt:~/ntopng/python/examples$ cat legit_domains.txt
www.google.com,0
www.youtube.com,0
www.tmall.com,0
www.qq.com,0
...
```

[Link to legit_domains.txt | head -n 100000](https://github.com/PMorphin/ntop2023/tree/main/resources/legit_domains.txt)


#### create a single file 'dataset.csv' with legit and suspicious domains


