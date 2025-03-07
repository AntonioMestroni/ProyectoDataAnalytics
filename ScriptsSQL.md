# 📌 Análisis y limpieza de Datos de Alquileres en SQL

Esta etapa consiste principalmente en la carga y limpieza de datos extraídos anteriormente mediante web scraping, preparandolos para posteriormente crear un informe limpio de errores en Power BI. Adicionalmente se aplicaron cambios también en dicho programa utilizando Power Query y Dax.
### Creación de Base de datos
Primero creé la base de datos.
```sql
CREATE DATABASE Database_Alquileres_Capital;
GO
USE Database_Alquileres_Capital;
```
Antes de cargar los datos creé la tabla con la estructura y columnas adecuadas.

```sql 
CREATE TABLE Departamentos (
	ID INT IDENTITY (1,1) PRIMARY KEY,
	Propiedad Varchar (50),
	Barrio Varchar (100),
	Valor_Alquiler Varchar (50),  #Para mantener los terminos "USD" y "ARS"
	Valor_Expensas Varchar (50),
	Metros_Cuadrados INT,
	Ambientes INT,
	Dormitorios INT,
	Baños INT,
	Cocheras INT
);
```
### Carga de datos
Cargué los datos desde el CSV. Utilicé como delimitador ";" para evitar errores o "," que podían estar en los datos que obtuve en el web scraping.

```sql
BULK INSERT dbo.Departamentos
FROM 'C:\Users\Usuario\Desktop\Alquileres.csv' #Ubicación del archivo
WITH (
		FIELDTERMINATOR = ';',
		ROWTERMINATOR = '0x0A',
		CODEPAGE = '65001',
		FIRSTROW = 2,
		TABLOCK,
		ERRORFILE = 'C:\Users\Usuario\Desktop\Errores.log',
		MAXERRORS = 1000
		)
```
### Limpieza de datos
Inicié limpiando la columna de Valor_Expensas.
```sql
UPDATE dbo.Departamentos
SET Valor_Expensas = TRIM(REPLACE(REPLACE(Valor_Expensas , "Expensas, ""), "$", ""))
WHERE Valor_Expensas IS NOT NULL;
```
Pasé la columna a INT.
```sql
ALTER TABLE dbo.Departamentos
ALTER COLUMN Valor_Expensas INT NULL;
```
Creé nuevas columnas para tener el valor de alquiler en USD y ARS.
```sql
ALTER TABLE dbo.Departamentos
ADD Valor_Alquiler_USD Varchar (50) NULL;
		 Valor_Alquiler_ARS Varchar (50) NULL;
```
Completé las columnas nuevas extrayendo los datos de la columna Valor Alquiler. Luego las converti en INT.
```sql
UPDATE dbo.Departamentos
SET 
    Valor_Alquiler_USD = CASE 
        WHEN LEFT(LTRIM(RTRIM(Valor_Alquiler)), 3) = 'USD' 
        THEN REPLACE(SUBSTRING(LTRIM(RTRIM(Valor_Alquiler)), 5, LEN(Valor_Alquiler)), ' ', '')
        ELSE NULL 
    END,
    Valor_Alquiler_ARS = CASE 
        WHEN LEFT(LTRIM(RTRIM(Valor_Alquiler)), 1) = '$' 
        THEN REPLACE(SUBSTRING(LTRIM(RTRIM(Valor_Alquiler)), 2, LEN(Valor_Alquiler)), ' ', '')
        ELSE NULL 
    END;
```
```sql
ALTER TABLE dbo.Departamentos
ALTER COLUMN Valor_Alquiler_USD INT NULL;

ALTER TABLE dbo.Departamentos
ALTER COLUMN Valor_Alquiler_ARS INT NULL;
```
Creé una nueva columna para saber en que divisa fue tasado inicialmente el departamento.
```sql
ALTER TABLE dbo.Departamentos
ADD Tasación_Inicial Varchar(3) NULL;
```
```sql
UPDATE dbo.Departamentos
SET Tasación_Inicial = CASE
		WHEN Valor_Alquiler_USD IS NOT NULL THEN "USD"
		WHEN Valor_Alquiler_ARS IS NOT NULL THEN "USD"
		ELSE NULL
END;
```
Completé los datos faltantes o nulos de las columnas Valor Alquiler USD y Valor Alquiler ARS usando el tipo de cambio actual.
```sql
UPDATE dbo.Departamentos
SET Valor_Alquiler_ARS = Valor_Alquiler_USD * 1230
WHERE Tasación_Inicial = "USD" AND Valor_Alquiler_ARS IS NULL;

UPDATE dbo.Departamentos
SET Valor_Alquiler_USD = Valor_Alquiler_ARS / 1230
WHERE Tasación_Inicial ="ARS" AND Valor_Alquiler_USD IS NULL;
```
Para finalizar limpié datos incorrectos como "Otro" en la columna de Barrios y valores demasiado elevados que estaban mal cargados en la web  y correspondían a departamentos en venta.
```sql
DELETE FROM dbo.Departamentos 
WHERE Valor_Alquiler_USD > 31000 AND Valor_Alquiler IS NOT NULL;
```
```sql
DELETE FROM dbo.Departamentos 
WHERE Barrio = 'Otro';
```
Posteriormente conecté la base de datos con Power BI y agregué columnas adicionales que eran necesarias utilizando Power Query y Dax.
