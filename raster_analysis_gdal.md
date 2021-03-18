# Analizy rastrowe
---
## Konsola OSGeo4W Shell Start-Programy- OSGeo4W 
## Funkcjonalnosc GDAL

sprawdzenie wersji i wyświetlenie pomocy do funkcji gdalwarp
 >gdalinfo --version

 >gdalwarp --help

transformacja SRTM z WGS84 to UTM 34 (EPSG:32634) 
C:\Program Files\QGIS 3.16\bin
>gdalwarp -t_srs EPSG:32634 n50_e021_1arc_v3.tif  srtm.tif

---
## Konsola Windows
## Import SRTM do postgres

z konsoli (cmd) C:\Program Files\QGIS 3.16\bin 
>raster2pgsql.exe -s 32634 -C c:\tmp1\srtm.tif srtm | psql -d egib -U postgres -p 5432

---

## Konsola Postgres 
## Zapytania SQL....

### Obcinanie rastra

>CREATE TABLE obszar (id serial PRIMARY KEY, nazwa varchar (15)); 

>SELECT AddGeometryColumn ('public', 'obszar','wsp',32634,'POLYGON',2);
 
>INSERT INTO obszar (id,nazwa,wsp) VALUES (1,'p1',ST_GeomFromText('POLYGON((562000 5613000, 5700000 5613000, 570000 5605000, 562000 5605000, 562000 5613000))',32634)); 
 
>CREATE TABLE srtm_clip AS SELECT rid,ST_CLIP(rast,wsp) AS rast FROM srtm,obszar WHERE rid=1  AND id=1; 

>ALTER TABLE srtm_clip ADD PRIMARY KEY (rid); 

### Obliczanie nachyleń: ST_SLOPE
>CREATE TABLE srtm_slope AS SELECT rid,ST_Slope(rast) FROM srtm_clip;

>ALTER TABLE srtm_slope ADD PRIMARY KEY (rid); 

### Wyznaczanie terenów płaskich, reklasyfikacja ST_RECLASS

>CREATE TABLE maska_s AS SELECT * FROM srtm_slope;

>UPDATE maska_s SET st_slope = ST_Reclass(st_slope,1,'[0-2]:1, (2-99999]:0','16BSI',0); 

>ALTER TABLE maska_s ADD PRIMARY KEY (rid); 

### Obliczanie ekspozycji ST_Aspect

>CREATE TABLE srtm_aspect AS SELECT rid,ST_Aspect(rast,1,’16BSI’) FROM srtm_clip;

>ALTER TABLE srtm_aspect ADD PRIMARY KEY (rid); 


### Wyznaczanie terenów o espozycji nie zachodnej

>CREATE TABLE maska_a AS SELECT * FROM srtm_aspect;

>UPDATE maska_a SET st_aspect = ST_Reclass(st_aspect,1,’[-1-225):1, (225-315]:0, (315-360]:1’,’16BSI’,0); 

>ALTER TABLE maska_a ADD PRIMARY KEY (rid); 

### Wyznaczenie części wspólnej - przecinanie rastrów

>CREATE TABLE maska (rid serial PRIMARY KEY, rast raster);

>INSERT INTO maska (rid) VALUES (1);

>UPDATE maska SET rast=(SELECT ST_MapAlgebra (maska_s.st_slope,1,maska_a.st_aspect,1,'([rast1]*[rast2.val])') AS rast FROM maska_s,maska_a) WHERE rid=1;


### Zamiana rastra na wektor

warto sprawdzić po każdym zapytaniu wynik w QGIS

>CREATE TABLE maska_w AS SELECT dp.* FROM maska, LATERAL ST_DumpAsPolygons(rast) AS dp;

>ALTER TABLE maska_w ADD COLUMN rid serial;

>ALTER TABLE maska_w ADD PRIMARY KEY (rid);

>ALTER TABLE maska_w ALTER COLUMN geom TYPE geometry(Polygon);

### Nadawanie układu współrzędnych

>SELECT UpdateGeometrySRID('maska_w','geom',32634);
>SELECT FIND_SRID('public','maska_w','geom');

### Zamiana układu współrzędnych tabela1 EPSG 32634

>CREATE TABLE tabelaUtm AS SELECT id, pow_opis, pow2, ST_Transform(geom,32634) AS geom FROM tabela1;

