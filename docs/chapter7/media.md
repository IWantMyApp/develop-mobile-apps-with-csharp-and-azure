# Azure Media Services

One of the common use cases for mobile applications involves video streaming.  In the consumer space, this
can include applications like Hulu or Netflix along with video reviews and new segments in apps like CNN and
CNET.  In the enterprise space, we see video learning and employee broadcast solutions.

Whatever their source, they have some basic functionality in common:

* The video asset is uploaded and converted (also known as encoding) to a streaming format.
* A live video channel can be provided for multiplex streaming to a large audience.
* The encoded video is provided to clients for download with a suitable web endpoint.
* Additional services extract information from the video for search capabilities.

Many enterprises wrap such functionality in a combined web and mobile site to provide streaming video for
eLearning.  We are going to look at what it takes to produce the mobile side of such a site in this section.

## The Video Search Application with Media Services

We are going to produce a media services mobile application for this section, based on our last example for
Azure Search.  In the new example, this is the approximate flow of the application:

![][img1]

The administrator will upload an MP4 video via the Visual Studio Storage Explorer.  This will be Automatically
picked up by an Azure Function that will extract information from the MP4 file (such as title, release year,
etc.) and place that information into a queue element.  The queue element will be used to drive another Azure
service - Logic Apps.  Logic Apps are used to marshal sequential tasks in a workflow.  In our case, the workflow
will use Media Services to process the video and produce a streamable version.  Once that is done, we will use
Cognitive Services to convert the audio and video into searchable textual content, which will be submitted to
an Azure Search instance.  Finally, we will insert a record into our SQL databse for Azure Mobile Apps to expose
the video to our clients.

On the client side, we will use the Azure Search instance to find apps, display the information held within
Azure Mobile Apps, and allow the user to stream the video using the video player.

As you can see, there are many more services in use in this example than our previous examples:

* [Azure Media Services] is used for video encoding and streaming endpoints.
* [Azure Logic Apps] is used for workflow automation.
* [Azure Functions] are used for automated individual steps.
* [Cognitive Services] are used to extract information from the videos.
* [Azure App Service] is used to act as a coordinator for the mobile app.
* [Azure Search] is used as our full text search engine.
* [Azure Storage] is used to store the individual video assets and for some queuing capabilities.
* [SQL Azure] is used as the backing store for the Azure Mobile Apps data store.

This is now a fairly typical mobile application.  We are using 8 different Azure services in a composite manner
to provide the facilities needed by our application.

## Creating the Mobile Encoding flow

When I look at the architecture for our mobile backend, I see two distinct parts.  The first is the backend flow
that processes the incoming videos.  As videos are uploaded, they need to be injected into a queue.  From there,
a series of processes are kicked off to process the incoming video.  First, the video is encoded; then data is
extracted from the video for search purposes; finally, the video is added to the SQL database so it can be searched.

The other flow is from the mobile client - it connects to the App Service and makes requests based on what it needs
to do.  In this case, we have a set of data tables for providing data about the video and a few custom APIs for
handling search and video streaming.

Let's take each of these in turn.   The configuration of most of the services  have already been discussed, so I
will not go over them and only provide the options I used.  This includes Storage, Search, SQL Azure, and Functions.

### Creating pre-requisite services

Before I start with the new services, I need an [Azure Storage] account, an [Azure Search] instance and
an [Azure Function App].  I've covered all these items in previous sections, so I won't go into them here.
The configuration is relatively simple:

* My [Azure Storage] account called `zumomediach7.core.windows.net` as **General Purpose** storage with **LRS** replication.
* My [Azure Search] instance called `zumomediach7.search.windows.net` in the **Free** pricing tier.
* My [Azure Functions] app called `zumomediach7-functions.azurewebsites.net` in the **Consumption Plan**.  I'm
   using my `zumomediach7` storage account.
* My [SQL Azure] service is called `zumomediach7.database.windows.net`.
* My [SQL Azure] database is called `videosearch` in the **B Basic** pricing plan.
* My [Azure App Service] is created via the **Mobile App** template and called `zumomediach7.azurewebsites.net`.  It
    has an **B1 Basic** app service plan associated with it.

In addition, I've linked the SQL Azure database and storage accounts to the App Service via the Data Connections menu
option.  I've also added a query key to my Azure Search instance and provided a pair of App Settings in
the App Service - `SEARCH_APIKEY` holds the query key and `SEARCH_ENDPOINT` holds the URI of my service.

Our resource group looks quite extensive now:

![][img2]

Configuration for the services is as follows:

**Azure Storage** has a container for incoming videos called `incoming`.

**Azure Search** has an index with the following fields:

| Field Name | Type | Attributes |
| --- | --- | --- |
| videoId | Edm.String | Key, Retrievable |
| audio | Edm.String | Retrievable, Filterable, Searchable |

The `audio` field is new and will be populated with the textual content from analysis of the video.

**Azure App Service** has a basic TableController which is based on the following DTO model:

