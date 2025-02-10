---
title: Querying OpenSearch from Amazon SageMaker

---

# Querying OpenSearch from Amazon SageMaker
Charlotte Roscoe & Lydia Schubert

### Step 1: Connecting to OpenSearch

Install OpenSearch python library:
```python=
!pip install opensearch-py
```

```python=
from opensearchpy import OpenSearch, RequestsHttpConnection, AWSV4SignerAuth

```
Connect to OpenSearch
```python=
host = 'OPENSEARCH ENDPOINT'

auth = ('OPENSEARCH USERNAME', 'OPENSEARCH PASSWORD')

client = OpenSearch(
    hosts=host,
    http_compress=True,
    http_auth=auth,
    use_ssl=True,
    verify_certs=True,
    ssl_assert_hostname=False,
    ssl_show_warn=True
)

```
### Step 2: Query OpenSearch
Query for table names:
```python=
query = {
    "_source": False,
        "fields": ["_id"],
        "size": 10000
}

response = client.search(
    body=query,
    index='db_instance'
)

print('\nSearch results:')
table_names = [hit['_id'] for hit in response['hits']['hits']]

print(table_names)

```
Query for column names:
```python=
# Retrieve documents including their full content
query = {
    "_source": True,
    "size": 10000
}

response = client.search(
    body=query,
    index='db_instance'
)

# Set to store unique keys across all tables
all_keys = set()

for hit in response['hits']['hits']:
    rows = hit['_source']['rows']

    # Collect keys from all rows
    for row in rows:
        all_keys.update(row.keys())

# Convert the set of all keys to a list and print the result
all_keys_list = list(all_keys)
print(all_keys_list)
```
Results:
![Query Results](https://i.postimg.cc/d0LqQy6x/image.png)
### Step 3: Query trained model with row names
In our previous SageMaker Jupyter Notebook, replace the `words` array with `all-keys-list` like so:
```python=
#importing libraries
from transformers import AutoTokenizer, AutoModelForSequenceClassification
import torch

# load saved model
tokenizer = AutoTokenizer.from_pretrained('./sensitivity_model')
model = AutoModelForSequenceClassification.from_pretrained('./sensitivity_model')

# Loads the pre-trained tokenizer and model from the ./sensitivity_model directory.
def get_sensitivity_score(text):
    encoding = tokenizer(
        text,
        return_tensors='pt',
        truncation=True,
        padding='max_length',
        max_length=128
    )
    with torch.no_grad():
        outputs = model(**encoding)
    score = outputs.logits.item()
    return score

highest_number = float('-inf')  # Start with the lowest possible value
highest_word = None

# Loop through each word in the list, getting the sensitivity score for each and storing the highest
# THIS IS WHERE YOU REPLACE 'words' WITH ALL_KEYS_LIST
for word in all_keys_list:
    number = get_sensitivity_score(word)
    if number > highest_number:
        highest_number = number
        highest_word = word

# Printing the highest score
print(f"Highest Sensitivity Score: {round(highest_number)}")
```
![Query results](https://i.postimg.cc/3wV3kxbJ/image.png)

This is the final step of the project. Thank you for reading.
### Relevant Documentation
[Low Level Python OpenSearch Client](https://opensearch.org/docs/latest/clients/python-low-level/)