### Przecięcie maska_w i tabelaUtm

>CREATE TABLE wynik1 AS SELECT  id, pow_opis, pow2, (ST_Intersection(tabelaUtm.geom, maska_w.geom)) AS geom FROM tabelaUtm, maska_w;

### Dodanie klucza głównego

>ALTER TABLE wynik1 ADD COLUMN gid serial PRIMARY KEY; 

### Sprawdzenie poprawności i usunięcie błędłów

>SELECT id, St_IsValid(geom) as test from tabelautm where St_IsValid(geom) = 'f'; 

>DELETE FROM tabelautm where St_IsValid(geom) = 'f';


## dodatek

DROP TABLE obszar;

DROP TABLE srtm_clip;

DROP TABLE srtm_slope;

DROP TABLE srtm_aspect;

DROP TABLE maska_s;

DROP TABLE maska_a;

DROP TABLE maska;

DROP TABLE maska_w;

---
---

CREATE TABLE obszar (id serial PRIMARY KEY, nazwa varchar (15)); 

SELECT AddGeometryColumn ('public', 'obszar','wsp',32634,'POLYGON',2);
 
INSERT INTO obszar (id,nazwa,wsp) VALUES (1,'p1',ST_GeomFromText('POLYGON((562000 5613000, 5700000 5613000, 570000 5605000, 562000 5605000, 562000 5613000))',32634)); 

CREATE TABLE srtm_clip AS SELECT rid,ST_CLIP(rast,wsp) AS rast FROM srtm,obszar WHERE rid=1  AND id=1; 

ALTER TABLE srtm_clip ADD PRIMARY KEY (rid); 

CREATE TABLE srtm_slope AS SELECT rid,ST_Slope(rast) FROM srtm_clip;

ALTER TABLE srtm_slope ADD PRIMARY KEY (rid); 

CREATE TABLE maska_s AS SELECT * FROM srtm_slope;

UPDATE maska_s SET st_slope = ST_Reclass(st_slope,1,'[0-2]:1, (2-99999]:0','16BSI',0); 

ALTER TABLE maska_s ADD PRIMARY KEY (rid); 

CREATE TABLE srtm_aspect AS SELECT rid,ST_Aspect(rast,1,’16BSI’) FROM srtm_clip;

ALTER TABLE srtm_aspect ADD PRIMARY KEY (rid); 

CREATE TABLE maska_a AS SELECT * FROM srtm_aspect;

UPDATE maska_a SET st_aspect = ST_Reclass(st_aspect,1,’[-1-225):1, (225-315]:0, (315-360]:1’,’16BSI’,0); 

ALTER TABLE maska_a ADD PRIMARY KEY (rid); 

CREATE TABLE maska (rid serial PRIMARY KEY, rast raster);

INSERT INTO maska (rid) VALUES (1);

UPDATE maska SET rast=(SELECT ST_MapAlgebra (maska_s.st_slope,1,maska_a.st_aspect,1,'([rast1]*[rast2.val])') AS rast FROM maska_s,maska_a) WHERE rid=1;

CREATE TABLE maska_w AS SELECT dp.* FROM maska, LATERAL ST_DumpAsPolygons(rast) AS dp;

ALTER TABLE maska_w ADD COLUMN rid serial;

ALTER TABLE maska_w ADD PRIMARY KEY (rid);

ALTER TABLE maska_w ALTER COLUMN geom TYPE geometry(Polygon);

SELECT UpdateGeometrySRID('maska_w','geom',32634);

SELECT FIND_SRID('public','maska_w','geom');

CREATE TABLE tabelaUtm AS SELECT id, pow_opis, pow2, ST_Transform(geom,32634) AS geom FROM tabela1;

CREATE TABLE wynik1 AS SELECT  id, pow_opis, pow2, (ST_Intersection(tabelaUtm.geom, maska_w.geom)) AS geom FROM tabelaUtm, maska_w;

ALTER TABLE wynik1 ADD COLUMN gid serial PRIMARY KEY; 

SELECT id, St_IsValid(geom) as test from tabelautm where St_IsValid(geom) = 'f'; 

DELETE FROM tabelautm where St_IsValid(geom) = 'f';



------
