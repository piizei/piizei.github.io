#  Azure Data Explorer with Remote Sensing Data
## Why would anyone...

I was looking for a convenient way to query and aggregate raster data (e.g. satellite images) with Azure PaaS services. Yes I know there is plenty of other cloud services, open-source tools and commercial products that are well suited for this. But that's not the point here :) I decided to give Azure Data Explorer (ADX) a try with the problem as I previously had great experiences with it.

## Finding data to ingest.

This example uses  [MOD13A1.006](https://lpdaac.usgs.gov/products/mod13a1v006/) from NASA that contains [Normalized Difference Vegetation Index (NDVI)](https://en.wikipedia.org/wiki/Normalized_difference_vegetation_index) values for the whole globe. I chose it mainly because it's easily available as [Cloud Optimized GeoTiff](https://www.cogeo.org/) from AWS S3. The approach should work with any raster data however. 

Because each image took so much space and after conversion even more, I used a VM in Azure to do the job.
For this VM I installed following tools:
* GDAL 2.4.2
* AWS CLI
* AZ CLI
* proj, needed in Appendix 1 for the cs2cs tool

### Extract

The source I used is https://modis-vi-nasa.s3-us-west-2.amazonaws.com/ and you can start by copying example image with following command:

`aws s3 cp s3://modis-vi-nasa/MOD13A1.006/2018.03.22.tif ./`

Each image is about 1.5Gb  and present NDVI measurements of the whole globe. Each pixel in the image is a value between 0 and 1. There are areas that were not measured by the satellite. These are presented with value -3000. 

### Transform

To ingest the image into ADX, it needs to be in text/row format. I used GDAL to convert the Tiff into CSV file with following command:

`gdal_translate -of xyz -co ADD_HEADER_LINE=NO -co COLUMN_SEPARATOR="," 2018.03.22.tif /vsistdout | grep -v "\-3000" > 2018.03.22.csv`

The pipe with grep in the command is about filtering the 'non values' out. Otherwise the result will get really big.
The gdal_translate AFAIK does not work with multiple bands when using xyz translation. In this case you need either convert each band to a different file and then merge them, or just write a python script that does the transformation using the GDAL library (or find one from Github).

GDAL will generate some unwanted content in the beginning of the file which you can remove with this command:

`ex -sc '1d1|x' 2018.03.22.csv`

You will likely notice that the file is really big and any operations are going to take lot of time :)

ADX does not ingest files larger than 2G. With `ls -lah` we can see that our CSV file is already 21G. To fix that and make following operations faster, let's split the file to smaller chunks. To use `split`command, we need to first know how many lines are there in the file. You can count number of the lines relatively fast with command:

`dd if=2018.03.22.csv bs=256M | wc -l`

If we split the result of 352424394 lines with 20 then we should get number of lines we need for about 1G chunks. 10000000 is close enough :).  To split the file with this number, use command:

`split -l 10000000 2018.03.22.csv 2018.03.22- --additional-suffix=.csv`

