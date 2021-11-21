# Laboratorio 5 - ETL con manejo de historia

## Objetivos
- Definir un proceso ETL incremental con manejo de historia en los atributos de las dimensiones
- Identificar y aplicar los tipos de manejo de historia a nivel de atributos de dimensiones
- Cargar una serie de datos a partir de unos archivos csv manejando carga incremental

## Herramientas
- PostgreSQL
- Pentaho PDI (Data Integration) de forma local
- Google Cloud BigQuery o una herramienta de ETL

## Preparación
- Para la realización de este laboratorio es necesario contar con **las herramientas del laboratorio 4 y el resultado del mismo**. Se recomienda tener una copia de ese laboratorio para no afectar lo que entregó y se va a calificar, con lo que se va a trabajar en este laboratorio..
En este punto recuerde generar una copia de los jobs y transformations utilizadas en el laboratorio 4.

## Enunciado
Este laboratorio tiene dos partes que son equivalentes y corresponden al uso de herramientas específicas para realizar procesos de ETL, o de propósito general, las cuales son respectivamente: 1) Spoon de la suit de Pentaho y 2) Google Cloud, en especial: CloudStorage y BigQuery o la herramienta que desee utilizar. 

Al final deben realizar una comparación de las herramientas de ETL utilizadas y sacar conclusiones al respecto. Puede usar la combinación de las dos herramientas que desee (ej. Spoon y BigQuery, BigQuery y Python).
Se sugiere el trabajo del laboratorio en grupos de **máximo tres personas**.

## Caso de estudio
WWI (World Wide Importers) [^1] es una empresa encargada de realizar importaciones y venderlas a diferentes clientes en diferentes ciudades de Estados Unidos. Actualmente, la empresa se encuentra buscando servicios de consultoría de BI puesto que desean optimizar sus ganancias, pues consideran que algunos de sus productos no están generando las ganancias que deberían. También, están interesados en saber si hay otros factores que le impiden optimizar sus ganancias. 
En la anterior etapa del trabajo, WWI lo contrató a usted y a su equipo para que realizaran una consultoría de BI. En particular, para la creación del modelo relacional y la carga de datos, para la realización consultas sobre los datos.
En esta ocasión, la empresa necesita que se mejore el proceso de ETL para que les permita manejar correctamente los cambios en los datos, y por consecuencia, en las dimensiones del modelo propuesto.

Al finalizar este laboratorio, se espera contar con un proceso ETL que lea nuevos archivos CSV provenientes de la base de datos original (SQLServer) y maneje correctamente el ingreso de nuevos datos o actualización de los previamente cargados.

## A. Proceso relacionado con Spoon

### I. Manejo de historia en la dimensión stockitem

Como se mencionó en la sección de "Preparación", el proceso de este laboratorio parte desde donde finalizó el laboratorio 4.

A continuación, se muestran los cambios que se deben realizar sobre una dimensión para el manejo de la historia:

1. Abrir la transformación correspondiente a la dimensión 'stock_item' `(ej. "transformacion_stock_item.ktr")` y guardar una copia con el nombre `"transformacion_stock_item_historia.ktr"`.  

![imagen 1](images/imagen_1.jpg)

2. Abrir el job principal `(ej. job_dimensiones.kjb)` y deshabilitar todos los flujos distintos al flujo correspondiente a la dimensión stock_item. Para deshabilitar un flujo, hacer clic derecho en el conector y elegir la opción "deshabilitar". El resultado se debe ver así:  

![imagen 2](images/imagen_2.jpg)

- Luego, editar el nodo `Transformation` del flujo de la dimensión stock_item para que el campo `"Transformation:"` apunte hacia la ruta de la nueva transformación.

3. Para realizar el manejo de historia, la tabla de la base de datos correspondiente a la dimensión "stock_item" debe contar con cuatro nuevos campos: `"tk_stock_item"`, `"_version"`, `"date_from"`, `"date_to"`. Estas columnas permiten el manejo tipo 2 para historia de atributos de dimensiones.
- Modificar el nodo "SQL" del flujo para que:
    - Apunte a una nueva tabla en la base de datos llamada `"stockitem_historia"`
    - Elimine el constraint `PRIMARY KEY` de la columna `Stock_item_key`. 
    - Agregue las tres nuevas columnas con los tipos de datos `BIGINT,INT, DATE y DATE` correspondientemente. 
	### Pregunta
	> - ¿Cual es el objetivo de la columna tk_stock_item?. 

