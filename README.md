
# Ingesta de Datos de SAP a SQL Server para Ventas y Productos

Este documento detalla el proceso y estructura para la ingesta de datos desde SAP a SQL Server, enfocándose en productos, ventas, vendedores, y ciudades.

## Flujo de datos sobre la ingesta
![Diagrama DFLOW](https://github.com/jmorejonm/ETLSAP/blob/main/diagDataFlow.png)

## Estructuras de Tabla
![Diagrama ER](https://github.com/jmorejonm/ETLSAP/blob/main/diagERSAP.png)
### 1. Tabla de Productos
```sql
CREATE TABLE Productos (
    ProductoID INT PRIMARY KEY,
    Nombre NVARCHAR(100),
    Descripcion NVARCHAR(255),
    Precio DECIMAL(10, 2),
    CantidadStock INT,
    Categoria NVARCHAR(50)
);
```

### 2. Tabla de Vendedores
```sql
CREATE TABLE Vendedores (
    VendedorID INT PRIMARY KEY,
    Nombre NVARCHAR(100),
    Email NVARCHAR(100),
    Telefono NVARCHAR(15),
    CiudadID INT
);
```

### 3. Tabla de Ciudades
```sql
CREATE TABLE Ciudades (
    CiudadID INT PRIMARY KEY,
    NombreCiudad NVARCHAR(100),
    Estado NVARCHAR(100),
    Pais NVARCHAR(100)
);
```

### 4. Tabla de Ventas
```sql
CREATE TABLE Ventas (
    VentaID INT PRIMARY KEY,
    ProductoID INT,
    VendedorID INT,
    CiudadID INT,
    FechaVenta DATE,
    Cantidad INT,
    TotalVenta DECIMAL(10, 2)
);
```
## Consideraciones
Las claves primarias (ProductoID, VendedorID, CiudadID, VentaID) son enteros que se autoincrementan si se desea (esto depende de la configuración de la base de datos en SQL Server, no especificada aquí).
La tabla de ventas vincula productos, vendedores y ciudades a través de sus respectivos IDs.
Las tablas están diseñadas para mantener una simple relación de datos que pueda representar la actividad de ventas de una empresa.
La tabla de ventas vincula productos, vendedores y ciudades a través de sus respectivos IDs.
Las tablas están diseñadas para mantener una simple relación de datos que pueda representar la actividad de ventas de una empresa.

## Implementación en SQL Server
Para mejorar la funcionalidad y gestionar la integridad de datos en un sistema que incluye tablas de Productos, Ventas, Vendedores y Ciudades, podemos considerar la implementación de varios procedimientos almacenados y disparadores (triggers). 
### Procedimiento almacenado para agregar productos
```sql
CREATE PROCEDURE spAgregarProducto
    @Nombre NVARCHAR(100),
    @Descripcion NVARCHAR(255),
    @Precio DECIMAL(10, 2),
    @CantidadStock INT,
    @Categoria NVARCHAR(50)
AS
BEGIN
    INSERT INTO Productos (Nombre, Descripcion, Precio, CantidadStock, Categoria)
    VALUES (@Nombre, @Descripcion, @Precio, @CantidadStock, @Categoria);
END;
GO
```
### Disparador para verificar Stock al realizar una venta

```sql
CREATE TRIGGER trgVerificarStock
ON Ventas
AFTER INSERT
AS
BEGIN
    DECLARE @stock INT;
    SELECT @stock = CantidadStock FROM Productos WHERE ProductoID = inserted.ProductoID;
    IF @stock < inserted.Cantidad
        RAISERROR ('No hay suficiente stock para completar la venta.', 16, 1);
    UPDATE Productos
    SET CantidadStock = CantidadStock - inserted.Cantidad
    WHERE ProductoID = inserted.ProductoID;
END;
GO

```

## Modelo Estrella
![Diagrama ETL](https://github.com/jmorejonm/ETLSAP/blob/main/diagSTARSAP.png)
El diseño del almacén de datos utiliza un modelo estrella para análisis.
```sql
-- Fact Table for Sales
CREATE TABLE FactVentas (
    FactVentaID INT PRIMARY KEY,
    ProductoID INT,
    VendedorID INT,
    CiudadID INT,
    FechaVenta DATE,
    Cantidad INT,
    TotalVenta DECIMAL(10, 2),
    FOREIGN KEY (ProductoID) REFERENCES Productos(ProductoID),
    FOREIGN KEY (VendedorID) REFERENCES Vendedores(VendedorID),
    FOREIGN KEY (CiudadID) REFERENCES Ciudades(CiudadID)
);
```
```sql
-- Dimension Table for Products
CREATE TABLE DimProductos (
    ProductoID INT PRIMARY KEY,
    Nombre NVARCHAR(100),
    Descripcion NVARCHAR(255),
    Precio DECIMAL(10, 2),
    CantidadStock INT,
    Categoria NVARCHAR(50)
);
```
```sql
-- Dimension Table for Vendors
CREATE TABLE DimVendedores (
    VendedorID INT PRIMARY KEY,
    Nombre NVARCHAR(100),
    Email NVARCHAR(100),
    Telefono NVARCHAR(15),
    CiudadID INT
);
```
```sql
-- Dimension Table for Cities
CREATE TABLE DimCiudades (
    CiudadID INT PRIMARY KEY,
    NombreCiudad NVARCHAR(100),
    Estado NVARCHAR(100),
    Pais NVARCHAR(100)
);
```
## Copy Data entre diferentes servers
![Diagrama COPYDATA](https://github.com/jmorejonm/ETLSAP/blob/main/diagCOPYDATA.png)

Para la comunicacion entre bases de datos entre diferentes servidores en el mismo dominio, se debe de crear Linked Server, dejo un ejemplo de comunicación

## Pasos para configurar el LS
Crear un Linked Server: Esto se hace generalmente en el SQL Server Management Studio (SSMS) o mediante scripts T-SQL. Aquí hay un ejemplo de cómo se podría configurar un Linked Server usando T-SQL:

```sql
EXEC sp_addlinkedserver
    @server='TARGETSERVER', -- Nombre del Linked Server
    @srvproduct='', 
    @provider='SQLNCLI', -- SQL Server Native Client
    @datasrc='nombre_del_servidor_destino'; -- Nombre del servidor destino

```
```sql
EXEC sp_addlinkedsrvlogin
    @rmtsrvname='TARGETSERVER',
    @useself='FALSE',
    @locallogin=NULL,
    @rmtuser='usuario_remoto',
    @rmtpassword='contraseña';

```
## Script de ETL para Transferir Datos
Ejemplo de cómo podría ser un procedimiento almacenado que copia datos desde una tabla en el servidor local a una tabla en el servidor remoto:
```sql
CREATE PROCEDURE TransferDataAS
AS
BEGIN
    -- Suponiendo que la tabla destino tiene el mismo esquema que la tabla origen
    INSERT INTO [TARGETSERVER].database_name.schema_name.tabla_destino (columnas)
    SELECT columnas
    FROM tabla_origen;
    
    -- Mensaje opcional para confirmar la ejecución
    PRINT 'Datos transferidos exitosamente.';
END;
GO

EXEC TransferDataAS;
```

## Consideraciones Importantes
* Asegúrate de tener los permisos necesarios en ambos servidores.
* Verifica que los tipos de datos y las estructuras de las tablas coincidan.
* Considera la posibilidad de manejar errores y reintentos dentro del procedimiento almacenado.
* Este script es bastante simple y no incluye transformaciones de datos, que pueden ser necesarias dependiendo de tu situación específica.