Next we add date column into the CSV file for later use in ADX. The date must be in a format recognized by ADX ([ISO 8601](https://www.iso.org/iso-8601-date-and-time-format.html)). Here we use 2018-03-22. 

`sed -i "s/$/,2018-03-22/" 2018.03.22-*.csv`

That is the transformation done. Finally let's archive the original CSV file, so that it does not affect the commands executed later.
`gzip 2018.03.22.csv`

### Load
Let's push the files into Azure Blob Storage for ADX ingestion. First we need to create a storage account and there a blob storage container that we call 'modis' (or what ever you want :), just remember the name later ). [Check out these docs if you don't know how.](https://docs.microsoft.com/en-us/azure/storage/blobs/storage-quickstart-blobs-portal)

Then copy from the access-keys part of your storage account page (in azure portal) the connection string. Export this into your shell environment with:
`export AZURE_STORAGE_CONNECTION_STRING="THEKEYICOPIEDFROMAZUREPORTAL"`

With this, you can upload all the csv files into the container (named modis) with single command:
`az storage blob upload-batch -d modis -s . --pattern "*.csv"`

## Ingest to ADX

To start your ingest, [first create a ADX cluster and a database.](https://docs.microsoft.com/en-us/azure/data-explorer/create-cluster-database-portal) 

Create a Shared Access Signature (_SAS_) for your storage account. This is a URL that allows the ADX connect into your protected storage. Think it as a link that also contains a password. The page where you generate it looks like this:
![ebce02aa9cdac91922bc124328de1ee3.png](ebce02aa9cdac91922bc124328de1ee3.png)


Make sure you have selected all the resource types (Service, Container, Object) before clicking Generate button

Copy the 'Blob service SAS URL' that it generated. It should be something like https://storageaccountname.blob.core.windows.net/?sv=1203u120938LOTOFTHINGS
you need to append the name of your container (modis) to the end of the link, so that it looks like this:
https://storageaccountname.blob.core.windows.net/modis?sv=1203u120938LOTOFTHINGS

Then let's go to the newly created ADX:

Right click your database and select 'ingest new data'
![8ff9552cb5eb2c9df07b58df1e5e4d22.png](8ff9552cb5eb2c9df07b58df1e5e4d22.png)

This should open a screen like this:
![879e2c132e1062080f1717331dcd3b6e.png](879e2c132e1062080f1717331dcd3b6e.png)

Here select source type 'from container' instead of blob. Add your modified _SAS url_ into the _link to storage_ field.
Set _Sample Size_ to be larger than number of files you uploaded to the storage. The files from your storage should appear into the box beneath. Select one of the files, and click *Edit Schema*.

The default settings in Schema should be ok. Set the datatype of Column4 to datetime, if its not automatically correct.
![7467aea664d45fab62c426952a866260.png](7467aea664d45fab62c426952a866260.png)
After this let's wait a bit and then run some queries.

## Query and aggregate 
I draw a random square (with geojson.io) near _Curry_ that looks like this: 

![05e1a40d2cb5a0bb85ce1ff32e1de279.png](05e1a40d2cb5a0bb85ce1ff32e1de279.png)

The polygon is overlaid with the NDVI values from the raster. You can see that the pixels in the raster are not straight squares. This is because how the satellite is positioned. [Find out more here.](https://modis-land.gsfc.nasa.gov/MODLAND_grid.html)

How to find out what is the average NDVI over this region between all the dates we uploaded?
Here is the query that will do that for you.
```sql
let curry = dynamic({  
        "type": "Polygon",  
        "coordinates":  [[[-87.21136093139648, 33.87953701355924], 
                          [-87.0663070678711, 33.87953701355924],
                          [-87.0663070678711, 33.944499207394635],  
                          [-87.21136093139648,33.944499207394635],
                          [-87.21136093139648,33.87953701355924]]]  
      });  
MODIS   
| where geo_point_in_polygon(Column1, Column2, curry)  
| summarize avg(Column3) by Column4  
| render timechart
```
The query gives you this chart:

![e75f0bd23b656bad8a575f0b960b9154.png](e75f0bd23b656bad8a575f0b960b9154.png)


There is only 2 dates in my database at the query moment, so it's bit short chart :) So that was not too bad. The ingestion part could be easily automated and the query part looks nice enough :D 

### Problems with the points

The approach to convert something like MODIS data into points was likely a bit naivistic. There seems to be slight difference with reference calculations I did with [Google Earth Engine](https://earthengine.google.com/).

If we look the leftmost points returned by the ADX query overlaid with the area and the NDVI layer, it's clear there is some sort of projection error. Oh well, at least we cannot blame ADX calculating wrong here :) 

![96ac986f0e1b07391f493ba5cc473584.png](96ac986f0e1b07391f493ba5cc473584.png)

## Getting the bottom of the projection error
My assumption was that the error was coming from the NASA cloud optimised geotiff. To test it out, I downloaded one tile of  original MODIS HDF set and converted it directly to csv with it's sinusoidal projection. I'm not describing the process of downloading here as it's not really in scope. Conversion itself from HDF files to csv was easy:

`gdal_translate -of xyz -co ADD_HEADER_LINE=NO -co COLUMN_SEPARATOR=","  MOD13Q1.A2019081.h08v06.006.2019098110145_01.tif 20190322-8-6.csv`

This particular tile is somewhere in Mexico.

Then I transformed the Sinusoidal coordinates to WGS84 with the proj tool's cs2cs command:

`cat 20190322-8-6.csv |sed 's/,/ /' | cs2cs +proj="sinu +lon_0=0 +x_0=0 +y_0=0 +a=6371007.181 +b=6371007.181 +units=m +no_defs" -f '%.6f' | grep -v "\-3000" | sed 's/0.000000//' | sed $'s/\t/,/g' |sed 's/ //' | sed "s/$/,2019-03-22/" > 20190322.csv`

This transformation also adds the date into the csv file and cleans tabs & spaces and some other garbage from the cs2cs run. How do you know the parameters for the projection? I suppose the options are either to study something related that in university or just google around like me :) Tricky part is knowing when it's right. This you have to test by trying out how the points look in map. Visualising points around islands is a good trick.

Here is image where left you see Google Earth Engine visualization of NDVI values over 0.3. Right side is visualization of same points in ADX. Looks like a good match. Also average NDVI calculations now match between ADX and GEE.

![44b99266a43ade3041aa8182b6f4eaf1.png](44b99266a43ade3041aa8182b6f4eaf1.png)


## Performance

Performance of the ADX is linear in all of the dimensions. Add data and the query gets slower. Add CPU and the query gets faster. There is no spatial indexing or any other kind of indexing. It's NoSQL database that works with partitions, row orders and shards.
With V3 engine and cluster of 2 Standard D11 V2 nodes it takes about 10 seconds per day of full global data to query the averages of NDVI per day of the island presented above in the picture. I tried few partitioning options but it did not affect the query performance. Increasing the node count from 2 to 8 however drops the waiting time to about 3 seconds per day.

### Work to do
At the moment I'm looking into inline python which seems like particular handy functionality with remote sensing data. Typically there are multiple data values in the raster image called bands. Some of these bands are data from different sensors like infrared or blue spectrum and others are quality data like percentage of clouds. Typically you would calculate several bands together and ignore cloudy values in averages etc.

## Notes

ADX database seems to grow linearly. Single 1.5G raster ingestion increased size by 4G. This 1.5G was 500x500m resolution landmass image.

There is still no autoshutdown feature in ADX. Remember to turn it off after your experiments specially with larger VM sizes.