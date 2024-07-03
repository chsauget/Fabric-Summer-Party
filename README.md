# Fabric-Summer-Party

## Lab 2 - Transformation et modélisation ##

### Création des tables DimClient et  DimProduit###
```sql
 DROP TABLE IF EXISTS [dwh].[DimClient];

CREATE TABLE [dwh].[DimClient]
(
	[ClientId] [int]  NULL,
	[ClientSourceId] [int]  NULL,
	[Identifiant] [varchar](8000)  NULL,
	[Email] [varchar](8000)  NULL,
	[Prenom] [varchar](8000)  NULL,
	[Nom] [varchar](8000)  NULL,
	[Entreprise] [varchar](8000)  NOT NULL,
	[CodePostal] [varchar](8000)  NULL,
	[Ville] [varchar](8000)  NULL,
	[Pays] [varchar](8000)  NULL,
	[EstActif] [bit]  NULL,
	[EstSupprime] [bit]  NULL
)
GO; 

 DROP TABLE IF EXISTS [dwh].[DimProduit];

CREATE TABLE [dwh].[DimProduit]
(
	[ProduitId] [int]  NULL,
	[Nom] [varchar](8000)  NULL,
	[Référence] [varchar](8000)  NULL,
	[Description] [varchar](8000)  NULL,
	[Description Complète] [varchar](8000)  NULL,
	[Exonéré de Taxes] [bit]  NULL,
	[Quantité en Stock] [int]  NULL,
	[Prix] [decimal](18,4)  NULL
)
GO
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
END;
GO
CREATE OR ALTER PROCEDURE [dwh].[PsDimProduit]
AS
BEGIN
    DELETE FROM [dwh].[DimProduit]

    INSERT INTO [dwh].[DimProduit] (
        [ProduitId]
        ,[Nom]
        ,[Référence]
        ,[Description]
        ,[Description Complète]
        ,[Exonéré de Taxes]
        ,[Quantité en Stock]
        ,[Prix]
    )
    SELECT 
        [ProduitId] = p.[Id],
        [Nom] = p.[Name],
        [Référence] = p.[Sku],
        [Description] = p.[ShortDescription],
        [Description Complète] = p.[FullDescription],
        [Exonéré de Taxes] = p.[IsTaxExempt],
        [Quantité en Stock] = p.[StockQuantity],
        [Prix] = p.[Price]
    FROM [AmazingZoneLH].[dbo].[dbo_Product] p
END;
GO
CREATE OR ALTER   PROCEDURE [dwh].[PsFaitCommande]
AS
BEGIN
    DROP TABLE IF EXISTS [dwh].[FaitCommande]

    SELECT
        ItemCommandeSourceId = ordit.[Id]
        ,ProductSourceId = ordit.[ProductId]
        ,ProduitId = pdt.[ProduitId]
        ,Quantité = ordit.[Quantity]
        ,PrixUnitaireTTC = ordit.[UnitPriceInclTax]
        ,[CommandeSourceId] = ord.[Id]
        ,[ClientId] = cli.[ClientId]
        ,[ClientSourceId] = ord.[CustomerId]
        ,[DateCommandeId] = CONVERT(VARCHAR(8), ord.[CreatedOnUtc], 112)
        ,[DateCommande] = ord.[CreatedOnUtc]
        ,[DatePaiementId] = CONVERT(VARCHAR(8), ord.[PaidDateUtc], 112)
        ,[DatePaiement] = ord.[PaidDateUtc]
        ,[TauxRemise] = ord.[OrderDiscount]
        ,[MontantTotal] = ord.[OrderTotal]
        ,[Device] = ord.[CustomerCurrencyCode]
        ,[ModeLivraison] = ord.[ShippingMethod]
        ,[EstSupprime] = ord.[Deleted]
    INTO [dwh].[FaitCommande] --> Création automatique de la table qui a été supprimée
    FROM [AmazingZoneLH].[dbo].[dbo_OrderItem] ordit
    LEFT JOIN [AmazingZoneLH].[dbo].[dbo_Order] ord ON ord.[Id] = ordit.[OrderId] 
    INNER JOIN [dwh].[DimClient] cli ON ord.[CustomerId] = cli.[ClientSourceId]
    INNER JOIN [dwh].[DimProduit] pdt ON ordit.[ProductId] = pdt.[ProduitId]
END;
```
### Vue Fait Budget ###
```sql
SELECT      [Country]
            ,[Period]
            ,[BudgetAmount]
FROM [AmazingZoneLH].[dbo].[ExcelBudget]
```

### Notebook Dim Produit ###

[CreateDimProduit.ipynb](./CreateDimProduit.ipynb)

### Create view DimProduitNBK ###
```sql
CREATE VIEW [dwh].[DimProduit_nbk] AS (SELECT [Id],
        [ProduitId] = p.[Id],
        [Nom] = p.[Name],
        [Référence] = p.[Sku],
        [Description] = p.[ShortDescription],
        [Description Complète] = p.[FullDescription],
        [Exonéré de Taxes] = p.[IsTaxExempt],
        [Quantité en Stock] = p.[StockQuantity],
        [Prix] = p.[Price]
FROM [AmazingZoneLH].[dbo].[dimProduit_nbk] p)
```




## Lab 4 - Data Science ##
[fraud_detection_clean.ipynb](./fraud_detection_clean.ipynb)

