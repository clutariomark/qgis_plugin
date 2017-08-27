# Country Administrative Divisions #

This discusses the source of data for `Divisions` and `DivisionTypes` tables. This also discusses the format of the source file, how it was processed and how to query the `Divsions` table.

## Data Source ##
Most of the data that was used to generate the administrative division tables came from Global Administrative Areas (GDAM) website: [http://www.gadm.org](http://www.gadm.org). Data for each country can be downloaded separately and can have different file formats. The coordinate reference system is **longitude/latitude** and the **WGS84 (SRID 4326)** datum.

The data file formats available are:

- **ESRI file geodatabase** - standard format used by *ArcGIS*
- **Shapefile** - consist of at least four actual files (.shp, .shx, .dbf, .prj). This is a commonly used format that can be directly used in Arc-anything, DIVA-GIS, and many other programs. Unfortunately, many of the non standard latin (roman / english) characters are lost in the shapefile, so you should avoid using it. If you insist, you can use the .csv file that comes with the shapefiles, or the attribute data in the geodatabase for the correct attributes (the geodatabase is a MS Access database that (on windows) can be accessed via ODBC). 
- **Geopackage** - a very good general spatial data file format (for vector data). It is based on the SpatiaLite format, and can be read by software using GDAL/OGR, including QGIS and ArcMap. 
- **R (SpatialPolygonsDataFrame)** - can be used in R. To use it, first load the sp package using library(sp) and then use readRDS("filename.rds") (obviously replacing "filename.rds" with the actual filename). See the CRAN spatial task view. Note that this is different R file format than used in previous versions (the RData format that could be read via 'load').
- **Google Earth (.kmz)** - can be opened in Google Earth. 
- **ESRI personal geodatabase** - a MS Access file that can be opened in ArcGIS. One of its advantages, compared to a shapefile, is that it can store non-latin characters (e.g. Cyrillic and Chinese characters). You can also query the (attribute) data in Access or via ODBC. 

## Source File (CSV File) Description ##
**Shapefile format** was downloaded. The country and administrative subdivisions information were extracted from the **csv files** accompanying the shapefile package. The information for each administrative level (i.e. country, state, province) are contained in separate csv files. The naming convention of the csv files is as follows:

`[3 letter Isocode]_adm[administrative level].csv` 

The Administrative level are represented by integers:

- 0 - country level
- 1 - 1st administrative division
- 2 - 2nd adminsitrative division 

The isocode are based on [ISO-3166](https://www.iso.org/new-way-of-using-iso-3166.html) which is a standard devised by the International Organization on Standardization (ISO) for defining codes for countries (ISO-3166-1) and their principal subdivisions (ISO-3166-2).

e.g. Afghanistan (Country Level) - `AFG_adm0.csv`

The columns per csv file vary depending on the admin level. Country level csv files usually have the following columns:

- `OBJECTID` - id (integer) of polygon/feature in shapefile
- `ID_0` - internal id (integer) of the country
- `ISO` - 3 letter isocode of the country
- `NAME_ENGLISH` - country name in english
- `NAME_ISO` - country name based on ISO specification
- `NAME_FAO` - country name based on FAO specification
- `NAME_LOCAL` - local name of country

The country subdivisions csv files (level 1, 2 etc) have the following columns:

- `OBJECTID` - id (integer) of polygon/feature in shapefile
- `ID_0` - internal id (integer) of country
- `ISO` - 3 letter isocode of the country
- `NAME_0` - name of the country
- `ID_[Parent Admin Level]` - internal id (integer) of the parent admin level
- `NAME_[Parent Admin Level]` - name of the parent admin level
- `ID_[Current Admin Level]` - internal id (integer) of the current admin level
- `NAME_[Current Admin Level]` - name of the current admin level
- `HASC_[Current Admin Level]` - Hierarchical administrative subdivision codes (HASC) of the current admin level
- `TYPE_[Current Admin Level]` - division type of the current admin level in local language
- `ENGTYPE_[Current Admin Level]` - division type of the current admin level in English

The [Hierarchical administrative subdivision codes (HASC)](http://www.iscramlive.org/ISCRAM2013/files/198.pdf) were developed by Gwillim Law and Martin Hammitzsch which basically extends the ISO-3166 coding convention to succeeding administrative levels (i.e. level 2, 3 onwards). Example:

    DE - Germany
    DE.NW - North Rhine-Westphalia
    DE.NW.CE - Kreis Coesfeld 


## Data Processing ##
CSV files were processed to create a hierarchical table. The resulting output of the processed files is a table having the following columns:

- `id` - obtained from `HASC_[Current Admin Level]` columns for division level 1+. For country level, obtained from 2 letter isocode from ISO website 
- `name` - obtained from `NAME_ENGLISH` for level 0 and `NAME_[Current Admin Level]` for level 1+ 
- `parentId` - for level 0 this is set to `NULL`, for level 1+ obtained from id of `NAME_[Parent Admin Level]`
- `divLevel` - obtained from csv file name
- `divisionTypeId` - division type names are obtained from `ENGTYPE_[Current Admin Level]` column. Unique division types are stored and assigned a unique id (integer). 

**Sample Divisions Table**

| id        | name         | parentId | divLevel | divisionTypeId |
| --------- | ------------ | -------- | -------- | -------------- |
| BR        | Brazil       | NULL     | 0        | 1              |
| BR.AC     | Acre         | BR       | 1        | 19             |
| BR.AL     | Alagoas      | BR       | 1        | 19             |
| BR.AC.001 | Acrelandia   | BR.AC    | 2        | 6              |
| BR.AC.002 | Assis Brazil | BR.AC    | 2        | 6              |

**Sample DivisionTypes Table**

| id    | name         |
| ----- | ------------ |
| 1     | Country      |
| 19    | State        |
| 6     | Municipality |
| 10000 | Unknown      |

## SQL Queries ##

###From the top level query###
e.g. Get all the places in Libya (all levels)

```sql
WITH RECURSIVE nodes(id, name, parentId, divLevel, typeId) AS (
   SELECT s1.id, s1.name, s1.parentId, s1.divLevel, s1.typeId
   FROM Divisions s1 WHERE parentId = 'LY'
     UNION
   SELECT s2.id, s2.name, s2.parentId, s2.divLevel, s2.typeId
   FROM Divisions s2, nodes s1 WHERE s2.parentId = s1.id
)
SELECT id, name FROM nodes;
```

Output:

| id       | name               |
| -------- | ------------------ |
| LY.BU    | Al Butnan          |
| LY.JA    | Al Jabal al Akhdar |
| LY.JA.01 | Albayda            |
| LY.JA.02 | Shahhat            |
| LY.BU.01 | Bir Alashhab       |
| LY.BU.02 | Emsaed             |


### From the bottom level query ###
e.g. Get all parents of Emsaed (LY.BU.02)

```sql
WITH RECURSIVE nodes(id, name, "parentId", "divLevel", "divisionTypeId") AS (
   SELECT s1.id, s1.name, s1."parentId", s1."divLevel", s1."divisionTypeId"
   FROM "Divisions" s1 WHERE id = 'LY.BU.02'
     UNION
   SELECT s2.id, s2.name, s2."parentId", s2."divLevel", s2."divisionTypeId"
   FROM "Divisions" s2, nodes s1 WHERE s2.id = s1."parentId"
)
SELECT id, name, "divLevel" FROM nodes;
```

Output:

| id       | name      | divLevel |
| -------- | --------- | -------- |
| LY.BU.02 | Emsaed    | 2        |
| LY.BU    | Al Butnan | 1        |
| LY       | Libya     | 0        |

