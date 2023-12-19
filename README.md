# hudi-aws-glue-0.14
How to use Hudi 0.14 on AWS glue
![Screenshot 2023-12-12 at 7 12 06 PM](https://github.com/soumilshah1995/hudi-aws-glue-0.14/assets/39345855/526eab16-a37f-4ca3-b6db-98ab267a48b2)


# Video Guide 
* https://youtu.be/HJ6QQN408AE



# Download The JAR 
* https://repo1.maven.org/maven2/org/apache/hudi/hudi-spark3.3-bundle_2.12/0.14.0/hudi-spark3.3-bundle_2.12-0.14.0.jar
* https://repo1.maven.org/maven2/org/apache/spark/spark-avro_2.13/3.3.0/spark-avro_2.13-3.3.0.jar

Step 2: Set the Job Variables 
![Screenshot 2023-12-19 at 4 10 41 PM](https://github.com/soumilshah1995/hudi-aws-glue-0.14/assets/39345855/e50564a0-38d2-41e6-bd37-92d284f36d6e)


# Step  3: Ru GLue job 

```

import sys
import os
from pyspark.context import SparkContext
from pyspark.sql.session import SparkSession
from awsglue.context import GlueContext
from awsglue.job import Job
from awsglue.dynamicframe import DynamicFrame
from awsglue.utils import getResolvedOptions
from pyspark.sql.functions import col, to_timestamp, monotonically_increasing_id, to_date, when
from pyspark.sql.types import *
from datetime import datetime, date
from faker import Faker
import random
import uuid

os.environ['PYSPARK_PYTHON'] = sys.executable
os.environ['PYSPARK_DRIVER_PYTHON'] = sys.executable

args = getResolvedOptions(sys.argv, options=[])

global faker
faker = Faker()

class DataGenerator(object):

    @staticmethod
    def get_data():
        return [
            (
                uuid.uuid4().__str__(),
                faker.name(),
                faker.random_element(elements=('IT', 'HR', 'Sales', 'Marketing')),
                faker.random_element(elements=('CA', 'NY', 'TX', 'FL', 'IL', 'RJ')),
                faker.random_int(min=10000, max=150000),
                faker.random_int(min=18, max=60),
                faker.random_int(min=0, max=100000),
                faker.unix_time()
            ) for x in range(10)
        ]

# Spark Session
spark = (SparkSession.builder
         .config('spark.serializer', 'org.apache.spark.serializer.KryoSerializer')
         .config('spark.sql.hive.convertMetastoreParquet', 'false')
         .getOrCreate())

sc = spark.sparkContext
glueContext = GlueContext(sc)
job = Job(glueContext)
logger = glueContext.get_logger()

# Variables
table_name = "employees"
bucket = "XXX"
recordkey = "emp_id"
partitionpath = "state"
operation = "upsert"
precombine = "ts"
index = "RECORD_INDEX"

hudi_options = {
    'hoodie.table.name': table_name,
    'hoodie.datasource.write.recordkey.field': recordkey,
    'hoodie.datasource.write.partitionpath.field': partitionpath,
    'hoodie.datasource.write.table.name': table_name,
    'hoodie.datasource.write.operation': operation,
    'hoodie.datasource.write.precombine.field': precombine,
    'hoodie.upsert.shuffle.parallelism': 2,
    'hoodie.insert.shuffle.parallelism': 2,
    "hoodie.metadata.record.index.enable": "true",
    "hoodie.index.type": index
}

# Create Spark Data Frame
data = DataGenerator.get_data()
columns = ["emp_id", "employee_name", "department", "state", "salary", "age", "bonus", "ts"]
df = spark.createDataFrame(data=data, schema=columns)
print(df.show())

hudi_path = f"s3://{bucket}/hudi/{table_name}"
df.write.format("hudi").options(**hudi_options).mode("overwrite").save(hudi_path)


```
