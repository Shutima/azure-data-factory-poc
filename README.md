Initialized by Azure Data Factory!
## Introduction

This project is a POC for testing the functionalities of Azure Data Factory in order to adapt with the requirements from the problem statement.

### What is Azure Data Factory?

Azure Data Factory is a cloud-based data integration service for creating ETL (Extract, Transform and Load) and ETL pipelines. It allows users to create data processing workflows in the cloud, either through a graphical interface or by writing code, for orchestrating and automating data movement and data transformation. It is possible to build complex ETL processes that transform data visually with data flows or by using compute services such as Azure HDInsight Hadoop, Azure Databricks, and Azure SQL Database.

### Requirement

- Create, run and manage pipelines for data transfer between Azure Virtual Machine (simulation of the edge machine at the factories) and Azure Blob storage.
- After successfully transfer data from Azure Virtual Machine to Blob Storage, the blob filenames need to be extracted, then inserted into Azure SQL database table. The blob filenames being inserted into the table as per the following columns format:
For example, blob file name `RUSPE1ROW1ST_00000000_20200323-065617_back_camera_0.jpeg`
1. **RU** is CountryCode (corresponds to **Russia**)
2. **SPE** is City (corresponds to **Saint Petersburg**)
3. **RUSPE1ROW1ST** is ProductionLine
4. **00000000** is Barcode
5. **20200323** is Date
6. **065617** is Number of Seat
7. **back_camera** is Camera Position
- Automation of the pipeline to be able to check the state of the last pipeline run.

## Getting Started

1. Required to have Azure Virtual Machine with Windows Server to host Azure Data Factory.
2. On the Azure Windows VM, some production raw data were copied into this VM to simulate the scenario of raw data at the edge machine. We use azcopy tool to perform the data transfer from Blob storage to Azure VM.

>azcopy copy "https://`storage-account-name`.`blob or dfs`.core.windows.net/`container-name`/`blob-path`?`SASToken`" "`local-file-path`" --recursive 

3. Once we have the raw data on Azure Windows VM, we create different pipelines to transfer these data to test container on Azure Blob Storage, **media-dfpoc1**.

