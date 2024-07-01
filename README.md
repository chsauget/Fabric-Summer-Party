# Fabric-Summer-Party

## Lab 2 ##
### Vue Fait Budget ###
```sql
SELECT      [Country]
            ,[Period]
            ,[BudgetAmount]
FROM [AmazingZoneLH].[dbo].[ExcelBudget]
```
### Procédure Entrepot ###

```sql
CREATE OR ALTER  PROCEDURE [dwh].[PsDimClient]
AS
BEGIN
    -- TRUNCATE TABLE n'est encore pas supporté
    DELETE FROM [dwh].[DimClient]

    INSERT INTO [dwh].[DimClient] (
        [ClientId]
        ,[ClientSourceId]
        ,[Identifiant]
        ,[Email]
        ,[Prenom]
        ,[Nom]
        ,[Entreprise]
        ,[CodePostal]
        ,[Ville]
        ,[Pays]
        ,[EstActif]
        ,[EstSupprime]
    )
     SELECT 
        [ClientId] = c.Id
        ,[ClientSourceId] = c.[Id]
        ,[Identifiant] = c.[Username]
        ,c.[Email]
        ,[Prenom] = c.[FirstName]
        ,[Nom] = c.[LastName]
        ,[Entreprise] = ISNULL(c.[Company], '')
        ,[CodePostal] = c.[ZipPostalCode]
        ,[Ville] = c.[City]
        ,[Pays] = c.[County]
        ,[EstActif] = c.[Active]
        ,[EstSupprime] = c.[Deleted]
    FROM [AmazingZoneLH].[dbo].[dbo_Customer] c --> Lakehouse
END
```
