# ntopconf 2023 - takeaway #3

### Let's write a small Python script to analyze our dataset with machine learning using [sklearn](https://scikit-learn.org/stable/)
#### Example 1 - Analysis based on the domain length
``` python
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score

# Assume we have a dataset with two columns: 'domain' and 'label', where 'label' indicates if the domain is used for DNS tunneling (1) or not (0).
data = pd.read_csv('dataset.csv')

# Extract features from the domain (for simplicity, we just use the domain length as a feature in this example)
data['domain_length'] = data['domain'].apply(len)
X = data[['domain_length']]
y = data['label']

# Split the data into training and test sets
X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.3, random_state=42)

# Create and train a model
model = RandomForestClassifier()
model.fit(X_train.values, y_train)

# Evaluate the model
y_pred = model.predict(X_test.values)  
print(f"Accuracy: {accuracy_score(y_test, y_pred)}")

def is_dns_tunneling(domain: str, model):
    length = len(domain)
    prediction = model.predict([[length]])  
    return bool(prediction[0])

# Test
test_domain = "udcgnlxqnlgltzw91dccsihr5cgu9aw4.exfiltration.test"
legit_domain = "www.ntop.org"
print(f"Is the domain {test_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(test_domain, model) else 'No'}")
print(f"Is the domain {legit_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(legit_domain, model) else 'No'}")
```

### Ouput example
```
Accuracy: 0.9987677141096735
Is the domain udcgnlxqnlgltzw91dccsihr5cgu9aw4.exfiltration.test used for DNS tunneling? Yes
Is the domain www.ntop.org used for DNS tunneling? No
```

### Ouput example
```python
...
test_domain = "u9aw4.exfiltration.test"
legit_domain = "www.ntop.org"
print(f"Is the domain {test_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(test_domain, model) else 'No'}")
print(f"Is the domain {legit_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(legit_domain, model) else 'No'}")
...
```
```shell
Accuracy: 0.9987677141096735
Is the domain u9aw4.exfiltration.test used for DNS tunneling? No
Is the domain www.ntop.org used for DNS tunneling? No
```

## Why?

#### Example 2 - Analysis based on the domain strings
```python
import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.feature_extraction.text import TfidfVectorizer

# Read the dataset
with open('dataset.csv', 'r') as file:
    lines = file.readlines()

domains = []
labels = []

for line in lines:
    try:
        domain, label = line.strip().split(',')
        domains.append(domain)
        labels.append(int(label))
    except ValueError:
        continue

df = pd.DataFrame({'domain': domains, 'is_tunneling': labels})

# Convert the domain strings into a numerical representation using TF-IDF.
# what is TF-IDF? -> https://monkeylearn.com/blog/what-is-tf-idf/
vectorizer = TfidfVectorizer(max_features=500, analyzer='char', ngram_range=(1, 3))
X = vectorizer.fit_transform(df['domain'])
y = df['is_tunneling']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = RandomForestClassifier()
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(f"Accuracy: {accuracy_score(y_test, y_pred)}")

def is_dns_tunneling(domain: str, model, vectorizer):
    transformed_domain = vectorizer.transform([domain])
    prediction = model.predict(transformed_domain)
    return bool(prediction[0])

# Test
test_domain = "u9aw4.exfiltration.test"
legit_domain = "www.ntop.org"
print(f"Is the domain {test_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(test_domain, model, vectorizer) else 'No'}")
print(f"Is the domain {legit_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(legit_domain, model, vectorizer) else 'No'}")
```
```shell
ap@userdeb:~/ntopconf$ python3 ml2.py 
Accuracy: 1.0
Is the domain u9aw4.exfiltration.test used for DNS tunneling? Yes
Is the domain www.ntop.org used for DNS tunneling? No
```

## Extra: since how long the domain is registered?
```python
import whois
from datetime import datetime, timedelta

import numpy as np
import pandas as pd
from sklearn.model_selection import train_test_split
from sklearn.ensemble import RandomForestClassifier
from sklearn.metrics import accuracy_score
from sklearn.feature_extraction.text import TfidfVectorizer

days=5 # threshold to consider a domain suspicious

def domain_recently_registered(domain_name):
    try:
        w = whois.whois(domain_name)
        if w.creation_date:
            # If there are multiple dates (happens sometimes), take the first one
            if type(w.creation_date) is list:
                creation_date = w.creation_date[0]
            else:
                creation_date = w.creation_date
                
            if type(creation_date) is str:
                creation_date = datetime.strptime(creation_date, '%Y-%m-%d %H:%M:%S')
            
            today = datetime.now()
            if today - creation_date <= timedelta(days):
                return True
    except whois.WhoisError:
        print(f"Error fetching WHOIS information for {domain_name}")
    except Exception as e:
        print(f"Unexpected error: {e}")
    return False

def is_dns_tunneling(domain: str, model, vectorizer):
    transformed_domain = vectorizer.transform([domain])
    prediction = model.predict(transformed_domain)
    return bool(prediction[0])


with open('dataset.csv', 'r') as file:
    lines = file.readlines()

domains = []
labels = []

for line in lines:
    try:
        domain, label = line.strip().split(',')
        domains.append(domain)
        labels.append(int(label))
    except ValueError:
        continue

df = pd.DataFrame({'domain': domains, 'is_tunneling': labels})

# Convert the domain strings into a numerical representation using TF-IDF.
# what is TF-IDF? -> https://monkeylearn.com/blog/what-is-tf-idf/
vectorizer = TfidfVectorizer(max_features=500, analyzer='char', ngram_range=(1, 3))
X = vectorizer.fit_transform(df['domain'])
y = df['is_tunneling']

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, random_state=42)

model = RandomForestClassifier()
model.fit(X_train, y_train)

y_pred = model.predict(X_test)
print(f"Accuracy: {accuracy_score(y_test, y_pred)}")

# Test
test_domain = "u9aw4.exfiltration.test"
legit_domain = "www.ntop.org"
print(f"Is the domain {test_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(test_domain, model, vectorizer) else 'No'}")
# not possible in this case because the domain exfiltration.test is not a public domain but a resolved host in my intranet
# print(f"Is the domain {test_domain} a new domain? {'Yes' if domain_recently_registered(test_domain) else 'No'}")
print(f"Is the domain {legit_domain} used for DNS tunneling? {'Yes' if is_dns_tunneling(legit_domain, model, vectorizer) else 'No'}")
print(f"Is the domain {legit_domain} a new domain? {'Yes' if domain_recently_registered(legit_domain) else 'No'}")
```
```shell
ap@userdeb:~/ntopconf$ python3 ml2.py 
Accuracy: 1.0
Is the domain u9aw4.exfiltration.test used for DNS tunneling? Yes
Is the domain www.ntop.org used for DNS tunneling? No
Is the domain www.ntop.org a new domain? No
```



