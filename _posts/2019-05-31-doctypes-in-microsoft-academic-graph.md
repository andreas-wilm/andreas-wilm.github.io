---
layout: post
title: Document Types in Microsoft Academic Graph
subtitle: Baby steps with U-SQL and Databricks
#gh-repo: daattali/beautiful-jekyll
#gh-badge: [star, fork, follow]
#tags: [test]
comments: true
---

I've had a chance to play with [Microsoft Academic](https://academic.microsoft.com/) and [Microsoft Academic Graph (MAG)](https://www.microsoft.com/en-us/research/project/microsoft-academic-graph/) for a while and wanted to document some baby steps here .  Describing the two "services" in detail is stuff for another post, suffice it to say that [Microsoft Academic](https://academic.microsoft.com/) is is like [Google Scholar](https://scholar.google.com/), just richer (semantic search, annotation with topics inferred with NLP techniques etc.) and [MAG](https://www.microsoft.com/en-us/research/project/microsoft-academic-graph/) is the underlying data. The important point is that the MAG raw data is freely accessible via Azure, whereas Google Scholar is closed. 



To get started with MAG, follow the steps on how to [deploy the data on Azure storage](https://docs.microsoft.com/en-us/academic-services/graph/get-started-setup-provisioning). After that you will have roughly 500 GB of raw MAG data in your storage account. Next, set up [Azure Data Lake Analytics](https://docs.microsoft.com/en-us/academic-services/graph/get-started-setup-azure-data-lake-analytics) and you're good to go. In the following I describe how to summarize document types in MAG with U-SQL and also Azure Databricks. If you are looking for something more sophisticated, there are great [tutorials on how to compute the h-index](
https://docs.microsoft.com/en-us/academic-services/graph/#tutorials) with MAG on the official website. For more examples and comparison of MAG with other offerings, have a look at this [awesome list](https://github.com/andreas-wilm/awesome-academic-graph/blob/master/README.md). 


## Count Document Types with U-SQL

[U-SQL](https://docs.microsoft.com/en-us/u-sql/) is an SQL variant developed by Microsoft that can efficiently analyze data across relational stores, including data lakes, SQL databases etc. A nice U-SQL introduction can be found [here](https://devblogs.microsoft.com/visualstudio/introducing-u-sql-a-language-that-makes-big-data-processing-easy/). Using the Azure portal you can submit U-SQL batch jobs that scale instantly without having to manage any infrastructure.

To submit a job that summarizes document types in MAG, simply:

- Head over to the Azure Data Lake Analytics account you just created 
- Click on new job
- Give the job a name
- Copy and paste the code from this [Gist](https://gist.github.com/andreas-wilm/a409defc3ea526839af69e804d103575)
- Replace `blobAccount` and `dataVersion` values with your account details
- Change AU to, say, 2 (see [here for details on AUs](https://mitra.computa.asia/articles/msdn-understanding-adl-analytics-unit))
- Hit submit

That's it. To view the created output file, click on Data/Outputs and then the file itself (`TypeCounts.csv`). It will look roughly as follows:

![Screenshot for DLA Data Outputs](/img/2019-05-31-mag-doctype-outputs.png)

| Type | Counts |
|------|--------|
| Book | 1095185 |
| Journal |	82164726 |
| | 78297141 |
| Patent | 45486148 |
| BookChapter | 2544813 |
| Conference | 4385543 |
| Dataset |	39424 |


## Count Document Types with Azure Databricks

While the above is for batch jobs, you would use [Azure Databricks](https://azure.microsoft.com/en-us/services/databricks/) for more interactive work on Data Lake. Azure Databricks is an Apache Spark based analytics platform optimized for Azure. It features interactive workspaces with Jupyter-style notebooks, automated Spark cluster management and effortless integration with a wide variety of data stores and services. Follow [these simple steps](https://docs.microsoft.com/en-us/academic-services/graph/get-started-setup-databricks) to create a workspace and an autoscaling Spark cluster. Below is a screenshot showing how effortless the cluster creation is:

![Screenshot of Databricks cluster creation](/img/2019-05-31-databricks-cluster.png)

That's really all there is to it. 

With this we'll repeat the same analysis as above just in Python/Pyspark. First, [import PySparkMagClass.py](https://docs.microsoft.com/en-us/academic-services/graph/tutorial-azure-databricks-hindex#import-pysparkmagclasspy-as-a-notebook)  (this provides a convenience class called `MicrosoftAcademicGraph`) into a newly created notebook (directly attached to your just created Spark cluster) and follow the steps there down to the section called
[Define configuration variables](https://docs.microsoft.com/en-us/academic-services/graph/tutorial-azure-databricks-hindex#define-configration-variables)

Now create a MAG instance, load the 'Papers' dataframe and list the first entries:

```
mag = MicrosoftAcademicGraph(container=MagContainer, account=AzureStorageAccount, key=AzureStorageAccessKey)
papers = mag.getDataframe('Papers')
papers.show(10)
```

![Screenshot of loading of Papers Dataframe](/img/2019-05-31-mag-papers.png)


Now extract the document types:

```
docTypeCounts = papers.select(papers.DocType).groupBy(papers.DocType).count()
display(docTypeCounts.na.fill('Others').orderBy("count"))
```

The data is displayed as table by default:

![Screenshot of the table](/img/2019-05-31-mag-doctypes-table.png)

 But the cool thing is that you can immediately convert it into a variety of plots by clicking on the plot icon (second left on bottom):

![Screenshot of the pie chart](/img/2019-05-31-mag-doctypes-piechart.png)

And that's all I wanted to show.