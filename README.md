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

CREATE OR ALTER PROCEDURE [dwh].[PsDimDate] 
    @PremiereDate DATE = '2022-01-01',
    @DerniereDate DATE
AS
BEGIN
 DROP TABLE IF EXISTS [dwh].[PsDimDate]
    DECLARE @IterateurDate DATE = @PremiereDate;

    WHILE @IterateurDate <= @DerniereDate
    BEGIN
        INSERT INTO [dwh].[DimDate] (
            [DateId]
            ,[Date]
            ,[Annee]
            ,[CodeAnneeMois]
            ,[CodeMois]
            ,[MoisCourt]
            ,[MoisLong]
            ,[Trimestre]
            ,[CodeTrimestre]
            ,[CodeAnneeTrimestre]
            ,[Semaine]
            ,[CodeSemaine]
            ,[CodeAnneeSemaine]
            ,[JourSemaine]
            ,[Jour]
            ,[JourCourt] 
            ,[JourLong]
        )
        --DECLARE @IterateurDate DATE = '2023-09-01';
        SELECT
            [DateId] = CONVERT(VARCHAR(8), @IterateurDate, 112)
            ,[Date] = CAST(@IterateurDate AS DATE)
            ,[Annee] = DATEPART(YEAR, @IterateurDate)
            ,[CodeAnneeMois] = FORMAT(@IterateurDate, 'yyyy-MM')
            ,[CodeMois] = FORMAT(@IterateurDate, 'MM')
            ,[MoisCourt] = FORMAT(@IterateurDate, 'MMM', 'fr-fr')
            ,[MoisLong] = FORMAT(@IterateurDate, 'MMMM', 'fr-fr')
            ,[Trimestre] = DATEPART(QQ, @IterateurDate)
            ,[CodeTrimestre] = CONCAT('Q', DATEPART(QQ, @IterateurDate))
            ,[CodeAnneeTrimestre] = CONCAT(DATEPART(YEAR, @IterateurDate), '-Q', DATEPART(QQ, @IterateurDate))
            ,[Semaine] = DATEPART(WK, @IterateurDate)
            ,[CodeSemaine] = CONCAT('S', DATEPART(WK, @IterateurDate))
            ,[CodeAnneeSemaine] = CONCAT(DATEPART(YEAR, @IterateurDate), '-S', DATEPART(WK, @IterateurDate))
            ,[JourSemaine] = DATEPART(DW, @IterateurDate)
            ,[Jour] = DATEPART(DD, @IterateurDate)
            ,[JourCourt] = FORMAT(@IterateurDate, 'ddd', 'fr-fr')
            ,[JourLong] = FORMAT(@IterateurDate, 'dddd', 'fr-fr')
        WHERE NOT EXISTS (SELECT 1 FROM [dwh].[DimDate] WHERE [Date] = @IterateurDate)
            
        SET @IterateurDate = DATEADD(DAY, 1, @IterateurDate);
    END;
END;


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
END

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
END
```