La sentencia de SQL completa se deberá ver así:

```
CREATE TABLE IF NOT EXISTS stockitem_historia(
Stock_Item_Key INT,
WWI_Stock_Item_ID INT,
Stock_Item VARCHAR(200),
Color VARCHAR(50),
Selling_Package VARCHAR(50),
Buying_Package VARCHAR(50),
Brand VARCHAR(50),
Size_val VARCHAR(50),
Lead_Time_Days INT,
Quantity_Per_Outer INT,
Is_Chiller_Stock BOOLEAN,
Tax_Rate DECIMAL,
Unit_Price DECIMAL,
Recommended_Retail_Price DECIMAL,
Typical_Weight_Per_Unit DECIMAL,
tk_stock_item INT,
_Version INT,
Date_from DATE,
Date_to DATE
);
```

4. Abrir la transformación `"transformacion_stock_item_historia.ktr"` y eliminar el nodo `Insert/Update`. Luego, insertar el nodo `Dimension lookup/update` que está en la carpeta Data Warehouse, y conectar el lector CSV a éste.  

![imagen 3](images/imagen_3.jpg)

- Configure el nodo para que se conecte con su base de datos PostgreSQL y apunte a la nueva tabla creada `stockitem_historia`

- En la pestaña `Keys` seleccionar el campo correspondiente a la llave primaria de la tabla `stock_item_key`.  

![imagen 4](images/imagen_4.jpg)

- Seleccionar la pestaña `Fields` y hacer clic en el botón `Get Fields` que se encuentra en la parte inferior de la ventana. Verificar el correcto mapeo de los campos obtenidos.  
 
![imagen 5](images/imagen_5.jpg)

- Si se da clic en la columna `Type of dimension update` de alguno de los campos, se abrirá una lista con diferentes opciones. 

