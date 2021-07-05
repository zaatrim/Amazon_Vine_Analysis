# ** Amazon_Vine_Analysis


## *Project Overview*
In this Project I will analyze Amazon reviews written by members of the paid Amazon Vine program. The Amazon Vine program is a service that allows manufacturers and publishers to receive reviews for their products. Companies like SellBy pay a small fee to Amazon and provide products to Amazon Vine members, who are then required to publish a review. I will pick one of the 50 Datasets and use PySpark to perform the ETL process to extract the dataset, transform the data, connect to an AWS RDS instance, and load the transformed data into pgAdmin. Then I will use Pandas to determine if there is any bias toward favorable reviews from Vine members in the dataset.


                  
## *Analysis & Results*
### Analysis
#### Amazon Data ETL

1) I will use Google Colab to perform the ETL Process, 
  a) will Load Amazon Data into Spark DataFrame
  
   ![AmazonData2DF](https://user-images.githubusercontent.com/80013773/124473267-f3f6e480-dd53-11eb-89d6-0c2b7266e6bd.PNG)


    b) Create DataFrames to create the following matching Tables:

        b1)    from pyspark.sql.functions import to_date
            # Read in the Review dataset as a DataFrame

            review_id_df = df.select(["review_id", "customer_id", "product_id", "product_parent", to_date("review_date", 'yyyy-MM-dd').alias("review_date")])
            review_id_df.show()

   ![AmazonData2_Review_id_DF](https://user-images.githubusercontent.com/80013773/124473422-29033700-dd54-11eb-9d54-6745c9c4247f.PNG)


        b2) # Create the customers_table DataFrame
            customers_df = df.groupby("customer_id").agg({"customer_id": "count"}).withColumnRenamed("count(customer_id)", "customer_count")
            customers_df.show()

   ![AmazonData2_customer_id_DF](https://user-images.githubusercontent.com/80013773/124473521-47693280-dd54-11eb-8282-9c9dc9acea71.PNG)

        b3) # Create the products_table DataFrame and drop duplicates. 
            products_df = df.select(["product_id", "product_title"]).drop_duplicates()
            products_df.show()
   ![AmazonData2_products_DF](https://user-images.githubusercontent.com/80013773/124473575-5bad2f80-dd54-11eb-9f7f-17897e41c142.PNG)


        b4) # Create the vine_table. DataFrame
            vine_df = df.select(["review_id", "star_rating", "helpful_votes", "total_votes", "vine", 'verified_purchase'])
            vine_df.show()

   ![AmazonData2_vine_DF](https://user-images.githubusercontent.com/80013773/124473654-6f589600-dd54-11eb-9694-8ce378b8b243.PNG)


2) Create PGAdmin Tables and Load the Data:

                    CREATE TABLE review_id_table (
            review_id TEXT PRIMARY KEY NOT NULL,
            customer_id INTEGER,
            product_id TEXT,
            product_parent INTEGER,
            review_date DATE -- this should be in the formate yyyy-mm-dd
            );

            -- This table will contain only unique values
            CREATE TABLE products_table (
            product_id TEXT PRIMARY KEY NOT NULL UNIQUE,
            product_title TEXT
            );

            -- Customer table for first data set
            CREATE TABLE customers_table (
            customer_id INT PRIMARY KEY NOT NULL UNIQUE,
            customer_count INT
            );

            -- vine table
            CREATE TABLE vine_table (
            review_id TEXT PRIMARY KEY,
            star_rating INTEGER,
            helpful_votes INTEGER,
            total_votes INTEGER,
            vine TEXT,
            verified_purchase TEXT
            );


3) Next I will Connect to the AWS RDS instance and write each DataFrame to its table.

            # Configure settings for RDS
            mode = "append"
            jdbc_url="jdbc:postgresql://<>:5432/<DataBaseName>"
            config = {"user":"postgres", 
                    "password": "<password>", 
                    "driver":"org.postgresql.Driver"}
            # Write review_id_df to table in RDS
            review_id_df.write.jdbc(url=jdbc_url, table='review_id_table', mode=mode, properties=config)
            # Write products_df to table in RDS
            products_df.write.jdbc(url=jdbc_url, table='products_table', mode=mode, properties=config)
            # Write customers_df to table in RDS
            customers_df.write.jdbc(url=jdbc_url, table='customers_table', mode=mode, properties=config)
            # Write vine_df to table in RDS
            vine_df.write.jdbc(url=jdbc_url, table='vine_table', mode=mode, properties=config)

