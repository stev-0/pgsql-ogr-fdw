# PostgreSQL OGR Foreign Data Wrapper

## Motivation

OGR is the vector half of the [GDAL](http://www.gdal.org/) spatial data access library. It allows access to a [large number of GIS data formats](http://www.gdal.org/ogr_formats.html) using a [simple C API](http://www.gdal.org/ogr__api_8h.html) for data reading and writing. Since OGR exposes a simple table structure and PostgreSQL [foreign data wrappers](https://wiki.postgresql.org/wiki/Foreign_data_wrappers) allow access to table structures, the fit seems pretty perfect. 

## Limitations

This implementation currently has the following limitations:

* **PostgreSQL 9.3+** This wrapper does not support the FDW implementations in older versions of PostgreSQL.
* **Tables are read-only.** Foreign data wrappers support read/write and many OGR drivers support read/write, so this limitation can be removed given some development time.
* **Query restrictions are not pushed down to the OGR driver.** PostgreSQL foreign data wrappers support delegating portions of the SQL query to the underlying data source, in this case OGR. This implementation does not currently do this. Instead it reads the **entire contents** of the table into memory and the database then applies the SQL clauses there. It would be a lot more efficient to push as many restriction clauses as possible down to the OGR layer. This limitation can also be removed given some development time.
* **All columns are retrieved every time.** PostgreSQL foreign data wrappers don't require all columns all the time, and some efficiencies can be gained by only requesting the columns needed to fulfill a query. This would be a minimal efficiency improvement, but can be removed given some development time, since the OGR API supports returning a subset of columns.

## Basic Operation

In order to access geometry data from OGR, the PostGIS extension has to be installed: if it is not installed, geometry will be represented as bytea columns, with well-known binary (WKB) values.

To build the wrapper, make sure you have the GDAL library and development packages (is `gdal-config` on your path?) installed, as well as the PostgreSQL development packages (is `pg_config` on your path?)

Build the wrapper with `make` and `make install`. Now you are ready to create a foreign table.

First install the `postgis` and `ogr_fdw` extensions in your database.

    -- Install the required extensions
    CREATE EXTENSION postgis;
    CREATE EXTENSION ogr_fdw;

For a test data set, copy the `pt_two` example shape file from the `data` directory to a location where the PostgreSQL server can read it (like `/tmp/test/` for example). 

Use the `ogr_fdw_info` tool to read an OGR data source and output a server and table definition for a particular layer. (You can write these manually, but the utility makes it a little more foolproof.)

    > ogr_fdw_info -s /tmp/test
    
    Layers:
      pt_two
    
    > ogr_fdw_info -s /tmp/test -l pt_two
    
    CREATE SERVER myserver
      FOREIGN DATA WRAPPER ogr_fdw
      OPTIONS (
        datasource '/tmp/test',
        format 'ESRI Shapefile' );

    CREATE FOREIGN TABLE pt_two (
      fid integer,
      geom geometry,
      name varchar,
      age integer,
      height real,
      birthdate date )
      SERVER myserver
      OPTIONS ( layer 'pt_two' );
    
Copy the `CREATE SERVER` and `CREATE FOREIGN SERVER` SQL commands into the database and you'll have your foreign table definition.

                 Foreign table "public.pt_two"
      Column  |       Type        | Modifiers | FDW Options 
    ----------+-------------------+-----------+-------------
     fid      | integer           |           | 
     geom     | geometry          |           | 
     name     | character varying |           | 
     age      | integer           |           | 
     height   | real              |           | 
     birthday | date              |           | 
    Server: tmp_shape
    FDW Options: (layer 'pt_two')

And you can query the table directly, even though it's really just a shape file.

    > SELECT * FROM pt_two;

     fid |                    geom                    | name  | age | height |  birthday  
    -----+--------------------------------------------+-------+-----+--------+------------
       0 | 0101000000C00497D1162CB93F8CBAEF08A080E63F | Peter |  45 |    5.6 | 1965-04-12
       1 | 010100000054E943ACD697E2BFC0895EE54A46CF3F | Paul  |  33 |   5.84 | 1971-03-25

## Examples

### WFS FDW

Since we can access any OGR data source as a table, how about a public WFS server?

    CREATE EXTENSION postgis;
    CREATE EXTENSION ogr_fdw;

    CREATE SERVER opengeo
      FOREIGN DATA WRAPPER ogr_fdw
      OPTIONS (
        datasource 'WFS:http://demo.opengeo.org/geoserver/wfs',
        format 'WFS' );

    CREATE FOREIGN TABLE topp_states (
      fid integer,
      geom geometry,
      gml_id varchar,
      state_name varchar,
      state_fips varchar,
      sub_region varchar,
      state_abbr varchar,
      land_km real,
      water_km real,
      persons real,
      families real,
      houshold real,
      male real,
      female real,
      workers real,
      drvalone real,
      carpool real,
      pubtrans real,
      employed real,
      unemploy real,
      service real,
      manual real,
      p_male real,
      p_female real,
      samp_pop real )
      SERVER opengeo
      OPTIONS ( layer 'topp:states' );

### FGDB FDW

Unzip the `Querying.zip` file from the `data` directory to get a `Querying.gdb` file, and put it somewhere public (like `/tmp`). Now run the `ogr_fdw_info` tool on it to get a table definition.

    CREATE SERVER fgdbtest
      FOREIGN DATA WRAPPER ogr_fdw
      OPTIONS (
        datasource '/tmp/Querying.gdb',
        format 'OpenFileGDB' );

    CREATE FOREIGN TABLE cities (
      fid integer,
      geom geometry,
      city_fips varchar,
      city_name varchar,
      state_fips varchar,
      state_name varchar,
      state_city varchar,
      type varchar,
      capital varchar,
      elevation integer,
      pop1990 integer,
      popcat integer )
      SERVER fgdbtest
      OPTIONS ( layer 'Cities' );

Query away!

