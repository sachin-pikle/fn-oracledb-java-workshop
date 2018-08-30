# Fn with Oracle DB

Fn functions executing `CRUD` operations on Oracle DB. This sample uses a simple `Employee` entity for demonstration purposes 

> For the purposes of this lab, an [Oracle Database instance on Oracle Cloud Infrastructure](https://docs.cloud.oracle.com/iaas/Content/Database/Concepts/databaseoverview.htm?tocpath=Services%7CDatabase%7C_____0) has already been provisioned for you along with the associated schema and table

## Setup

### Build the (base) Docker image containing Oracle JDBC driver

- Clone this repo - `git clone https://github.com/abhirockzz/fn-oracledb-java`
- download the Oracle JDBC driver from [this link](https://www.oracle.com/technetwork/database/features/jdbc/default-2280470.html) (`ojdbc7.jar` should be fine) and copy it to the `oracle_driver_docker` folder
- Build a Docker image with the driver JAR (You will use an existing Dockerfile)
	- `cd fn-oracledb-java/oracle_driver_docker`
	- `docker build -t oracle_jdbc_driver_docker .` (if you choose to change the name of the image i.e. `oracle_jdbc_driver_docker`, you'll need to update those references in the `build_image` section of the `func.yaml` for all the functions)

(if successful) You should see an output as below

	Sending build context to Docker daemon  3.401MB
	Step 1/3 : FROM fnproject/fn-java-fdk-build:jdk9-1.0.63
	 ---> 973847bef180
	Step 2/3 : COPY ojdbc7.jar .
	 ---> dd9005799e07
	Step 3/3 : RUN mvn deploy:deploy-file -Durl=file:///function/repo -Dfile=ojdbc7.jar -DgroupId=com.oracle -DartifactId=ojdbc7 -Dversion=12.1.0.1 -Dpackaging=jar
	 ---> Running in 94bce78bf6bb
	[INFO] Scanning for projects...
	Downloading from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-clean-plugin/2.5/maven-clean-plugin-2.5.pom
	Downloaded from central: https://repo.maven.apache.org/maven2/org/apache/maven/plugins/maven-clean-plugin/2.5/maven-clean-plugin-2.5.pom (3.9 kB at 1.6 kB/s)
	
	............ skipping all the maven magic ..............
	
	Uploading to remote-repository: file:///function/repo/com/oracle/ojdbc7/12.1.0.1/ojdbc7-12.1.0.1.jar
	Uploaded to remote-repository: file:///function/repo/com/oracle/ojdbc7/12.1.0.1/ojdbc7-12.1.0.1.jar (3.4 MB at 6.3 MB/s)
	Uploading to remote-repository: file:///function/repo/com/oracle/ojdbc7/12.1.0.1/ojdbc7-12.1.0.1.pom
	Uploaded to remote-repository: file:///function/repo/com/oracle/ojdbc7/12.1.0.1/ojdbc7-12.1.0.1.pom (392 B at 49 kB/s)
	Downloading from remote-repository: file:///function/repo/com/oracle/ojdbc7/maven-metadata.xml
	Uploading to remote-repository: file:///function/repo/com/oracle/ojdbc7/maven-metadata.xml
	Uploaded to remote-repository: file:///function/repo/com/oracle/ojdbc7/maven-metadata.xml (302 B at 22 kB/s)
	[INFO] ------------------------------------------------------------------------
	[INFO] BUILD SUCCESS
	[INFO] ------------------------------------------------------------------------
	[INFO] Total time: 27.221 s
	[INFO] Finished at: 2018-08-30T02:08:09Z
	[INFO] ------------------------------------------------------------------------
	Removing intermediate container 94bce78bf6bb
	 ---> fc46eb9cc4de
	Successfully built fc46eb9cc4de
	Successfully tagged oracle_jdbc_driver_docker:latest

### Start Fn

- (if you haven't already) Start local Fn server - `fn start`
- Switch context - `fn use context default`

### Configure Docker

> This is not required if you don't want to push the function Docker images to an external Docker registry

- `docker login` (use your docker registry credentials)
- `export FN_REGISTRY=<name of your docker repository>`


### Create an app with required database configuration

`fn create app --config DB_URL=jdbc:oracle:thin:@//129.213.138.81:1521/workshop_iad1hs.sub08241855250.workshoplisbonv.oraclevcn.com --config DB_USER=workshopNNN --config DB_PASSWORD=<password> fn-oradb-java-app` 

> Please replace `workshopNNN` in the `DB_USER` attribute with your username (**no hyphen**) and your database password (provided by the lab instructor) in the `DB_PASSWORD` attribute. Please leave the other parameters unchanged since they have been pre-configured based on the lab environment

## Deploy

Deploy one function at a time. For example, to deploy the `create` function

- `cd fn-oracledb-java/create`
- `fn -v deploy --app fn-oradb-java-app --local --no-bump` (`-v` will activate verbose mode)

> Make sure you do not miss the `--local` option without which Fn will try to push the function Docker images to an external Docker registry

For `read` function deployment

- `cd ../read`
- `fn -v deploy --app fn-oradb-java-app --local --no-bump`

> Repeat for other functions i.e. `delete` and `update`

- Run `fn inspect app fn-oradb-java-app` to check your app (and its config)

		{
		        "config": {
		                "DB_PASSWORD": "t0ps3cr3t",
		                "DB_URL": "jdbc:oracle:thin:@//129.213.138.81:1521/workshop_iad1hs.sub08241855250.workshoplisbonv.oraclevcn.com",
		                "DB_USER": "workshop130"
		        },
		        "created_at": "2018-08-30T07:08:35.791Z",
		        "id": "01CP4TK42FNG8G00GZJ0000002",
		        "name": "fn-oradb-java-app",
		        "updated_at": "2018-08-30T07:24:15.182Z"
		}

- Run `fn list functions fn-oradb-java-app` to check associated functions

		NAME                            IMAGE
		fn-oradb-app-create-func        fn-oradb-app-create-func:0.0.2
		fn-oradb-app-delete-func        fn-oradb-app-delete-func:0.0.1
		fn-oradb-app-read-func          fn-oradb-app-read-func:0.0.1
		fn-oradb-app-update-func        fn-oradb-app-update-func:0.0.1

> A custom Docker image has been used as `build_image` (see `func.yaml`) - this Docker image pre-packages the Oracle JDBC driver

## Test

.. with Fn CLI using `fn invoke`

### Create

`echo -n '{"emp_email": "a@b.com","emp_name": "abhishek","emp_dept": "Product Divison"}' | fn invoke fn-oradb-java-app fn-oradb-app-create-func`

If successful, you should a response similar to this `Created employee CreateEmployeeInfo{emp_email=a@b.com, emp_name=abhishek, emp_dept=Product Divison}`

Create as many as you want - make sure that the `emp_email` is unique

### Read

- `fn invoke fn-oradb-java-app fn-oradb-app-read-func` (to fetch all employees)

You should get back a JSON response similar to below

	[
	  {
	    "emp_email": "y@z.com",
	    "emp_name": "abhishek",
	    "emp_dept": "PM"
	  },
	  {
	    "emp_email": "a@b.com",
	    "emp_name": "abhishek",
	    "emp_dept": "Product Divison"
	  },
	  {
	    "emp_email": "x@y.com",
	    "emp_name": "kehsihba",
	    "emp_dept": "QA Divison"
	  }
	]

- `echo -n 'a@b.com' | fn invoke fn-oradb-java-app fn-oradb-app-read-func` (to fetch employee with email `a@b.com`)

		[
		  {
		    "emp_email": "a@b.com",
		    "emp_name": "abhishek",
		    "emp_dept": "Product Divison"
		  }
		]

### Update

It is possible to update the department of an employee

`echo -n '{"emp_email": "a@b.com", "emp_dept": "Support Operations"}' | fn invoke fn-oradb-java-app fn-oradb-app-update-func`

Successful invocation will return back a message similar to `Updated employee UpdateEmployeeInfo{emp_email=a@b.com, emp_dept=Support Operations}`

Check to make sure - `echo -n 'a@b.com' | fn invoke fn-oradb-java-app fn-oradb-app-read-func` - the updated department should reflect

		[
		  {
		    "emp_email": "a@b.com",
		    "emp_name": "abhishek",
		    "emp_dept": "Support Operations"
		  }
		]

### Delete

Use employee email to specify which employee record you want to delete

`echo -n 'a@b.com' | fn invoke fn-oradb-java-app fn-oradb-app-delete-func` and you should see `Deleted employee a@b.com` message

Check to make sure - `echo -n 'a@b.com' | fn invoke fn-oradb-java-app fn-oradb-app-read-func`