#### Determine Bias of Vine Reviews
Usingh Pandas I will load CSV file and then perform the following steps:
1) filter By Votes:

        # Create a new DataFrame that retrieves all the rows where the total votes is equal to or greater than 20.
        df1 = vine_df.loc[vine_df["total_votes"] >= 20]
        df1
        -----------------------------------------------------------------------------------------------
        #  Create a new DataFrame that retrieves all the rows where 
        # the number of helpful votes divided by total votes is equal to or greater than 0.5
        df2 = df1.loc[(df1["helpful_votes"]/df1["total_votes"] >= 0.5)]
        df2
2) Analyze the Vine Reviews

        # Create a DataFrame that retrieves all the rows where a review was written as part of the Vine program (vine == Y).
        paid_df = df2.loc[df2["vine"]=="Y"]
        paid_df.head(10)

        # Create a DataFrame that retrieves all the rows where a review wasn't written as part of the Vine program (vine == N).
        unpaid_df = df2.loc[df2["vine"]=="N"]
        unpaid_df

3) Determine the percentage of five-star reviews among Vine reviews

        paid_five_star_number = paid_df.loc[(paid_df['star_rating']== 5)]["star_rating"].count()
        type(paid_five_star_number)
        # Retrieve the number of 5 star ratings from the DataFrame that has a written review.
        paid_five_star_number = paid_df.loc[(paid_df['star_rating']== 5)]["star_rating"].count()

        # Retrieve the total number of star ratings from the DataFrame that has a written review.
        paid_number = paid_df["star_rating"].count()

        # Calculate the percentage of five star reviews.
        percentage_five_star_vine = paid_five_star_number / paid_number * 100

        # Print the results. 
        print(paid_number)
        print(paid_five_star_number)
        print(f"{round(percentage_five_star_vine,2)}%")

4) Determine the percentage of five-star reviews among non-Vine reviews

        # Retrieve the number of 5 star ratings from the DataFrame that doesn't have a written review.
        unpaid_five_star_number = unpaid_df.loc[(unpaid_df['star_rating']== 5)]["star_rating"].count()

        # Retrieve the total number of star ratings from the DataFrame that doesn't have a written review.
        unpaid_number = unpaid_df["star_rating"].count()

        # Calculate the percentage of five star reviews.
        percentage_five_star_non_vine = unpaid_five_star_number / unpaid_number * 100

        # Print the results. 
        print(unpaid_number)
        print(unpaid_five_star_number)
        print(f"{round(percentage_five_star_non_vine,2)}%")

### Results

#### An Amazon Review dataset is extracted as a products Table in PgAdmin

![products_pgadmin](https://user-images.githubusercontent.com/80013773/124474375-4b498480-dd55-11eb-8123-2c8c076155b5.PNG)

#### An Amazon Review dataset is extracted as  a review_id Table in PgAdmin

![review_id_pgadmin_tablecount](https://user-images.githubusercontent.com/80013773/124474504-746a1500-dd55-11eb-9ff6-04196108ae1e.PNG)

#### An Amazon Review dataset is extracted as a customers Table in PgAdmin

![customer_pgadmin_table](https://user-images.githubusercontent.com/80013773/124474226-26551180-dd55-11eb-8f65-3232402cb0e6.PNG)


#### An Amazon Review dataset is extracted as a vine Table in PgAdmin

![vine_tabe_pgadmin](https://user-images.githubusercontent.com/80013773/124474630-9e233c00-dd55-11eb-9a40-0674967778f9.PNG)



#### Determine the percentage of five-star reviews among Vine vs. non-Vine reviews

Insert two images of the analysis 
  

    
## *Summary*
### Advantages
 Statistical Analysis is a must for any product line. it's used to evaluate and predict our Production Health, performance, and Quality. R.Studio is a powerful tool and is used for conducting such statistical analysis 
 
