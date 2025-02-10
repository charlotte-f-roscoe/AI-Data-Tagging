---
title: Creating an AI Data Tagging Solution for Amazon RDS

---

# Creating an AI Data Tagging Solution for Amazon RDS
Charlotte Roscoe & Lydia Schubert

**In this markdown document we will be giving a general overview of the steps we took to create an AI Data Tagging Solution for Amazon RDS. Detailed instructions can be found in the documents corresponding to the steps.**

## Step 0: Defining the Project
In our project, we created an AI data tagging solution that assigns a sensitivity score on a scale of 1 to 100 to an Amazon RDS instance based on the data contained in the database. The system will assign the sensitivity score based on the most sensitive word it finds in the database columns.

![Original Overview](https://i.postimg.cc/j2wy69yc/AWS-Flowchart.png)

1. Creating an Amazon RDS instance and storing data
2. Creating an Amazon OpenSearch domain
3. Creating Lambda function to use vector embedding of Amazon Bedrock Titan on RDS database data, 
4. And then storing those Vector Embeddings in OpenSearch
5. Creating a Lambda function to retrieve the vector embeddings from OpenSearch,
6. And use them to query the Claude LLM with RAG
7. Using the API Gateway to simplify user experience and handle user input

Because of our time limit, we scaled down our plan to a smaller version that keeps the same core elements of the original.
![Reduced Scope Overview](https://i.postimg.cc/tg15N2WB/AWS-Reduced-Flowchart.png)

Step 1: Indexing an Amazon RDS database with OpenSearch
* Creating an RDS instance and storing data
* Creating an OpenSearch domain
* Creating a Lambda function that stores the RDS database data in OpenSearch

Step 2: Training and Tuning a pre-trained model in Amazon SageMaker
* Training and Tuning a pre-trained model (distilBERT) in Amaazon SageMaker

Step 3: Connecting to OpenSearch from Amazon SageMaker
* Connecting to the OpenSearch index from Amazon SageMaker using the endpoint, username and password
* Using the pre-trained model to generate a sensitivity score based on OpenSearch Index data, determining the score based on the most sensitive word retrieved from the index


## Step 1: Indexing an Amazon RDS instance with OpenSearch
* Creating an Amazon RDS instance and OpenSearch Domain
* Creating a Lambda function that indexes the contents of the RDS instance to OpenSearch.
* The format is one index per database, and one document per table

[Full Instructions Here](https://hackmd.io/MjcH-_gVQlaAtg7t_XmPQA?view)
## Vector Embedding with Amazon Bedrock (Titan)
* Ensure you have necessary permissions by updating IAM roles
* Use the AWS SDK to initialize the Titan client
```python
import boto3
import json

bedrock = boto3.client(
    service_name = 'bedrock',
    region_name = 'us-west-2',
)

bedrock_runtime = boto3.client(
    service_name = 'bedrock-runtime',
    region_name = 'us-west-2',
)
```
* Define model parameters
```python
model_id = 'amazon.titan-embed-text-v1'
accept = 'application/json'
content_type = 'application/json'
```
* Create a request to the Amazon Titan API to generate vector embeddings for the data
* Example:
```python
response = bedrock_runtime.invoke_model(
    body = json.dumps({"inputText":prompt_data,}),
    modelID = model_id,
    accept = accept,
    contentType = content_type
    
)
```
* Extract the embeddings from the response
* Store embeddings in OpenSearch Serverless vector engine

[Relevant Documentation](https://aws.amazon.com/blogs/machine-learning/getting-started-with-amazon-titan-text-embeddings/)
## Step 2: Training and Testing a LLM in SageMaker Lab
* Creating a dataset with over 1000 entries relevant to our use case. 
Example of Dataset:

| word | sensitivity score | 
| -------- | -------- | 
| Classified     | 99     | 
| Apple | 1 |
| Address | 50 |
|... | ...|

* Choosing a LLM for our use case. In our execution we selected distilBERT.
* Training and Tuning the LLM with the created data
*  Creating an example array of collumn names
*  Looping through column names, determining sensitivity score for each and storing the largest found.
* Printing the largest found sensitivity score as the score for the entire array.

[Full Instructions Here](https://github.com/sb-thorben/aidatatagging/blob/main/Documentation/Databases/3%20-%20Training%20and%20Testing%20an%20LLM.ipynb)

## Step 3: Querying OpenSearch in SageMaker Studio Lab
* Connecting to the OpenSearch index using username, password, and the endpoint URL
* Querying for only the columns ('Rows' in OpenSearch)
* Determining a sensitivity score for each column name, storing the highest retrieved
* Returning the highest retrieved score to the user

[Full Instructions Here](https://hackmd.io/@9M1my1S9ThO994UfBCAfow/ryw57b1YC)