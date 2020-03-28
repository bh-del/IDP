# hfiahfoifh

fjajajf
---

# Funkcjonalnosc GDAL

## konsola OSGeo4W Shell Start-Programy- OSGeo4W

gdalinfo --version
gdalwarp --help

transformacja SRTM z WGS84 to UTM 34
gdalwarp -t_srs EPSG:32634 n50_e021_1arc_v3.tif  srtm.tif

---

## import SRTM do postgres

z konsoli (cmd) C:\Program Files\PostgreSQL\9.6\bin
raster2pgsql.exe -s 32634 -C c:\tmp1\srtm.tif srtm | psql -d egib -U postgres -p 5432

---

## Konsola postgres zapytania SQL....

obcinanie
CREATE TABLE obszar (id serial PRIMARY KEY, nazwa varchar (15)); 
SELECT AddGeometryColumn ('public', 'obszar','wsp',32634,'POLYGON',2); 
INSERT INTO obszar (id,nazwa,wsp) VALUES (1,'p1',ST_GeomFromText('POLYGON((562000 5613000, 5700000 5613000, 570000 5605000, 562000 5605000, 562000 5613000))',32634));  
CREATE TABLE srtm_clip AS SELECT rid,ST_CLIP(rast,wsp) AS rast FROM srtm,obszar WHERE rid=1  AND id=1; 
ALTER TABLE srtm_clip ADD PRIMARY KEY (rid); 
CREATE TABLE srtm_slope AS SELECT rid,ST_Slope(rast) FROM srtm_clip;
ALTER TABLE srtm_slope ADD PRIMARY KEY (rid); 
CREATE TABLE maska_s AS SELECT * FROM srtm_slope;
UPDATE maska_s SET st_slope = ST_Reclass(st_slope,1,'[0-2]:1, (2-99999]:0','16BSI',0); 

CREATE TABLE srtm_aspect AS SELECT rid,ST_Aspect(rast,1,’16BSI’) FROM srtm_clip;
ALTER TABLE srtm_aspect ADD PRIMARY KEY (rid); 
CREATE TABLE maska_a AS SELECT * FROM srtm_aspect;
UPDATE maska_a SET st_aspect = ST_Reclass(st_aspect,1,’[-1-225):1, (225-315]:0, (315-360]:1’,’16BSI’,0); 

CREATE  TABLE  sentinel_NDVI  AS  SELECT  ST_MapAlgebraExpr(sentinel_k8_clip.rast, 
1,sentinel_k4_clip.rast,  1,'(([rast1]  -  [rast2.val]  )/([rast1]+  [rast2.val]))*100’)  AS  rast 
FROM sentinel_k8_clip,sentinel_k4_clip; 

## dodatek

ST_ASPECT
ST_Reclass
przeciecia, algebra map 

DROP TABLE srtm_clip;
DROP TABLE srtm_slope;
DROP TABLE srtm_aspect;
DROP TABLE maska_s;
DROP TABLE srtm_clip;
DROP TABLE obszar;
DROP TABLE srtm_slope;
DROP TABLE maska_a;