## Build and Test
- Navigate to Azure Data Factory portal, [data-factory-qualif](https://portal.azure.com/#@Faurecia.onmicrosoft.com/resource/subscriptions/520c16d8-dd43-4109-b271-8e427f58edd5/resourceGroups/rg-dev-app-aivi-001-data/providers/Microsoft.DataFactory/factories/data-factory-qualif/overview). This Azure Data Factory is running on the Windows Server Virtual Machine. Click on Author & Monitor to open the editor mode of Azure Data Factory.

![ADF1](https://user-images.githubusercontent.com/57285863/97153217-6c8fb500-1772-11eb-84dd-1d4045c94606.png)

- Navigate to Author menu on the left to view and edit Azure Data Factory Pipeline

![ADF2](https://user-images.githubusercontent.com/57285863/97153262-7d402b00-1772-11eb-8676-629637e991af.png)

- Under the left menu `Factory Resources` is where you can find all the available pipelines.

![ADFnew1](https://user-images.githubusercontent.com/57285863/97582943-a5d85700-19f6-11eb-8c88-87d895014787.png)

For this project, I propose 2 solutions for Blob filenames extraction and insert into SQL tables.

### **Solution 1** Using Azure Data Factory Functions and Expression in Combination with the Stored Procedure
More details about [Expressions and Functions in Azure Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/control-flow-expression-language-functions#:~:text=%20Function%20reference%20%201%20addDays.%20Add%20a,the%20result%20string.%20This%20function%20is...%20More%20)

- For this solution, we have to open the pipeline `PLZ_BlobNameExtraction_POC`

![ADFnew2](https://user-images.githubusercontent.com/57285863/97583151-e89a2f00-19f6-11eb-8bd3-e1cd182871be.png)

This pipeline has the following workflow:

- **TransferFilesFromVMtoBlob**: this will copy all binary files (both *jpeg and *json files) from File System in Azure Virtual Machine (act like an Edge Machine at the Factory). Then, it will transfer these binary files to Blob storage at a specific location, **output**
  
- **Get_Filenames_InBlob**: this will take the dataset in the above output folder, **output**, then it will get their metadata, `Child Items`. With this metadata options, the activity will retrieve the list of subfolders and files in the given folder. The returned value is a list of the name and type of each child item.
More details on [Get Metadata activity in Azure Data Factory](https://docs.microsoft.com/en-us/azure/data-factory/control-flow-get-metadata-activity)

![ADFnew3](https://user-images.githubusercontent.com/57285863/97584267-251a5a80-19f8-11eb-9cae-01b165495ff4.png)

- **FilterForJSONfiles**: this filter activity will filter for *json files. It uses the same logic as the previous Filter activity.

![ADFnew4](https://user-images.githubusercontent.com/57285863/97584496-63b01500-19f8-11eb-8174-2748cc80be13.png)

- **FilterForJPGfiles**: this filter activity will filter for *jpeg files. It takes as an input the output.childitems from the previous Get Metadata activity. And, it uses ADF build-in expression `endswith` to filter for the image files.

![ADFnew5](https://user-images.githubusercontent.com/57285863/97584657-893d1e80-19f8-11eb-8def-6edb3245bf77.png)

- **ForEach_JPG**: this activity contains:
  - **Input Settings**: it takes as an input the output of child items from the previous Get Metadata activity with the expression:

  >@activity('FilterForJPGfiles').output.value
    
  ![ADFnew6](https://user-images.githubusercontent.com/57285863/97585128-15e7dc80-19f9-11eb-9cea-f6822bcaf879.png)

  - **Activities**: there is only 1 activity in the ForEach_JPG, which is the **Stored_Procedure_Img_Files**. This Stored Procedure activity is connected to the linked service Azure SQL Database and uses the pre-written stored procedure called `InsertDataJSON2` in order to insert the Blob filenames into SQL Database table. It also uses Azure Data Factory Expression to extract different parts of Blob filename and insert them into the correct columns using `substring` function.

  ![ADFnew7](https://user-images.githubusercontent.com/57285863/97585385-619a8600-19f9-11eb-8554-765d32b52801.png)


  This is the stored procedure `InsertDataJSON2`
  ```sql
  /****** Object:  StoredProcedure [dbo].[InsertDataJSON2] Script Date: 26/10/2020 11:19:51 ******/
  SET ANSI_NULLS ON
  GO
  SET QUOTED_IDENTIFIER ON
  GO
    
  CREATE OR ALTER PROCEDURE [dbo].[InsertDataJSON2] (
	  @JsonDataBlobFileName NVARCHAR (MAX),
    @JsonDataCountryCode NVARCHAR (MAX),
	  @JsonDataPlantName NVARCHAR (MAX),
	  @JsonDataProductionLine NVARCHAR (MAX),
	  @JsonDataBarcode NVARCHAR (MAX),
	  @JsonDataDate NVARCHAR (MAX),
	  @JsonDataNumberOfSeat NVARCHAR (MAX),
	  @JsonDataCameraPosition NVARCHAR (MAX)
  )
  AS
  BEGIN
    
  IF NOT EXISTS (SELECT * FROM extractFileNameTest2 e 
			WHERE e.BlobFileName = @JsonDataBlobFileName)
	INSERT INTO extractFileNameTest2 values (@JsonDataBlobFileName, @JsonDataCountryCode, @JsonDataPlantName, @JsonDataProductionLine, @JsonDataBarcode, @JsonDataDate, @JsonDataNumberOfSeat, @JsonDataCameraPosition)
    
  END
  ````

- **ForEach_JSON**: this activity contains:
  - **Input Settings**: it takes as an input the output of child items from the previous Get Metadata activity with the expression:
  >@activity('FilterForJSONfiles').output.value

  ![ADFnew8](https://user-images.githubusercontent.com/57285863/97585612-a0304080-19f9-11eb-96e0-d4a9c3907d4d.png)

  - **Activities**: there is only 1 activity in the ForEach_JSON, which is the **Stored_Procedure_JSON_files**. This Stored Procedure activity is connected to the linked service Azure SQL Database and uses the pre-written stored procedure called `InsertDataJSONFormat`. This stored procedure is different than the previous one because all json dataset filenames do not contain the camera position. Therefore, we need to manage json files separately.
  
  ![ADFnew9](https://user-images.githubusercontent.com/57285863/97585747-c81fa400-19f9-11eb-8811-f9f03f985835.png)
  
  This is the stored procedure `InsertDataJSONFormat`

  ```sql
  /****** Object:  StoredProcedure [dbo].[InsertDataJSONFormat]    Script Date: 26/10/2020 11:20:56 ******/
  SET ANSI_NULLS ON
  GO
  SET QUOTED_IDENTIFIER ON
  GO

  CREATE OR ALTER PROCEDURE [dbo].[InsertDataJSONFormat] (
    @JsonDataBlobFileName NVARCHAR (MAX),
    @JsonDataCountryCode NVARCHAR (MAX),
	  @JsonDataPlantName NVARCHAR (MAX),
	  @JsonDataProductionLine NVARCHAR (MAX),
	  @JsonDataBarcode NVARCHAR (MAX),
	  @JsonDataDate NVARCHAR (MAX),
	  @JsonDataNumberOfSeat NVARCHAR (MAX)
  )
  AS
  BEGIN

  IF NOT EXISTS (SELECT * FROM extractFileNameTest2 e 
				WHERE e.BlobFileName = @JsonDataBlobFileName)
    INSERT INTO extractFileNameTest2 values (@JsonDataBlobFileName, @JsonDataCountryCode, @JsonDataPlantName, @JsonDataProductionLine, @JsonDataBarcode, @JsonDataDate, @JsonDataNumberOfSeat, DEFAULT)

  END
  ````
### **Solution 1 - Result**: Here is the final SQL table displaying the Blob filenames extraction and insertion into the corresponding column names. Notice that the stored procedure avoids the insertion of duplicate Blob filenames.

![SQL1](https://user-images.githubusercontent.com/57285863/97199423-47ba3280-17b0-11eb-9f3e-58dfe6298d63.png)

### **Solution 2** Using ONLY SQL in Stored Procedure
- For this solution, we have to open the pipeline `PLZ_BlobNameExtraction_POC_usingSQL`

![ADFsqlNew1](https://user-images.githubusercontent.com/57285863/97586501-a1ae3880-19fa-11eb-8004-c98867ef3d80.png)
This pipeline contains the following workflow:
- **TriggerTransferDataToBlob**: this Copy Data activity will transfer all dataset from file system on Azure Virtual Machine to Blob storage
- **Get_BlobFilenames**: this Get Metadata activity will take the dataset in the above output folder, **test_input**, then it will get their metadata, `Child Items`.

![ADFsqlNew2](https://user-images.githubusercontent.com/57285863/97586962-2a2cd900-19fb-11eb-88b1-d3cb5f3238ae.png)

- **ForEach_BlobFilename**: this activity contains:
    - **Input Settings**: it takes as an input the output of child items from the previous Get Metadata activity with the expression:
    >@activity('Get_BlobFilenames').output.childItems

    - **Activities**: there is only 1 activity in the ForEach1, which is the **Stored_procedure_InsertBlobName**. This Stored Procedure activity is connected to the linked service Azure SQL Database and uses the pre-written stored procedure called `InsertBlobFileDetails`. It contains only 1 stored procedure parameter because for this solution, we manage json string inside stored procedure with SQL query and NOT with Azure Data Factory expressions.


    ![ADFsqlNew3](https://user-images.githubusercontent.com/57285863/97587382-9e677c80-19fb-11eb-9896-4fdbc056f062.png)

    This is the stored procedure `InsertBlobFileDetails`

    ```sql
    /****** Object:  StoredProcedure [dbo].[InsertBlobFileDetails]    Script Date: 26/10/2020 11:22:32 ******/
    SET ANSI_NULLS ON
    GO
    SET QUOTED_IDENTIFIER ON
    GO
    CREATE OR ALTER PROCEDURE [dbo].[InsertBlobFileDetails] (
	    @BlobFileName NVARCHAR (255)
    )
    AS
    BEGIN
	-- SET NOCOUNT ON added to prevent extra result sets from
	-- interfering with SELECT statements.
	SET NOCOUNT ON;

	DECLARE @extpos int;
	DECLARE @pos int;
	DECLARE @endpos int;
	DECLARE @CountryCode NVARCHAR (255);
	DECLARE @PlantName NVARCHAR (255);
	DECLARE @ProdLine NVARCHAR (255);
	DECLARE @BarCode NVARCHAR (255);
	DECLARE @FileType NVARCHAR (255);
	DECLARE @EventDate DATE;
	DECLARE @NumberOfSeat NVARCHAR(255);
	DECLARE @CameraPosition NVARCHAR (255);

	SET @CountryCode = LEFT(@BlobFileName, 2);
	SET @PlantName = SUBSTRING(@BlobFileName, 3, 3);
	SET @ProdLine = LEFT(@BlobFileName, 13);

	SET @extpos = CHARINDEX('.', @BlobFileName, 0) + 1;
	SET @FileType = SUBSTRING(@BlobFileName, @extpos, LEN(@BlobFileName) - @extpos + 1);

	SET @pos = CHARINDEX('_', @BlobFileName, 0) + 1;
	SET @BarCode = SUBSTRING(@BlobFileName, @pos, CHARINDEX('_', @BlobFileName, @pos) - @pos);

	SET @pos = CHARINDEX('_', @BlobFileName, @pos) + 1;
	SET @EventDate = CONVERT(DATE, SUBSTRING(@BlobFileName, @pos, CHARINDEX('-', @BlobFileName, @pos) - @pos), 112);

	SET @pos = CHARINDEX('-', @BlobFileName, @pos) + 1;
	SET @endpos = CHARINDEX('_', @BlobFileName, @pos) + 1;
	IF @endpos = 1
		SET @NumberOfSeat = SUBSTRING(@BlobFileName, @pos, @extpos - @pos - 1);
	ELSE
		BEGIN
			SET @NumberOfSeat = SUBSTRING(@BlobFileName, @pos, @endpos - @pos - 1);
			SET @CameraPosition = SUBSTRING(@BlobFileName, @endpos, @extpos - @endpos - 1);
		END
	
	IF NOT EXISTS (SELECT * FROM extractFileNameTest3 e 
				WHERE e.BlobFileName = @BlobFileName)
	INSERT INTO extractFileNameTest3 values (@BlobFileName, @CountryCode, @PlantName, @ProdLine, @FileType, @BarCode, @EventDate, @NumberOfSeat, @CameraPosition)
    END
    ```
### **Solution 2 - Result**: Here is the final SQL table

![SQL2](https://user-images.githubusercontent.com/57285863/97285091-138f5200-1842-11eb-89e3-7bc7a0b5e537.png)