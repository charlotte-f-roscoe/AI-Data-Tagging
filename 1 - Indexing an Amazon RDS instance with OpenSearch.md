---
title: Indexing an Amazon RDS instance with OpenSearch

---

## Indexing an Amazon RDS instance with OpenSearch
Charlotte Roscoe & Lydia Schubert

### Step 1: Environment Setup
Create your RDS DB instance
* Go to AWS Dashboard and search and click `RDS`
![Rds search](https://i.postimg.cc/t4cWQwsj/upload-d3382a6d06d089d7e2bc5f119adf6e75.png)
* Scroll down and click `Create database`
![Create Database](https://i.postimg.cc/bYPbhQws/upload-311b9033014339570abbefa52dac8a5e.png)
* For the Engine, choose `MySQL` or `PostgreSQL`
* For Templates, choose `Free Tier`
![Free Tier](https://i.postimg.cc/PT2Dgvs5/upload-dd3ca0b2be3077f660dd31d34831e28b.png)
* Name your DB instance identifier
* Decide a `self managed` master username and password
* **DISABLE** storage autoscaling
![storage autoscaling](https://i.postimg.cc/GrPGjSXY/upload-4537268f3a2fd056c691f27aedad2842.png)
* enable `public access` for testing purposes
* in advanced settings, make sure to `set an initial name` or else RDS will not create the Database correctly
![initial db name](https://i.postimg.cc/50SX0R2j/upload-62b7f880d66974f2d109636750fcfa14.png)
* Click `Create Database`


Adding Data to Database
* Make sure both your security group and subnets allow for your IP in their inbound traffic rules.
* Download a software that connects to databases such as `MySQLWorkbench` or `SQLectron`
* Enter:
    * `DB name`
    * `port`
    * `endpoint` (for hostname)
    * `username` 
    * `password`
* Set up tables with SQL (you can ask ChatGPT to generate sample data)
* Expected Result:
![db table example](https://i.postimg.cc/kqx1Kcbh/Screenshot-2024-07-16-at-14-13-50.png)


Create your OpenSearch Domain and Index
* Go to AWS Dashboard and search and click `Amazon OpenSearch Service`
![opensearch search](https://i.postimg.cc/BSHDkhsr/upload-67f5830e4bd2108761a41585d64cba4a.png)
* Click `Create domain`
* name your domain and choose `Standard create`
* Select `Dev/Test` as your template
![dev/test](https://i.postimg.cc/L2HfQBWW/upload-55498839223bf4d2cad1f9ba895098d7.png)
* For deployment option(s), click domain `without standby`, and `1 AZ`
![without standby and 1 AZ](https://i.postimg.cc/08JJKt6z/upload-d2aec270e88a8bf2ba663bfdce9c2c87.png)
* For data nodes, select `1 node`
* use `t3.small.search` instance type
![instance settings](https://i.postimg.cc/Xn1wCGQz/upload-2c3d3cd7ec34377a36fa861dacb7db72.png)
* use `less than 30 GB` of storage (10 should be fine)
* enable `public access` for testing purposes
* Enable `fine-grained access` control and create a master user
* `Access policy`: only use fine-grainied access control
* Create Domain
* Once created, Log in to the dashboard and create an index with the same field structure as your database table

### Step 2: Adding Permissions for Lambda
* Navigate to the IAM Console
![IAM search](https://i.postimg.cc/w9rmvKBK/upload-f66c095a6eba6413a8d99e06397573f3.png)
* Click 'Roles' and then 'Create Role'
* Select 'AWS service' as trusted entity type
![AWS service](https://i.postimg.cc/k7gQGFFS/upload-9ec9bf71e92354a29c185d9ac0295589.png)
* Select Lambda as the Service for Use case
![Selecting Lambda](https://i.postimg.cc/B4g5JwDs/upload-24f9a8434df294e6c51f302aa8fe49c4.png)
* Create a new role with these permissions:
`AmazonRDSReadOnlyAccess`
`AmazonOpenSearchServiceFullAccess`
`AWSLambdaBasicExecutionRole`
![pernmissions](https://i.postimg.cc/CYpbkVsS/upload-915ecc3cc6278507e5c16966eed27dec.png)

### Step 3: Writing Lambda Function
* To implement this, you must have node installed.
* Create a folder for your Lambda function on your computer
* Create a file called `index.js`
* In the terminal for this folder, run `npm init`
* Run `npm install mysql @opensearch-project/opensearch`
* then fill in the rest of the code as described below
* We recommend to write your code in an IDE of choice to install necessary node modules and make sure it functions correctly, and then converting it to a Lambda function and uploading it as a zip file

**Code:**
```javascript=
// Dependencies
const mysql = require('mysql');
const { Client } = require('@opensearch-project/opensearch');

// Authentication, use the password you set
var auth = "OPENSEARCH USERNAME:OPENSEARCH PASSWORD";
const dbName = 'DB NAME';  // Database name is used as the index name

// Create a connection to the MySQL database
const connection = mysql.createConnection({
  host: 'DB ENDPOINT',
  user: 'DB USERNAME',
  password: 'DB PASSWORD',
  database: dbName
});

// Create a client to connect to OpenSearch
// split opensearch endpoint in half after https:// and add authentication + @
const client = new Client({
  node: 'https://' + auth + '@' + 'OPENSEARCH ENDPOINT'
});

// Fetches table names
const getTables = () => {
    return new Promise((resolve, reject) => {
      const query = `
        SELECT table_name 
        FROM information_schema.tables 
        WHERE table_schema = '${dbName}';
      `;
  
      connection.query(query, (err, results) => {
        if (err) {
          console.error('Error executing query:', err);
          reject(err);
          return;
        }

        const tableNames = results.map(row => row.TABLE_NAME);
        console.log('Table Names:', tableNames);
        resolve(tableNames);
      });
    });
};

// Fetch data from the database
const fetchData = (tableName) => {
  return new Promise((resolve, reject) => {
    connection.query(`SELECT * FROM ${tableName}`, (error, results) => {
      if (error) {
        return reject(error);
      }
      resolve(results);
    });
  });
};

// Check if index exists
const indexExists = async (indexName) => {
    try {
        const response = await client.indices.exists({ index: indexName });
        return response.body;
    } catch (error) {
        console.error('Error checking index existence:', error);
        throw error;
    }
};

// Create index if it does not exist
const createIndexIfNotExists = async (indexName) => {
    if (!(await indexExists(indexName))) {
        try {
            await client.indices.create({ index: indexName });
            console.log(`Index ${indexName} created.`);
        } catch (error) {
            console.error('Error creating index:', error);
            throw error;
        }
    } else {
        console.log(`Index ${indexName} already exists.`);
    }
};

// Creating and indexing one document per table
async function createDocumentsForDatabase(tablesData, indexName) {
    for (const [tableName, data] of Object.entries(tablesData)) {
        // Create a document for each table
        const document = {
            table: tableName,
            rows: data
        };

        const response = await client.index({
            // Using table name as ID for document
            id: tableName,  
            index: indexName,
            body: document,
            refresh: true,
        });

        console.log(`Adding document for table ${tableName} in index ${indexName}:`);
        console.log(response.body);
    }
}

// Handler function to create index and index data
exports.handler = async (event, context) => {
    try {
        // Uses database name as the index name
        const indexName = dbName.toLowerCase(); 

        // Check if index exists, create if not
        await createIndexIfNotExists(indexName);
        
        // Getting all table names
        const tableNames = await getTables();
        
        // Collect data from all tables
        const tablesData = {};
        for (const tableName of tableNames) {
          console.log(`Fetching data from table: ${tableName}`);
          const data = await fetchData(tableName);
          console.log('Query results: ', data);

          // Store data by table name
          tablesData[tableName] = data; 
        }
    
        await createDocumentsForDatabase(tablesData, indexName);

      } catch (error) {
        console.error('Error: ', error);
      } finally {
        connection.end(); 
      }
};

```
![db query results](https://i.postimg.cc/MxfrKxKw/Screenshot-2024-07-16-at-12-40-41.png)
![indexing results](https://i.postimg.cc/hPrtPgLN/image.png)


* After writing the code, zip up the files (not the folder)
* Navigate to `AWS Lambda`
![AWS Lambda search](https://i.postimg.cc/TT1bf2fq/upload-f6fcdcf75f6512fcb1a73daa92e9eed2.png)
* Select `create from scratch`
* Attach the previously created role to the Lambda function
* Click `'create'`
* in the top right hand corner of the code box, there should be an `upload from` button
* upload your zip file

**Converting a function to a Lambda function**
Simply replace the line
`const main = async () => {`
with 
`exports.handler = async (event, context) => {`
* test your Lambda function
![Testing Lambda](https://i.postimg.cc/fRqjWkg1/Screenshot-2024-07-16-at-13-25-41.png)
* Depending on the size of your database, you may need to edit the time-out value for the Lambda function, which can be done in the 'Configuration' settings.


### Step 4: Testing in OpenSearch Dashboard

If done correctly, you should now be able to see the contents of the database in the OpenSearch dashboard.

![Opensearch result](https://i.postimg.cc/QxkrkbV7/image.png)

### Other Notes
* Depending on how large your database is, you may want to change the loops to create one index per database and one table per document instead of one index per table. This makes queries more complicated but provides better visibility.
    

**Relevant Documentation and Guides can be found here:**
* [OpenSearch JavaScript Client](https://opensearch.org/docs/latest/clients/javascript/index)
* [Connecting to a DB instance running the MySQL database engine](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/USER_ConnectToInstance.html)
* [Node.js MySQL](https://www.w3schools.com/nodejs/nodejs_mysql.asp)
