# ntopconf 2023 - takeaway #1
### Documentation https://www.ntop.org/guides/ntopng/api/python/index.html


### Prerequisites
```shell
1) Running ntopng 
2) apt-get install python3 python3-pip git # python3, python3-pip and git installed
3) git clone https:/github.com/pmorphin/ntop-2023.git
4) cd ntop-2023
5) pip3 install -r requirements.txt
6) jupyter notebook
```

### Run ntopng live mode
```shell
ap@nt:~/ntopng/$./ntopng --dont-change-user
```

### Run ntopng to analyze a pcap file
```shell
ap@nt:~/ntopng/$./ntopng --dont-change-user -i path/to/the/traffic/file.pcapng
```

### Login and list all available interfaces
```python
import os

from ntopng.ntopng import Ntopng

username     = "SET_YOUR_USERNAME"
password     = "SET_YOUR_PASSWORD"
ntopng_url   = "SET_URL" # http://localhost:3000

try:
	my_ntopng = Ntopng(username, password, None, ntopng_url)
	interfaces_available = my_ntopng.get_interfaces_list() # Return all available interfaces
	for interface in interfaces_available:
		print (interface)
except ValueError as e:
	print(e)
	os._exit(-1)
```
 ### Output live mode
```shell
ap@nt:~/ntopng/python/examples$ python3 ex1.py 
{'ifid': 1, 'name': 'ens33', 'ifname': 'ens33'}
{'ifid': 2, 'name': 'lo', 'ifname': 'lo'}
```
### Output with a pcapng file
```shell
ap@nt:~/ntopng/python/examples$ python3 ex1.py 
{'ifname': 'dns_tunnelling.pcapng', 'name': 'dns_tunnelling.pcapng', 'ifid': 3}
```
\
![Alt text](https://github.com/PMorphin/ntopconf2023/blob/main/img/ntop-dns-traffic.png)
\
### Retrieve the (paginated) list of active flows for the specified interface
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
	# OR
	# {'ifname': 'dns_tunnelling.pcapng', 'name': 'dns_tunnelling.pcapng', 'ifid': 3}

	my_interface = my_ntopng.get_interface(3)

	# get_active_flows_paginated(currentPage, perPage)
	traffic = my_interface.get_active_flows_paginated(1,500)
	traffic_b = json.dumps(traffic, indent=4)
	print (traffic_b)

except ValueError as e:
	print(e)
	os._exit(-1)
```
 ### Output
```shell
...
        {
            "hash_id": "16779742",
            "server": {
                "is_blacklisted": false,
                "is_broadcast": true,
                "name": "www.adsl.vf",
                "is_dhcp": false,
                "port": 53,
                "ip": "192.168.1.1"
            },
            "protocol": {
                "l7": "DNS",
                "l4": "UDP"
            },
            "key": "2169561366",
            "thpt": {
                "bps": 0.0,
                "pps": 0.0
            },
            "vlan": 0,
            "breakdown": {
                "cli2srv": 100,
                "srv2cli": 0
            },
            "first_seen": 1694761652,
            "bytes": 82,
            "duration": 0,
            "last_seen": 1694761652,
            "client": {
                "is_broadcast_domain": true,
                "is_blacklisted": false,
                "name": "nt.station",
                "is_dhcp": false,
                "port": 38693,
                "ip": "192.168.1.8"
            }
        },
...
``` 
![Alt text](https://github.com/PMorphin/ntopconf2023/blob/main/img/ntop-dns-traffic-info.png)

### Extend get_active_flows_paginated () results with "info" field 
```shell
ap@nt:~/$cat ntopng/python/ntopng/interface.py
```
```python
...
  def get_active_flows_paginated(self, currentPage, perPage):
        """
        Retrieve the (paginated) list of active flows for the specified interface
        
        :param currentPage: The current page
        :type currentPage: int
        :param perPage: The number of results per page
        :type perPage: int
        :return: All active flows
        :rtype: array
        """
        return(self.ntopng_obj.request(self.rest_v2_url + "/get/flow/active.lua", {"ifid": self.ifid, "currentPage": currentPage, "perPage": perPage}))
...
``` 

``` shell
ap@nt:~/$cat ntopng/scripts/lua/rest/v2/get/flow/active.lua
```
``` lua
...
for _key, value in ipairs(flows_stats) do
   local record = {}

   local key = value["ntopng.key"]

   record["key"] = string.format("%u", value["ntopng.key"])
   record["hash_id"] = string.format("%u", value["hash_entry_id"])
   -- training
   record["info"] = value["info"]

   record["first_seen"] = value["seen.first"]
   record["last_seen"] = value["seen.last"]

   local client = {}

   local cli_name = flowinfo2hostname(value, "cli")
   client["name"] = stripVlan(cli_name)
   client["ip"] = value["cli.ip"]
   client["port"] = value["cli.port"]
...
```
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
	# OR
	# {'ifname': 'dns_tunnelling.pcapng', 'name': 'dns_tunnelling.pcapng', 'ifid': 3}

	my_interface = my_ntopng.get_interface(3)

	# get_active_flows_paginated(currentPage, perPage)
	traffic = my_interface.get_active_flows_paginated(1,500)
	traffic_b = json.dumps(traffic, indent=4)
	print (traffic_b)

	unique_info = [] 
	if 'data' in traffic: # if data exists
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
ap@nt:~/ntopng/python/examples$ python3 ex1.py  > domains/results.txt
ap@nt:~/ntopng/python/examples$ cat domains/results.txt
y2hly2sg.exfiltration.test
zg5z.exfiltration.test
...
``` 
