[![Build Status](https://travis-ci.org/mappl/scidb4gdal.svg?branch=master)](https://travis-ci.org/mappl/scidb4gdal)
# scidb4gdal
A GDAL driver for SciDB arrays

## Description
This is a preliminary version of a [GDAL](http://www.gdal.org) driver for SciDB. Spatial reference of arrays is maintained if the SciDB database uses the [scidb4geo plugin](https://github.com/mappl/scidb4geo).
Otherwise, the GDAL driver might be still useful e.g. for converting two-dimensional arrays to a variety of image formats supported by GDAL. 

The driver offers support for reading and writing SciDB arrays. Update access to existing arrays is currently not implemented but planned for future releases.

## News
- (2015-09-01)
    - Driver now compiles under Windows
- (2015-08-27)
	- Support for empty SciDB cells (will be filled with NoData value) added
	- Fixed dimension mismatch between GDAL and SciDB
	- General GDAL metadata (e.g. color interpretation, no data, offset, and scaling values for bands) can be stored in SciDB's system catalog
- (2015-06-23)
	- Automated builds added, build/prepare_platform.sh might also help you to automatically build GDAL from source including this scidb driver
- (2015-04-02)
    - Support for HTTPS connections to Shim
    - Improved performance for both read and write access

## Getting Started
Similar to other database drivers for GDAL, we need to access a database in order to perform queries. Therefore two strategies can be utilized to do so. The first strategy was introduced by the early database drivers. They used a connection string to access the database. Typically the connection string was passed as the file name.
The second strategy was introduced as opening options parameter for the GDAL functions with GDAL version 2.0.

Examples on connection approaches:
- file name based: `"SCIDB:array=<arrayname> [host=<host> port=<port> user=<user> password=<password>]"`

- opening option based: `-oo "host=<host>" -oo "port=<port>" -oo "user=<user>" -oo "password=<password> -oo "ssl=true"`

Notice that the file name for SciDB must start with `SCIDB:` in order to let GDAL identify the dataset as a SciDB array. Default values for parameters, if additional information is provided, are the following:

    <host>     = https://localhost
    <port>     = 8083
    <user>     = scidb
    <password> = scidb

One of the benefits of SciDB is that it allows to store not only two dimensional data, but it supports multi-dimensional storage. The scidb4geo plugin, developed also in this context, extends SciDB to store images with a spatial and optional a temporal reference that is annotated to the SciDB arrays as metadata. With this annotations image coordinates (or dimension indices of arrays) can be transformed into real world coordinates and dates. GDAL can already deal with spatial operations like subsetting and image reprojections and therefore the standard parameter for those operations can be also used with SciDBArrays. For example the gdal_translate parameter like `-projwin` or `-srcwin` will work for spatial subsetting and querying an array in SciDB.

Since GDAL was developed primarily in a spatial context, the temporal query proved difficult to realize in the current GDAL version. For this purpose we introduced several ways to state the temporal context, when querying a temporally referenced array.

There are three major approaches to do this:

1. suffix of the array name: `"SCIDB:array=array_name[i,1]"` or `"SCIDB:array=array_name[t,2015-03-03T10:00:00]"`
2. as part of a property string within the connection string: `"SCIDB:array=<arrayname> [host=<host> port=<port> user=<user> password=<password>] [properties=i=<temporal_index>]"`
3. as part of the opening options: `-oo "i=<temporal_index>"`

To address the temporal component the identifiers "t" and "i" are used. "t" refers to a string representation of a data or data/time that is a valid string representation according to ISO 8601. With the identifier "i" the temporal index of an image is addressed. The temporal index refers to the discrete dimension value of the assigned temporal dimension. This means i=0 would query for the image that was inserted at the starting date of the time series. A value higher then zero would mean that SciDB would try to access the i*dt image from the start (dt being the time resolution stated with the temporal reference).

When using "gdal_translate" to access spatio-temporally references imagery in SciDB note that exactly one image will be returned. With this in mind the temporal request is limited to search with one time component, meaning that no interval query is currently supported. The temporal query will return the temporally nearest image that is found in the data base.

## Image injection into SciDB
If you want to load an image into SciDB, we consider three different array representations. Those array representation are important to create the correct array structure in SciDB, meaning that if the image shall have a temporal component that an additional dimension needs to be assigned.
The preferred mechanism to pass user defined settings is to use GDALs Create Options (-co flag). Each setting of the create options has to be a key-value pair that is separated by "=". 

1. Spatial Array
The spatial array is the default case of spatial image representation. This representation just assigns two dimensions for the spatial components and it attaches the spatial reference system that is stated in the metadata of the source file to the SciDB array. When creating this representation you should use the key-value pair `"type=S"`.

2. Spatio-temporal Array
The spatio temporal array is a representation where the spatial image has also a time stamp. For this purpose the driver will assign one additional temporal dimension to the array and it assigns a user defined temporal reference system (TRS). The temporal reference system consists of the starting date and the temporal resolution. The starting date needs to be written as a date or date/time string according to ISO 8601 and the temporal resolution is a temporal period string. For example one possible valid TRS statement would look like `"t=2014-08-10T10:00"` and `"dt=P1D"` meaning that this reference starts at 2014-08-10 and has a temporal resolution of one day. To create this array type use `"type=ST"`. Please be advised that this type refers simply to exactly one point in time. There is no way to add later images into this array. The minimum and maximum of the temporal dimension for this array will be set to zero.

3. Spatio-temporal Series
This array type is very similar to the Spatio-temporal Array, but it removes the restriction of the temporal dimension on carrying only one image. To be more concrete: This type only starts a time series. Additional images can then be inserted into this array by using the same array name and by using a Spatio-temporal Array. In order to start the spatio-temporal series, please use `"type=STS"`.

Now that we have covered the main types, there is another addition in creating SciDB arrays. The before mentioned types will restrict the spatial dimension to the images boundary. This means that if it not explicitly stated otherwise the spatial boundaries will be fixed to that extent. But to allow later insertion of images into in an existing array, a bounding box and its coordinates reference system need to be stated, when creating the image (the first upload). We use the parameter keys "bbox" and "srs" for this purpose. By setting an alternate bounding box we will refer to the data stored as a coverage, whereas before the data represented the original image. Here is also an example on setting a valid coverage statement: `"bbox=443000 4650000 455000 4629000"` and `"srs=EPSG:26716"`. Note that the spatial reference system is addressed to by stating the authority name and the systems id.

In the following we will give some explicit gdal_translate statements on how to create arrays:
Create a spatial image from a file:
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "type=S" -of SciDB input_image.tif "SCIDB:array=test_spatial"`

Create a spatial coverage from multiple files:
1. Start a coverage by stating a bounding box with SRS
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "bbox=443000 4650000 455000 4629000" -co "srs=EPSG:26716" -co "type=S" -of SciDB part1.tif "SCIDB:array=test_spatial_coverage"`

2. Insert an image into the coverage
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "type=S" -of SciDB part2.tif "SCIDB:array=test_spatial_coverage"`

Create a Spatio-Temporal Array:
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-15" -co "type=ST" -of SciDB input_image.tif "SCIDB:array=test_spatio_temporal"`

Create a Spatio-Temporal Series:
1. Start series
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-15" -co "type=STS" -of SciDB input_image_15.tif "SCIDB:array=test_spatio_temporal_series"`

2. Insert an image to another time index after the starting date
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-16" -co "type=ST" -of SciDB input_image_16.tif "SCIDB:array=test_spatio_temporal_series"`

Create a Spatio-Temporal Series with larger boundaries:
1. Start series and stating an additional extent
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-15" -co "bbox=443000 4650000 455000 4629000" -co "srs=EPSG:26716" -co "type=STS" -of SciDB input_image_15.tif "SCIDB:array=test_spatio_temporal_series_coverage"`
2. Add image into the image of the 15th october
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-15" -co "type=ST" -of SciDB input_image_15_2.tif "SCIDB:array=test_spatio_temporal_series_coverage"`
3. Add image at another date
`gdal_translate -co "host=https://your.host.de" -co "port=31000" -co "user=user" -co "password=passwd" -co "dt=P1D" -co "t=2015-10-17" -co "type=ST" -of SciDB input_image_17.tif "SCIDB:array=test_spatio_temporal_series_coverage"`

If a coverage is used please make sure that the coordinates of the images refer to the same spatial reference system. It is also important that all images that are inserted into a coverage are within the initially stated boundary!

## Deleting arrays
In order to allow GDAL to delete arrays, we enabled this particular feature via gdalmanage. Since gdalmanage does not support opening options, the connection string approach must be used, e.g. `gdalmanage delete "SCIDB:array=test_spatial host=https://your.host.de port=31000 user=user password=passwd confirmDelete=Y"`. This command completely removes the array from the database. Please be sure that the array is gone once this command is executed. An additional parameter "confirmDelete" was introduced in order to prevent accidental deletion of an array. This is due to GDALs QuietDelete function that is called on each gdal_translate call. As values the following strings are allowed (case-insensitive): YES, Y, TRUE, T or 1.

## Dependencies
- At the moment the driver requires [Shim](https://github.com/Paradigm4/shim) to run on SciDB databases you want to connect to. In the future, this may or may not be changed to connecting directly to SciDB sockets using Google's protocol buffers
- [ST plugin] (https://github.com/mappl/scidb4geo) for SciDB
- We use [cURL](http://curl.haxx.se/) to interface with SciDB's web service shim
- Some [Boost](http://www.boost.org) header-only libraries (no external libraries required for linking) for string functions


## Build Instructions

The following instructions show you how to compile GDAL with added SciDB driver on Unix environments.

1. Download GDAL source
2. Clone this repository `git clone https://github.com/mappl/scidb4gdal` 
3. Copy the source to `GDAL_SRC_DIR/frmts/scidb` by `cp scidb4gdal/src GDAL_SRC_DIR/frmts/scidb`
4. Add driver to GDAL source tree (see http://www.gdal.org/gdal_drivertut.html#gdal_drivertut_addingdriver):
    1. Add `GDALRegister_SciDB()`to `GDAL_SRC_DIR/gcore/gdal_frmts.h`
    2. Add call to `GDALRegister_SciDB()` in `GDAL_SRC_DIR/frmts/gdalallregister.cpp` within `#ifdef FRMT_scidb`
    3. Add "scidb" to `GDAL_FORMATS` in `GDAL_SRC_DIR/GDALmake.opt.in`
5. Build GDAL `./configure && make && sudo make install`. 
6. Eventually, you might need to run `sudo ldconfig` to make GDAL's shared library available.

If you get some missing include file errors, you need to install Boost manually. Either use your distribution's package manager e.g. `sudo apt-get install libboost-dev` or simply copy Boost header files to a standard include directory like `/usr/include`.


### Build on Windows

The following instructions demonstrate how to compile GDAL with added SciDB driver on Windows using Visual Studio 2013.  
Detailed information for tweaking windows builds can be found at http://trac.osgeo.org/gdal/wiki/BuildingOnWindows.
We recommend the [OSGeo4W](http://trac.osgeo.org/osgeo4w/) network installer for managing and installing external GIS libraries on Windows. 
In particular this allows you to easily install curl and boost development libraries that are needed to compile this driver.
 
1. Download GDAL source
2. Clone this repository
3. Copy the `src` directory of your clone to `GDAL_SRC_DIR/frmts` and rename it `scidb`
4. Add driver to GDAL source tree (see http://www.gdal.org/gdal_drivertut.html#gdal_drivertut_addingdriver):
    1. Add `GDALRegister_SciDB()`to `GDAL_SRC_DIR/gcore/gdal_frmts.h`
    2. Add call to `GDALRegister_SciDB()` in `GDAL_SRC_DIR/frmts/gdalallregister.cpp` within `#ifdef FRMT_scidb`
    3. Open `GDAL_SRC_DIR/frmts/makefile.vc` and add `-DFRMT_scidb`  within the `!IFDEF CURL_LIB` block
5. Setup external include and library paths
    1. Uncomment lines to set `CURL_INC` and `CURL_LIB` in `GDAL_SRC_DIR/nmake.opt`
    2. Edit link to Boost header directory in `GDAL_SRC_DIR/frmts/scidb/makefile.vc`
6. Start a command line 
    1. Change directory to the GDAL source `cd GDAL_SRC_DIR`
	2. Load Visual studio command line tools e.g. by running `"C:\Program Files (x86)\Microsoft Visual Studio 12.0\VC\bin\x86_amd64\vcvarsx86_amd64.bat"
` for x64 builds using Visual Studio 2013
    3. Run nmake e.g. `nmake /f makefile.vc MSVC_VER=1800 WIN64=YES`
 
