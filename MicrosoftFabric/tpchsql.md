# Microsoft Fabric TPCH SQL

## Process TPCH data in Spark

### Convert Parquet to Delta tables

- tpchspark.md in this repo will cover the steps to convert parquet to delta tables
- [Spark Parquet to Delta](tpchspark.md)
- Read the Customer data
- Right click the files and customer get the full path

```
dfcustomer = spark.read.format('delta').load('abfss://xxxx@onelakename.dfs.fabric.microsoft.com/xxxxx/Tables/tpchcustomer')
```

- Read line items

```
dflineitem = spark.read.format('delta').load('abfss://xxxx@onelakename.dfs.fabric.microsoft.com/xxxx/Tables/tpchlineitem')
```

- Read orders

```
dforders = spark.read.format('delta').load('abfss://xxxx@onelakename.dfs.fabric.microsoft.com/xxxxxx/Tables/tpchorders')
```

- now join the orders and line item

```
dforders.join(dflineitem, dforders.O_ORDERKEY == dflineitem.L_ORDERKEY, "inner").show()
```

![Architecture](https://github.com/balakreshnan/Samples2023/blob/main/MicrosoftFabric/Images/tpch18.jpg "Architecture")

- above will process about 4.19 TB of data
- First add year, month and day column

```
from pyspark.sql.functions import *
dflineitem = dflineitem.withColumn("year", year(col("L_SHIPDATE")))
dflineitem = dflineitem.withColumn("month", month(col("L_SHIPDATE")))
dflineitem = dflineitem.withColumn("day", dayofmonth(col("L_SHIPDATE")))
```

- now take a join dataset

```
dfallorder  = dforders.join(dflineitem, dforders.O_ORDERKEY == dflineitem.L_ORDERKEY, "inner")
```

- now lets sum the revenue by year, month and day

```
group_cols = ["year", "month", "day"]
dfallorder.groupBy(group_cols).agg(sum("L_QUANTITY").alias("Qty"), sum("L_TAX").alias("tax"), sum("L_EXTENDEDPRICE").alias("Price")).show(truncate=False)
```

- process subset of data

```
dfallorder1 = dfallorder.head(10000000)
```

![Architecture](https://github.com/balakreshnan/Samples2023/blob/main/MicrosoftFabric/Images/tpch22.jpg "Architecture")

- convert a list to spark dataframe

```
deptDF = spark.createDataFrame(data=dfallorder1)
```

- run group by

```
group_cols = ["year", "month", "day"]
display(deptDF.groupBy(group_cols).agg(sum("L_QUANTITY").alias("Qty"), sum("L_TAX").alias("tax"), sum("L_EXTENDEDPRICE").alias("Price")))
```

- to list the files to verify

```
mssparkutils.fs.ls("abfss://xxxxxxxxxx@msit-onelake.dfs.fabric.microsoft.com/xxxxxxxxxxxxxx/Tables/tpchlineitem")
```

- to vaccum data

```
from delta.tables import *

spark.sql("SET spark.databricks.delta.retentionDurationCheck.enabled = false")

deltaTable = DeltaTable.forPath(spark,'abfss://xxxxxx@msit-onelake.dfs.fabric.microsoft.com/xxxx/Tables/tpchlineitem')
```

```
deltaTable.vacuum(retentionHours = 10)
```

- write back to delta

```
deptDF.groupBy(group_cols).agg(sum("L_QUANTITY").alias("Qty"), sum("L_TAX").alias("tax"), sum("L_EXTENDEDPRICE").alias("Price")).write.format("delta").mode("overwrite").save('abfss://xxx@onelakename.dfs.fabric.microsoft.com/xxxxxx/Tables/tpchaggr1morders')
```

- now read the data to validate

```
tpchaggr1morders = spark.read.format("delta").load('abfss://xxxxxx@onelakename.dfs.fabric.microsoft.com/xxxxxxx/Tables/tpchaggr1morders')
```

- display the aggregated data

```
display(tpchaggr1morders)
```

- Done