### Pregunta
	> - ¿Qué significa cada una de estas opciones? 
	Para profundizar sobre cada uno de los campos de este nodo, leer el siguiente (https://blogs.perficient.com/2015/05/25/handle-slowly-changing-dimensions-with-pentaho-kettle-part1/)

- Configurar los siguientes campos:
    - `Technical key field:` tk_stock_item
    -  `Version field:` _version  

    ![imagen 6](images/imagen_6.jpg)

- Ejecutar el job principal con los datos utilizados para el cargue inicial ([dimension_stock_item.csv](datos/dimension_stock_item.csv)), y verificar, con pgAdmin o su herramienta de preferencia, que la tabla cuente con las nuevas columnas `tk_stock_item, _version, date_from, date_to`, y que se hayan subido los datos correctamente.  

![imagen 7](images/imagen_7.jpg)

- Para probar el manejo de la historia, ir a la transformación `transformacion_stock_item_historia.ktr` y editar el nodo `CSV Input` para que lea el archivo [dimension_stock_item_new.csv](datos/dimension_stock_item_new.csv). 
- Abrir de nuevo el nodo `Dimension Lookup/Update` y hacer clic en el botón `SQL`. Se abrirá un script de SQL que indica las alteraciones sobre la tabla que el nodo realizará. En este script se deben corregir algunas alteraciones automáticas que genera el nodo para que correspondan con las propiedades de la tabla creada `(ej. Cantidad máxima de caracteres de una columna)`. Entonces, ajustar la sentencia SQL para que los tipos de datos de cada columna, y sus propiedades, se ajusten al script de creación de la tabla inicial (ubicado en el job principal), y luego dar la opción de `Execute`.
A continuación, un ejemplo de los cambios automáticos que generó el nodo sobre la tabla **y que deben ser modificados:**  

![Imagen 16](images/imagen_16.jpg)

- Utilice como base para ajustar los alter table sugeridos las siguientes sentencias:
``` 
ALTER TABLE stockitem_historia ADD COLUMN Stock_Item_Key_KTL INT;
UPDATE stockitem_historia SET Stock_Item_Key_KTL=Stock_Item_Key;
ALTER TABLE stockitem_historia DROP COLUMN Stock_Item_Key;
ALTER TABLE stockitem_historia RENAME Stock_Item_Key_KTL TO Stock_Item_Key;

;
ALTER TABLE stockitem_historia ADD COLUMN Stock_Item_KTL VARCHAR(200);
UPDATE stockitem_historia SET Stock_Item_KTL=Stock_Item;
ALTER TABLE stockitem_historia DROP COLUMN Stock_Item;
ALTER TABLE stockitem_historia RENAME Stock_Item_KTL TO Stock_Item;

;
ALTER TABLE stockitem_historia ADD COLUMN Color_KTL VARCHAR(50);
UPDATE stockitem_historia SET Color_KTL=Color;
ALTER TABLE stockitem_historia DROP COLUMN Color;
ALTER TABLE stockitem_historia RENAME Color_KTL TO Color;

;
ALTER TABLE stockitem_historia ADD COLUMN Selling_Package_KTL VARCHAR(50);
UPDATE stockitem_historia SET Selling_Package_KTL=Selling_Package;
ALTER TABLE stockitem_historia DROP COLUMN Selling_Package;
ALTER TABLE stockitem_historia RENAME Selling_Package_KTL TO Selling_Package;

;
ALTER TABLE stockitem_historia ADD COLUMN Buying_Package_KTL VARCHAR(50);
UPDATE stockitem_historia SET Buying_Package_KTL=Buying_Package;
ALTER TABLE stockitem_historia DROP COLUMN Buying_Package;
ALTER TABLE stockitem_historia RENAME Buying_Package_KTL TO Buying_Package;

;
ALTER TABLE stockitem_historia ADD COLUMN Brand_KTL VARCHAR(50);
UPDATE stockitem_historia SET Brand_KTL=Brand;
ALTER TABLE stockitem_historia DROP COLUMN Brand;
ALTER TABLE stockitem_historia RENAME Brand_KTL TO Brand;

;
ALTER TABLE stockitem_historia ADD COLUMN size_val_KTL VARCHAR(50);
UPDATE stockitem_historia SET size_val_KTL=size_val;
ALTER TABLE stockitem_historia DROP COLUMN size_val;
ALTER TABLE stockitem_historia RENAME size_val_KTL TO size_val;

;
ALTER TABLE stockitem_historia ADD COLUMN Lead_Time_Days_KTL INT;
UPDATE stockitem_historia SET Lead_Time_Days_KTL=Lead_Time_Days;
ALTER TABLE stockitem_historia DROP COLUMN Lead_Time_Days;
ALTER TABLE stockitem_historia RENAME Lead_Time_Days_KTL TO Lead_Time_Days;

;
ALTER TABLE stockitem_historia ADD COLUMN Quantity_Per_Outer_KTL INT;
UPDATE stockitem_historia SET Quantity_Per_Outer_KTL=Quantity_Per_Outer;
ALTER TABLE stockitem_historia DROP COLUMN Quantity_Per_Outer;
ALTER TABLE stockitem_historia RENAME Quantity_Per_Outer_KTL TO Quantity_Per_Outer;

;
ALTER TABLE stockitem_historia ADD COLUMN Tax_Rate_KTL NUMERIC(9, 3);
UPDATE stockitem_historia SET Tax_Rate_KTL=Tax_Rate;
ALTER TABLE stockitem_historia DROP COLUMN Tax_Rate;
ALTER TABLE stockitem_historia RENAME Tax_Rate_KTL TO Tax_Rate;

;
ALTER TABLE stockitem_historia ADD COLUMN Unit_Price_KTL NUMERIC(9, 2);
UPDATE stockitem_historia SET Unit_Price_KTL=Unit_Price;
ALTER TABLE stockitem_historia DROP COLUMN Unit_Price;
ALTER TABLE stockitem_historia RENAME Unit_Price_KTL TO Unit_Price;

;
ALTER TABLE stockitem_historia ADD COLUMN Recommended_Retail_Price_KTL NUMERIC(9, 2);
UPDATE stockitem_historia SET Recommended_Retail_Price_KTL=Recommended_Retail_Price;
ALTER TABLE stockitem_historia DROP COLUMN Recommended_Retail_Price;
ALTER TABLE stockitem_historia RENAME Recommended_Retail_Price_KTL TO Recommended_Retail_Price;

;
ALTER TABLE stockitem_historia ADD COLUMN Typical_Weight_Per_Unit_KTL NUMERIC(9, 3);
UPDATE stockitem_historia SET Typical_Weight_Per_Unit_KTL=Typical_Weight_Per_Unit;
ALTER TABLE stockitem_historia DROP COLUMN Typical_Weight_Per_Unit;
ALTER TABLE stockitem_historia RENAME Typical_Weight_Per_Unit_KTL TO Typical_Weight_Per_Unit;

;

```
- Ejecute de nuevo el job principal `(ej. job_dimensiones.kjb)` y compruebe que se hayan subido los nuevos datos usando pgAdmin.
	### Pregunta
	> - ¿Cómo se puede saber que el proceso ETL manejó los cambios entre los dos archivos?` Pista: Puede revisar los valores de las columnas creadas para el manejo de historia.  

    ![imagen 8](images/imagen_8.jpg)

### II. Cambios en la tabla de hechos
Si quiere conservar la versión anterior de la tabla de hechos, le sugerimos crear una copia y trabajar sobre ella lo que sigue del laboratorio.
Es necesario transformar la tabla de hechos con base en las nuevas llaves subrogadas que se generan para el manejo de historia. 
En el caso de la dimensión stockitem_historia, se hará uso de la columna `tk_stock_item`.

1. Desde pgAdmin, otra herramienta de manejo de bases de datos o sentencias SQL, modificar los `Constraints` de la tabla de la dimensión `stockitem_historia`  para que la columna `tk_stock_item` sea `Unique`.  
![Imagen 9](images/imagen_9.jpg)
    - Si lo hace directamente sobre la base de datos, puede utilizar esta sentencia SQL:
```
alter table stockitem_historia add constraint uk_stock_item UNIQUE (tk_stock_item);
```
### Pregunta
> - ¿Por qué se debe hacer el cambio para que la columna de la llave subrogada sea de tipo `Unique`?

2. En el job principal `(ej. job_dimensiones.kjb)` modificar el script SQL de la tabla de hechos para que la columna `Stock_item_key` referencie ahora a la llave subrogada `tk_stock_item` de la dimensión.  
![Imagen 10](images/imagen_10.jpg)

3. En el job principal, utilizando un nuevo nodo SQL, crear una tabla idéntica a la tabla de hechos llamada `fact_order_temp`. Esta tabla se utilizará para realizar la transformación de llaves.  

![Imagen 11](images/imagen_11.jpg)

4. Crear una nueva transformación llamadada `transformacion_fact_order_new.ktr` y referenciarla desde el nodo `Transformation` del job principal. Luego, realizar los siguientes pasos dentro de esa transformación:
- Modificar el nodo `CSV Input` para que lea el nuevo archivo [fact_order_new.csv](datos/fact_order_new.csv)
- Insertar los datos provenientes del archivo CSV en la tabla temporal que acaba de crear.
- Añadir un nodo SQL para realizar un join entre la tabla temporal y la dimensión y guardarlo como una nueva tabla `fact_order_join`. Se debe tener en cuenta que el archivo CSV tiene la referencia hacia la llave privada de la dimensión y no hacia la llave subrogada, por lo que se debe cambiar el valor de la llave en la tabla de hechos para que referencie a la llave subrogada. A continuación un ejemplo de la sentencia:
```
DROP TABLE IF EXISTS fact_order_join;
CREATE TABLE fact_order_join AS
SELECT ft.*, tk_stock_item
FROM fact_order_temp ft
JOIN stockitem_historia ON 
(ft.stock_item_key = stockitem_historia.stock_item_key)
WHERE ft.order_date_key between stockitem_historia.date_from and stockitem_historia.date_to;
```

### Pregunta
> - ¿Qué pasa si uno de los datos reportados en la tabla de hechos no existe en alguna de las dimensiones?
> - ¿Qué sugiere para evitar esa situación?

- Utilizando un nodo `Table Input` traer los datos de la nueva tabla `fact_order_join` para luego insertarlos en la tabla `fact_order`.

Para realizar estas tareas es posible utilizar otros nodos, así que el flujo resultado  que se presenta a continuación es una guía, más no una respuesta única:  
![Imagen 12](images/imagen_12.jpg)

5. Finalmente, de vuelta en el job principal, agregar un SQL que elimine las tablas temporales `fact_order_temp` y `fact_order_join`. Ejecute el job.

    - Nota: Si al ejecutar el job principal no hay errores de ejcución, pero no hay registros en la tabla de hechos, volver a ejecutar el job para que se carguen los datos.

El flujo completo del job se vería así:  
![Imagen 13](images/imagen_13.jpg)

- Para probar el resultado de la ejecución, consulte la tabla de hechos `fact_order` y verifique que la columna `stock_item_key` referencia correctamente a la dimensión `stockitem_historia`. A continuación, un ejemplo de registro de la tabla `fact_order`, y su referencia en la tabla `stockitem_historia`:  

![Imagen 14](images/imagen_14.jpg)

![Imagen 15](images/imagen_15.jpg)

6. Reflexione sobre el proceso realizado y saque sus conclusiones.

## Instrucciones de Entrega
- El laboratorio se entrega en grupos de máximo 3 estudiantes
- Recuerde hacer la entrega por la sección unificada en Bloque Neón, antes del sábado 20 de noviembre a las 22:00.   
  Este será el único medio por el cual se recibirán entregas.

## Entregables
1. Diagrama de alto nivel describiendo el proceso de ETL
2. Documente ventajas y desventajas de las maneras propuestas para el manejo de historia y, seleccione la que considere es la más apropiada para el caso que se está trabajando en el laboratorio. 
3. Propuesta de tres maneras diferentes (entre los propuestos por Kimball desde el tipo 2 hasta el tipo 7), de manejar cambios sobre el atributo *category* de la dimensión *Customer*. 
Utilice el archivo [dimension_customer_new.csv](datos/dimension_customer_new.csv) que tiene cambios en el atributo de interés del cliente. 
Implemente los manejos de historia de atributos propuestos en las dos herramientas utilizadas y muestre cómo quedan los nuevos modelos creados en la base de datos posgresql. De ejemplos de las filas que carguen en dichos modelos.
4. Comparación de las herramientas de ETL. Analice las fortalezas y debilidades del uso de estas herramientas para procesos de carga incremental con manejo de historia de atributos de dimensiones y saque sus conclusiones.
5. Responda las preguntas incluidas en este enunciado


## Rúbrica de Calificación

A continuación se encuentra la rúbrica de calificación.

**Nota:** Los siguientes porcentajes hacen referencia a la nota grupal, que corresponde a un 80% de la nota inidividual.  
El 20% restante se calcula según el puntaje obtenido en el proceso ETL utilizando una herramienta, del cual el estudiante estuvo a cargo dentro del grupo.

| Concepto | Porcentaje |
|:---:|:---:|
| Diagrama de alto nivel describiendo el proceso de ETL | 10% |
| Documente ventajas y desventajas de las maneras propuestas para el manejo de historia y, seleccione la que considere es la más apropiada para el caso que se está trabajando en el laboratorio.  | 15% |
| Resultados que demuestran una implementación adecuada para Spoon, ejecución correcta del ETL con evidencia de las tablas cargadas con este proceso y el código asociado a la implementación de 3 tipos distintos para el manejo de historia de atributos| 22,5% |
| Resultados que demuestran una implementación adecuada para la segunda herramienta seleccionada, ejecucición correcta del ETL  con evidencia de las tablas cargadas con este proceso y el código asociado a la implementación de 3 tipos distintos para el manejo de historia de atributos| 22,5% |
| Comparación de las herramientas de ETL. . Analice las fortalezas y debilidades del uso de estas herramientas para procesos de carga incremental con manejo de historia de atributos de dimensiones y saque sus conclusiones. | 15% |
| Respuestas a las preguntas del laboratorio | 15% |

## Sugerencias

- Use la máquina virtual que le asignó admonsis para el curso. En dicha máquina, encuentra instalado pdi y en particular Spoon. 
- En las salas Waira, Pentaho Data Integration está instalado en `C:/Software/data-integration`.

## Referencias
- https://wiki.pentaho.com/display/EAIes/Manual+del+Usuario+de+Spoon
- https://cloud.google.com/solutions/performing-etl-from-relational-database-into-bigquery?hl=it
- [3] Capítulo 20 (desde pág. 512 hasta pág. 520). - [3] KIMBALL, Ralph, ROSS, Margy. “The Data Warehouse Toolkit: the definitive guide to dimensional modeling".  Third Edition.  John Wiley & Sons, Inc, 2013.
- Opc. [6] Capítulo 11 (págs. 464-472) - [6] KIMBALL, Ralph, ROSS, Margy, BECKER, Bob, MUNDY Joy, and THORNTHWAITE, Warren. "The Kimball Group Reader: Relentlessly Practical Tools for Data Warehousing and Business Intelligence Remastered Collection". Wiley, 2016 