```csharp
using Microsoft.Azure.Mobile.Server;

namespace Backend.DataObjects
{
    public class Video : EntityData
    {
        public string Filename { get; set; }

        public string VideoUri { get; set; }
    }
}
```

I also have a custom API that returns my search settings.  I'll add any settings I need to transmit to my
mobile app into this same controller so that they can easily be retrieved from the mobile app:

```csharp
using System;
using System.Web.Http;
using Microsoft.Azure.Mobile.Server.Config;

namespace Backend.Controllers
{
    [MobileAppController]
    public class SettingsController : ApiController
    {
        private Controllers.Settings _pSettings;

        public SettingsController()
        {
            _pSettings = new Controllers.Settings
            {
                SearchApiKey = Environment.GetEnvironmentVariable("SEARCH_APIKEY"),
                SearchEndpoint = Environment.GetEnvironmentVariable("SEARCH_ENDPOINT")
            };
        }

        // GET api/Settings
        public Controllers.Settings Get()
        {
            return _pSettings;
        }
    }

    public class Settings
    {
        public string SearchEndpoint { get; set; }
        public string SearchApiKey { get; set; }
    }
}
```

This encompasses information from the majority of the book thus far.  If you are uncertain on how to perform any
of this configuration, review the appropriate sections of the book:

* [Chapter 1](../chapter1/firstapp_pc.md) covers creating a Mobile App.
* [Chapter 4](../chapter4/functions.md) covers Azure Functions.
* [This chapter](./search.md) covers Azure Search.

### Creating an Azure Media Services account

So far, we've done a lot of infrastructure work.  We've generated an Azure Mobile App that our mobile app can use
to retrieve information about the videos, generated an Azure Search instance with a suitable index, and a storage
account for processing the videos.  We now want to move onto the meat of this section - working with video.  In
order to do that, we will need an Azure Media Services account.

Creating an Azure Media Services account is very similar to other Azure resources.  Log in to the [Azure Portal]
and open the resource group you are using to hold all the resources for this application.

* Click **+ ADD** to add a resource to the resource group.
* Search for **Media Services**, select it, then click on **Create**.
* Fill in the form:
    1. Select a name (I used `zumomediach7`).
    2. Select your existing storage account (note the limitations on the replication policy if you use your own).
    3. Ensure the region matches your storage account and other resources.
* Click **OK**.

![][img3]

The Media Services accounts may take a couple of minutes to create.  Do not continue until the deployment is complete.

You could stop here and do all the work manually.  If you wish to check out the full set of tutorials, follow the
official documentation:

* [Uploading Assets](https://docs.microsoft.com/en-us/azure/media-services/media-services-portal-upload-files)
* [Encoding Assets](https://docs.microsoft.com/en-us/azure/media-services/media-services-portal-encode)
* [Publish Assets](https://docs.microsoft.com/en-us/azure/media-services/media-services-portal-publish)

### The Azure Functions

We are expecting a completely automated workflow for processing our video assets.  As a result, we are going to
need some basic processing blocks.  Per our original specification, we need three processing blocks.  The first
block will take the uploaded media file and encode it for later consumption, passing the URI and other associated
information to the next block.  The second block will then extract information we want to search on and insert
that into Azure Search.  We'll deal with this block in the next section.  Our final block will insert the details
for the video into the database so that it can be exposed by Azure Mobile Apps.  We are only going to deal with
the first and third blocks in this section.

You can find a huge number of samples for doing various things with Azure Functions and Media Services on their
[Samples GitHub repository].  We are going to base the encoding block on the [EncodeBlob\_MultiOut\_Function][1]
sample.  This starts when the Logic App triggers based on a blob being uploaded.  It then triggers the following
Azure Function:

```csharp

```


** TO BE CONTINUED **



<!-- Images -->
[img1]: ./img/media-plan.PNG
[img2]: ./img/media-rg-view.PNG
[img3]: ./img/media-create.PNG

<!-- Azure Service Definition Overviews -->
[Azure Media Services]: https://docs.microsoft.com/en-us/azure/media-services/media-services-concepts
[Azure Logic Apps]: https://docs.microsoft.com/en-us/azure/logic-apps/logic-apps-what-are-logic-apps
[Azure Functions]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview
[Cognitive Services]: https://azure.microsoft.com/en-us/services/cognitive-services/
[Azure App Service]: https://docs.microsoft.com/en-us/azure/app-service/app-service-value-prop-what-is
[Azure Search]: https://docs.microsoft.com/en-us/azure/search/search-what-is-azure-search
[Azure Storage]: https://docs.microsoft.com/en-us/azure/storage/storage-introduction
[SQL Azure]: https://docs.microsoft.com/en-us/azure/sql-database/sql-database-technical-overview
[Azure Function App]: https://docs.microsoft.com/en-us/azure/azure-functions/functions-overview
[Samples GitHub repository]: https://github.com/Azure-Samples/media-services-dotnet-functions-integration
[1]: https://github.com/Azure-Samples/media-services-dotnet-functions-integration/tree/master/EncodeBlob_MultiOut_Function
