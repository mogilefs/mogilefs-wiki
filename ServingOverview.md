# Quick Overview of Serving a File

Storing and maintaining your MogileFS cluster is the first step.  The last step is to serve the files back out to your users.

The most typical setup is a perlbal front-end that has a pool of application servers that deliver the data from MogileFS.  Here is the basic path to get from an incoming user request that results in them getting a file out of MogileFS:

## 1. GET /some/path/somefile-123.jpg arrives at Perlbal ##

> For the sake of simplicity, we are not going to address caching (See ServingRecipes).  When Perlbal receives the request, it will replay it to the cluster of backend application servers.

## 2. GET /some/path/somefile-123.jpg arrives at your application ##
> The application receives a request for the asset URL, and because you programmed the application you know that it's in MogileFS.  So you translate it into the proper path and send the request to the tracker.

## 3. get\_paths somefile:123 command arrives at your MogileFS tracker ##
> The tracker then finds the relevant paths to somefile:123, and returns those back to your application.

## 4. The application now knows the locations to get the file ##

> Once the application has the list of internal URIs to fetch somefile:123 from, such as http://10.0.0.34:8034/dev1/0/0/0/0.fid (plus another URL or two), it simply has to set the proper headers for perlbal to fetch and return the file data from the MogileFS storage nodes.  In the easiest case, set the content type header and the "X-REPROXY-URL" header to one of the URIs returned from the tracker.

## 5. Perlbal gets back the response from your webserver ##

> This URL is now retrieved internally and Perlbal starts reproxying it.  The headers returned to the user are some combination of the response from your webserver and the response from the storage node.6) User gets their file.

## 6. The user gets the file ##

> And done!