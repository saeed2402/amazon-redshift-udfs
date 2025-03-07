# Amazon Redshift UDFs
A collection of stored procedures and user-defined functions (UDFs) for Amazon Redshift. The intent of this collection is to provide examples for defining useful functions which extend Amazon Redshift capabilities and support migrations from legacy DB platforms.

## Stored Procedures
Each procedure is allocated a folder.  At minimal each procedure will have a <procedure_name>.sql file which you may use to deploy the procedure.  Optionally, it may contain a README.md file for instructions on how to use the procedure and additional files used for test the procedure.

## UDFs
Each function is allocated a folder.  At minimal each function will have the following files which will be used by the [deployFunction.sh](#deployFunctionsh) script and [testFunction.sh](#testFunctionsh) scripts:

- **function.sql** - the SQL script to be run in the Redshift DB which creates the UDF.  If a Lambda function, use the string `:RedshiftRole` for the IAM role to be passed in by the deployment script.
- **input.csv** - a list of sample input parameters to the function, delimited by comma (,) and where strings are denoted with single-quotes.
- **output.csv** - a list of expected output values from the function.

### python-udfs

[Python UDFs](https://docs.aws.amazon.com/redshift/latest/dg/udf-creating-a-scalar-udf.html) may include the following additional file:

- **requirements.txt** - If your function requires modules not available already in Redshift, a list of modules.  The modules will be packaged, uploaded to S3, and mapped to a [library](https://docs.aws.amazon.com/redshift/latest/dg/r_CREATE_LIBRARY.html) in Redshift.  

### lambda-udfs

[Lambda UDFs](https://docs.aws.amazon.com/redshift/latest/dg/udf-creating-a-lambda-sql-udf.html) may include the following additional file:

- **lambda.yaml** - [REQUIRED] a CFN template containing the Lambda function.  The lambda function name should match the redshift function name with '_' replaced with '-'  e.g. (f-upper-python-varchar). The template may contain additional AWS services required by the lambda function and should contain an IAM Role which can be assumed by the lambda service and which grants access to those additional services (if applicable).  In a production deployment, be sure to add resource restrictions to the Lambda IAM Role to ensure the permissions are scoped down.

- **resources.yaml** - a CFN template containing external resources which may be referenced by the Lambda function.   These resources are for testing only.

- **package.json** - (NodeJS Only) If your function requires modules not available already in Lambda, a list of modules.  The modules will be packaged, uploaded to S3, and mapped to your Lambda function. See [f_mysql_lookup_nodejs](lambda-udfs/f_mysql_lookup_nodejs-varchar-varchar-varchar) for and example.  

- **index.js** (NodeJS Only) your javascript handler code.  See [f_upper_nodejs](lambda-udfs/f_upper_nodejs-varchar) for and example.  

- **pom.xml** - (Java Only) Lists out dependencies as well as the name of your handler function. See [f_upper_java](lambda-udfs/f_upper_java-varchar) for and example.  See [Maven Documentation](https://maven.apache.org/guides/introduction/introduction-to-the-pom.html) for more details on writing a pom.xml file.

- **src/main/java/`function`/Handler.java** - (Java Only) your java handler code. See [f_upper_java](lambda-udfs/f_upper_java-varchar) for and example.  

### sql-udfs
[SQL UDFs](https://docs.aws.amazon.com/redshift/latest/dg/udf-creating-a-scalar-sql-udf.html) do not require any additional files.

## Deployment & Testing
Located in the `bin` directory are tools to deploy and test your UDF functions.  

### deployFunction.sh
This script will orchestrate the deployment of the UDF to your AWS environment. This includes
1. Looping through modules in a `requirements.txt` file (if present) and installing them using the `libraryInstall.sh` script by uploading the packages to the `$S3_LOC` and creating the library in Redshift using the `$REDSHIFT_ROLE`.
2. If deploying a nodeJS lambda UDF, using `package.json` to run `npm install` packaging the code and uploading the `zip` file to the `$S3_LOC`.
3. If deploying a Java lambda UDF, using `pom.xml` to run `mvn package` packaging the code and uploading the `jar` to the `$S3_LOC`.
4. If deploying a lambda UDF, using `lambda.yaml` to run `aws cloudformation deploy` and build the needed resources.
5. Creating the UDF function in Redshift by executing the `function.sql` sql script.  If deploying a lambda UDF, replacing the `:RedshiftRole` parameter.

```
./deployFunction.sh -t lambda-udfs -f "f_upper_python(varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA -r $REDSHIFT_ROLE

./deployFunction.sh -t python-udfs -f "f_ua_parser_family(varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA -r $REDSHIFT_ROLE -s $S3_LOC

./deployFunction.sh -t sql-udfs -f "f_mask_varchar(varchar,varchar,varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA
```

### testFunction.sh
This script will test the UDF by
1. Creating a temporary table containing the input parameters of the function.
2. Loading sample input data of the function using the `input.csv` file.  
3. Running the function leveraging the sample data and comparing the output to the `output.csv` file.

```
./testFunction.sh -t lambda-udfs -f "f_upper_python(varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA

./testFunction.sh -t python-udfs -f "f_ua_parser_family(varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA

./testFunction.sh -t sql-udfs -f "f_mask_varchar(varchar,varchar,varchar)" -c $CLUSTER -d $DB -u $USER -n $SCHEMA
```

## Contributing / Pull Requests

We would love your contributions.  See the [contributing](contributing.md) page for more details on creating a fork of the project and a pull request of your contribution.

> Pull requests will be tested using a Github workflow which leverages the above testing scripts. Please execute these script prior to submitting a pull request to ensure the request is approved quickly.  When executed in the test enviornment the [RedshiftRole](#redshift-role) will be defined as follows. You can create a similar role in your local environment for testing.

##Appendix

### Redshift Role
For Lambda UDFs, These privileges ensure UDFs can invoke the Lambda Function as well as access the uploaded `*.whl` files located in s3.  
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "lambda:InvokeFunction"
            ],
            "Resource": [
                "arn:aws:lambda:*:*:function:f-*",
                "arn:aws:s3:::<test bucket>/*"
            ]
        }
    ]
}
```
