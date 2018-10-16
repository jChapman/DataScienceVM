# Running batch jobs on Data Science VM
In machine learning and data science, there are many common scenarios where you have to run programs or scripts in a batch.  Some examples of batch executions are recurring data pre-processing, model retraining, batch scoring. 
Here is a way you can develop your code on a [Data Science Virtual Machine (DSVM)](http://aka.ms/dsvmdoc) and setup batch execution using the [Azure Batch service](https://docs.microsoft.com/azure/batch/) where the execution happens on a similar DSVM environment
that you used for development. In this walkthrough, we go through steps to execute an R program as a very basic batch execution. 

The steps to do this are as follows:

1. Develop and test your code (in R, Python, Julia, C# etc) in the normal way on the DSVM using the rich prebuilt tools and frameworks. This code typically takes a data file that is used to process as a batch. 

2. Create an Azure batch account on your Azure  Subscription if you dont have one. You can run these commands on  your desktop if  you have Azure CLI installed and logged into your Azure account on the CLI. Otherwise you can also use the Cloud Shell on the Azure portal which will give you a predefined environment that is already logged into your subscription.  


```
# NOTE: Replace all placeholder "dsvmbatch" below with values unique to your application
az group create --name dsvmbatch --location westus2
az batch account  create --name dsvmbatch --resource-group dsvmbatch --location westus2
az batch account login --name dsvmbatch --resource-group dsvmbatch --shared-key-auth
```
3. Create a Batch Pool. You can choose the VM image that is used as base for the Batch Pool nodes. 
Here we choose the Windows 2016 DSVM as the pool node VM image. You can use Linux edition of the DSVM by specifying the corresponding image URN in the image parameter. You can also change the size of the VM to the desired size including using GPU based instances (NC-Series or ND Series VMs on Azure). 

```
az batch pool create \
    --id mypool --vm-size Standard_D2s_v3 \
    --target-dedicated-nodes 1 \
    --image microsoft-dsvm:dsvm-windows:server-2016 \
    --node-agent-sku-id "batch.node.windows amd64" 

az batch pool show --pool-id mypool --query "allocationState"

# Wait till state is steady
```
Tip: If using AZ CLI on a Windows machine use the ^ character instead of \ to continue command on next line.

4. Create a job and task on the Batch Pool. In example below, we just use a simple predefined sample R program for Tensorflow  built into the Windows 2016 DSVM and invoked through the Rscript command. You can run pretty much anything that you can invoke from command line as long it does not require any user interaction like a Python script, Julia code, an compiled binary. 
```
az batch job create  --id myjob --pool-id mypool
az batch task create --task-id mytask --job-id myjob --command-line "cmd /c ""rscript ""c:\dsvm\samples\R-Tensorflow\linear_regression_simple .R"""""	
```
5. Check for the status of the task

```
az batch task show --job-id myjob --task-id mytask
```

6. When the task has finished, list all the artifacts generated by the batch job and download them. In this case the program outputs result to standard out. In your case you may choose to write output to a Azure blob and just write a status message on standard output. 

```
# list all artifacts of the task
az batch task file list  --job-id myjob --task-id mytask -o table
# Download one of more artifacts. Here we download the stdout which is written to stdout.txt. 
az batch task file download   --job-id myjob --task-id mytask --file-path stdout.txt --destination ./stdout.txt
```
7. Delete your pool if you no longer need to execute any other tasks. You are charged for pools while the nodes are running, even if no jobs are scheduled. There is no charge for the batch account itself. 

```
az batch pool delete --pool-id mypool
# NOTE: Instead of deleting the pool you can also resize to 0 nodes to avoid compute charges and later resize the pool back to desired size when you are ready to run another task
# az batch pool resize --target-dedicated-nodes 0 --pool-id mypool
```

This tip just demonstrates a very basic usage of Batch to help get started easily. We used the CLI for managing the batch. However you can use the SDK in Python, node.js, .Net or access through REST API in language of your choice. There are other features of Batch like auto scaling, low priority nodes (which cost significantly lesser than dedicated nodes). 
More information  can be found on the [Azure Batch service](https://docs.microsoft.com/azure/batch/) documentation page. 

The Azure batch also makes it easy to submit tasks to run the batch execution from within a data or ML pipeline built on [Azure Data Factory Custom Activity](https://docs.microsoft.com/azure/data-factory/transform-data-using-dotnet-custom-activity). 
