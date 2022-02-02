# Unique Building Identifier (UBID)

Website: https://buildingid.pnnl.gov/

## Documentation

### Prerequisites

* PostgreSQL - https://www.postgresql.org/
* PostGIS (optional) - https://postgis.net/

### Install

1. Execute https://github.com/google/open-location-code/blob/main/plpgsql/pluscode_functions.sql
2. Execute [buildingid_functions.sql](https://github.com/pnnl/buildingid-plpgsql/blob/main/buildingid_functions.sql)
3. Execute [buildingid_postgis_functions.sql](https://github.com/pnnl/buildingid-plpgsql/blob/main/buildingid_postgis_functions.sql) (optional)

## Usage

### API

[buildingid_functions.sql](https://github.com/pnnl/buildingid-plpgsql/blob/main/buildingid_functions.sql) defines the following PL/pgSQL functions:
* `UBID_Encode(numeric, numeric, numeric, numeric, numeric, numeric[, integer])` &rarr; `text`
* `UBID_EncodeCentroid(numeric, numeric[, integer])` &rarr; `text`
* `UBID_Decode(text)` &rarr; `record`
* `UBID_IsValid(text)` &rarr; `boolean`
* `UBID_CodeArea(numeric, numeric, numeric, numeric, numeric, numeric, numeric, numeric, integer)` &rarr; `record`
* `UBID_CodeArea_Jaccard(record, record)` &rarr; `numeric`

[buildingid_postgis_functions.sql](https://github.com/pnnl/buildingid-plpgsql/blob/main/buildingid_postgis_functions.sql), which requires PostGIS, defines the following PL/pgSQL functions:
* `UBID_EncodeGeom(geometry[, integer])` &rarr; `text`
* `UBID_DecodeGeom(text)` &rarr; `record`

See source files for documentation.

### Examples

#### Encode UBID

Suppose that there exists a database table of buildings ("building"), with columns for the building's ID ("id") and the building's latitude and longitude coordinates ("lat" and "lng").

```sql
CREATE TABLE building (
  id serial PRIMARY KEY, -- building ID
  lat real NOT NULL,     -- latitude coordinate (WGS-84)
  lng real NOT NULL      -- longitude coordinate (WGS-84)
);
```

A new column is added for the building's assigned UBID ("ubid") and the UBID with Open Location Code length `11` is generated using the `UBID_EncodeCentroid(numeric, numeric[, integer])` function.

```sql
ALTER TABLE building
  ADD COLUMN ubid text NOT NULL;

UPDATE building
  SET ubid = UBID_EncodeCentroid(building.lat, building.lng, 11);
```

#### Encode UBID using PostGIS

Suppose that there exists a database table of buildings ("building"), with columns for the building's ID ("id") and the building's footprint geometry ("the_geom").

```sql
CREATE TABLE building (
  id serial PRIMARY KEY,      -- building ID
  footprint geometry NOT NULL -- building footprint geometry (SRID=4326)
);
```

A new column is added for the building's assigned UBID ("ubid") and the UBID with Open Location Code length `11` is generated using the `UBID_EncodeGeom(geometry[, integer])` function.

```sql
ALTER TABLE building
  ADD COLUMN ubid text NOT NULL;

UPDATE building
  SET ubid = UBID_EncodeGeom(building.footprint, 11);
```

#### Decode UBID

Suppose that there exists a database table of buildings ("building"), with columns for the building's ID ("id") and the building's assigned UBID ("ubid").

```sql
CREATE TABLE building (
  id serial PRIMARY KEY, -- building ID
  ubid text NOT NULL     -- assigned UBID
);
```

The `UBID_Decode(text)` function returns a table ("t") that contains:
* The latitude and longitude coordinates for the axis-aligned minimum bounding box for the decoded UBID;
* The latitude and longitude coordinates for the centroid of the decoded UBID; and
* The Open Location Code length for the decoded UBID.

```sql
SELECT
  t.lat_lo,              -- numeric
  t.lng_lo,              -- numeric
  t.lat_hi,              -- numeric
  t.lng_hi,              -- numeric
  t.centroid_lat_lo,     -- numeric
  t.centroid_lng_lo,     -- numeric
  t.centroid_lat_hi,     -- numeric
  t.centroid_lng_hi,     -- numeric
  t.centroid_code_length -- integer
FROM
  building,
  UBID_Decode(building.ubid) AS t
WHERE
  UBID_IsValid(building.ubid)
```

#### Decode UBID using PostGIS

Suppose that there exists a database table of buildings ("building"), with columns for the building's ID ("id") and the building's assigned UBID ("ubid").

```sql
CREATE TABLE building (
  id serial PRIMARY KEY, -- building ID
  ubid text NOT NULL     -- assigned UBID
);
```

The `UBID_DecodeGeom(text)` function returns a table ("t") that contains:
* The envelope for the axis-aligned minimum bounding box ("bbox") for the decoded UBID;
* The envelope for the centroid of the decoded UBID ("centroid"); and
* The Open Location Code length for the decoded UBID.

```sql
SELECT
  t.bbox,                -- geometry (SRID=4326)
  t.centroid,            -- geometry (SRID=4326)
  t.centroid_code_length -- integer
FROM
  building,
  UBID_DecodeGeom(building.ubid) AS t
WHERE
  UBID_IsValid(building.ubid)
```

#### Cross-reference UBIDs

Suppose that there exists a database table of buildings ("building"), with columns for the building's ID ("id") and the building's assigned UBID ("ubid").

```sql
CREATE TABLE building (
  id serial PRIMARY KEY, -- building ID
  ubid text NOT NULL     -- assigned UBID
);
```

Similarly, suppose that there exists a database table of land parcels ("land_parcel"), with columns for the land parcel's ID ("id") and the land parcel's assigned UBID ("ubid").

```sql
CREATE TABLE land_parcel (
  id serial PRIMARY KEY, -- land parcel ID
  ubid text NOT NULL     -- assigned UBID
);
```

The UBID cross-reference quality score ("ubid_score") for each building-land-parcel pair is calculated using the `UBID_CodeArea_Jaccard(record, record)` function.
A building-land-parcel pair is considered a match if the UBID cross-reference quality score is greater than zero.

Buildings and land parcels with valid UBIDs are selected using the `UBID_IsValid(text)` function.

If a building is matched to more than one land parcel, then the land parcels are ranked by their UBID cross-reference quality score.

```sql
SELECT
  building.id AS building_id,
  building.ubid AS building_ubid,
  land_parcel.id AS land_parcel_id,
  land_parcel.ubid AS land_parcel_ubid,
  UBID_CodeArea_Jaccard(UBID_Decode(building.ubid), UBID_Decode(land_parcel.ubid)) AS ubid_score
FROM
  building INNER JOIN land_parcel
    ON UBID_CodeArea_Jaccard(UBID_Decode(building.ubid), UBID_Decode(land_parcel.ubid)) > 0
WHERE
  UBID_IsValid(building.ubid) AND UBID_IsValid(land_parcel.ubid)
ORDER BY
  building_id ASC, ubid_score DESC, land_parcel_id ASC
```

#### Cross-reference UBIDs using PostGIS

The `SELECT` statement in the previous example has the following performance issues:
* For each building-land-parcel pair, the building's UBID and the land parcel's UBID are both validated once and then decoded twice (first as part of the `INNER JOIN` and then in the expression for the "ubid_score" column).
* For each building-land-parcel pair, the UBID cross-reference quality score is either not calculated or calculated twice (first as part of the `INNER JOIN` and then  in the expression for the "ubid_score" column).

The performance is improved by precomputing the bounding box for the building's UBID ("ubid_bbox"), if it is valid, and then by indexing the bounding box's geometry using a quadtree data structure.

```sql
ALTER TABLE building
  ADD COLUMN ubid_bbox geometry;

CREATE INDEX building_ubid_bbox_idx
  ON building USING gist (ubid_bbox);

UPDATE building
  SET
    ubid_bbox = t.bbox
  FROM
    UBID_DecodeGeom(building.ubid) AS t
  WHERE
    UBID_IsValid(building.ubid);
```

And similarly for the land parcels.

```sql
ALTER TABLE land_parcel
  ADD COLUMN ubid_bbox geometry;

CREATE INDEX land_parcel_ubid_bbox_idx
  ON land_parcel USING gist (ubid_bbox);

UPDATE land_parcel
  SET
    ubid_bbox = t.bbox
  FROM
    UBID_DecodeGeom(land_parcel.ubid) AS t
  WHERE
    UBID_IsValid(land_parcel.ubid);
```

The `UBID_CodeArea_Jaccard(record, record)` function is replaced with the quotient of the `ST_Intersection(geometry, geometry)` and `ST_Union(geometry, geometry)` functions.

The database indexes significantly improve the performance of the `ST_Intersects(geometry, geometry)` function in the `INNER JOIN`.

Only building-land-parcel pairs with non-`NULL` bounding boxes (i.e., with valid UBIDs) are tested.

```sql
SELECT
  building.id AS building_id,
  building.ubid AS building_ubid,
  land_parcel.id AS land_parcel_id,
  land_parcel.ubid AS land_parcel_ubid,
  (
    ST_Intersection(building.ubid_bbox, land_parcel.ubid_bbox)
      /
    ST_Union(building.ubid_bbox, land_parcel.ubid_bbox)
  ) AS ubid_score
FROM
  building INNER JOIN land_parcel
    ON ST_Intersects(building.ubid_bbox, land_parcel.ubid_bbox)
WHERE
  building.ubid_bbox IS NOT NULL AND land_parcel.ubid_bbox IS NOT NULL
ORDER BY
  building_id ASC, ubid_score DESC, land_parcel_id ASC
```

## License

The source files are available as open source under the terms of [The 2-Clause BSD License](https://opensource.org/licenses/BSD-2-Clause).

## Contributions

Contributions are accepted on [GitHub](https://github.com/) via the fork and pull request workflow. See [here](https://help.github.com/articles/using-pull-requests/) for more information.
