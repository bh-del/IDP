#Analizy wektorowe

##Konsola Windows

import shp, należy mieć urochomieone extension PostGIS w swojej bazie danych

>shp2pgsql.exe -I -s 3120 -W "latin1" C:\tmp1\egib_g\dzialki.shp public.dzialki | psql -d egib -U postgres -p 5432

##Konsola Postgres

### Utworzenie tabeli
 
>CREATE TABLE dze_opis (Dzialka_ewidencyjna varchar (50), Nazwa varchar (50), AM varchar (50), Obreb_ew varchar (50),  Nazwa_obreb varchar (50),Jednostka_rej varchar (50),Data_wladania varchar (50), Rejon_stat varchar (50), Wartosc varchar (50), Data_wyceny varchar (50), Dokladnosc varchar (50), Nr_rejestru_zab varchar (50), Odchylka varchar (50), Pow_ew varchar (50), Pow_geod varchar (50), SWDETag varchar (50), Data_werfikacji varchar (50), Data_utworzenia varchar (50), Archiwalny varchar (50));

### Skopiowanie danych z pliku csv do tabeli

>COPY dze_opis FROM 'C:\tmp1\egib_o.csv'  DELIMITER ',' CSV HEADER;

### Dodanie do tabeli nowej kolumny

>ALTER TABLE dzialki ADD COLUMN pow2 float;

### Obliczenie powiedzchni działek

>UPDATE dzialki SET pow2=St_Area(geom);

### Połaczenie tabel EGIB, cześci graficznej i opisowej

*uwaga poniżej jako atrybuty nowej tabeli "tabela" podane sa atrubuty z obu tabel*

>SELECT dzialki.id, pow_ew, pow2, geom INTO tabela  FROM dzialki LEFT OUTER JOIN dze_opis ON (dzialki.id=dze_opis.dzialka_ewidencyjna);

### Dodanie nowej kolumny

>ALTER TABLE tabela ADD COLUMN roznica float;

### Zamiana typu atrybutu pow_ew na liczbowy

*w przeciwnym razie nie da sie policzyć różnicy*

>ALTER TABLE tabela ALTER COLUMN pow_ew TYPE numeric(10,0) USING pow_ew::numeric;


### Obliczenie różnic powierzchni

>UPDATE tabela SET roznica=pow2-pow_ew;

### Obliczenie dopuszczalnej różnicy powierzchni

>ALTER TABLE tabela ADD COLUMN deltap float;
>UPDATE tabela SET deltap=0.001*pow2 +0.2*sqrt(pow2); 

### Wybranie działek o przekroczonej dopuszczalnej odchyłce powierzchni i zapisanie do nowej tabeli

>SELECT id, pow_ew, pow2, geom INTO tabela1 FROM tabela WHERE abs(roznica) > deltap;

>SELECT pow_ew,pow2,roznica,deltap FROM tabela LIMIT 10;



## Dodatek

*polecenia do skopiowania i uruchomienia w konsoli*

shp2pgsql.exe -I -s 3120 -W "latin1" C:\tmp1\egib_g\dzialki.shp public.dzialki | psql -d egib -U postgres -p 5432
CREATE TABLE dze_opis (Dzialka_ewidencyjna varchar (50), Nazwa varchar (50), AM varchar (50), Obreb_ew varchar (50),  Nazwa_obreb varchar (50),Jednostka_rej varchar (50),Data_wladania varchar (50), Rejon_stat varchar (50), Wartosc varchar (50), Data_wyceny varchar (50), Dokladnosc varchar (50), Nr_rejestru_zab varchar (50), Odchylka varchar (50), Pow_ew varchar (50), Pow_geod varchar (50), SWDETag varchar (50), Data_werfikacji varchar (50), Data_utworzenia varchar (50), Archiwalny varchar (50));
COPY dze_opis FROM 'C:\tmp1\egib_o.csv'  DELIMITER ',' CSV HEADER;
ALTER TABLE dzialki ADD COLUMN pow2 float;
UPDATE dzialki SET pow2=St_Area(geom);
SELECT dzialki.id, pow_ew, pow2, geom INTO tabela  FROM dzialki LEFT OUTER JOIN dze_opis ON (dzialki.id=dze_opis.dzialka_ewidencyjna);
ALTER TABLE tabela ADD COLUMN roznica float;
ALTER TABLE tabela ALTER COLUMN pow_ew TYPE numeric(10,0) USING pow_ew::numeric;
UPDATE tabela SET roznica=pow2-pow_ew;
ALTER TABLE tabela ADD COLUMN deltap float;
UPDATE tabela SET deltap=0.001*pow2 +0.2*sqrt(pow2); 
SELECT id, pow_ew, pow2, geom INTO tabela1 FROM tabela WHERE abs(roznica) > deltap;
SELECT pow_ew,pow2,roznica,deltap FROM tabela LIMIT 10;

DROP TABLE dze_opis;
DROP TABLE dzialki;
DROP TABLE dzialki1;
DROP TABLE tabela;
DROP TABLE tabela1;

