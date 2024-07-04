# Fabric-Summer-Party

## Lab 1 - Ingestion ##

### Base de données nopcommerce ###

- Serveur : nopcommerceoltpserv.database.windows.net
- Base : nopcommerceoltp
- Authentification : Compte professionnel
- Tables : Customer, Order, OrderItem, Product

### Fichier Excel hébergé sur SharePoint ###

<https://sauget.sharepoint.com/sites/AmazingZone/Shared%20Documents/Finance/Budget.xlsx>

### Code M de la table Budget ### 

```
let
  Source = Excel.Workbook(Web.Contents("https://sauget.sharepoint.com/sites/AmazingZone/Shared%20Documents/Finance/Budget.xlsx"), null, true),
  #"Navigation 1" = Source{[Item = "Budget", Kind = "Sheet"]}[Data],
  #"Type de colonne changé" = Table.TransformColumnTypes(#"Navigation 1", {{"Column1", type text}}, "fr"),
  #"Premières lignes supprimées" = Table.Skip(#"Type de colonne changé", 2),
  #"En-têtes promus" = Table.PromoteHeaders(#"Premières lignes supprimées", [PromoteAllScalars = true]),
  #"Autres colonnes supprimées du tableau croisé dynamique" = Table.UnpivotOtherColumns(#"En-têtes promus", {"Country"}, "Attribut", "Valeur"),
  #"Valeur remplacée" = Table.ReplaceValue(#"Autres colonnes supprimées du tableau croisé dynamique", "NULL", null, Replacer.ReplaceValue, {"Valeur"}),
  #"Type de colonne changé 1" = Table.TransformColumnTypes(#"Valeur remplacée", {{"Attribut", type date}, {"Valeur", Currency.Type}}, "en-US"),
  #"Colonnes renommées" = Table.RenameColumns(#"Type de colonne changé 1", {{"Attribut", "Period"}, {"Valeur", "BudgetAmount"}})
in
  #"Colonnes renommées"
```

### Code M de la dimension Date : ###

```
let
    Source = Table.FromRows(Json.Document(Binary.Decompress(Binary.FromText("i45WMjDUByIjAwMDpdhYAA==", BinaryEncoding.Base64), Compression.Deflate)), let _t = ((type nullable text) meta [Serialized.Text = true]) in type table [StartDate = _t]),
    #"Added Custom" = Table.AddColumn(Source, "EndDate", each Date.From(DateTime.LocalNow())),
    #"Changed Type2" = Table.TransformColumnTypes(#"Added Custom",{{"EndDate", type date}}),
    #"Changed Type" = Table.TransformColumnTypes(#"Changed Type2",{{"StartDate", type date}}),
    #"Added Custom1" = Table.AddColumn(#"Changed Type", "Dates", each {Number.From([StartDate])..Number.From([EndDate])}),
    #"Expanded Dates" = Table.ExpandListColumn(#"Added Custom1", "Dates"),
    #"Changed Type1" = Table.TransformColumnTypes(#"Expanded Dates",{{"Dates", type date}}),
    #"Removed Columns1" = Table.RemoveColumns(#"Changed Type1",{"StartDate", "EndDate"}),
    #"Added Year" = Table.AddColumn(#"Removed Columns1", "Year", each Date.Year([Dates])),
    #"Added Month" = Table.AddColumn(#"Added Year", "Month", each Date.Month([Dates])),
    #"Added MonthName" = Table.AddColumn(#"Added Month", "MonthName", each Date.MonthName([Dates])),
    #"Added ShortMonthName" = Table.AddColumn(#"Added MonthName", "ShortMonthName", each Text.Start([MonthName],3)),
    #"Added Quarter" = Table.AddColumn(#"Added ShortMonthName", "Quarter", each Date.QuarterOfYear([Dates])),
    #"Changed Type3" = Table.TransformColumnTypes(#"Added Quarter",{{"Quarter", type text}}),
    #"Added QtrText" = Table.AddColumn(#"Changed Type3", "QtrText", each "Qtr "& [Quarter]),
    #"Added Custom Column" = Table.AddColumn(#"Added QtrText", "DateId", each Text.Combine({Date.ToText([Dates], "yyyy"), "0", Date.ToText([Dates], "%M"), Date.ToText([Dates], "dd")}), type text),
    #"Type de colonne changé" = Table.TransformColumnTypes(#"Added Custom Column", {{"Year", Int64.Type}, {"Month", Int64.Type}, {"MonthName", type text}, {"ShortMonthName", type text}, {"Quarter", Int64.Type}, {"QtrText", type text}})
in
    #"Type de colonne changé"
```


## Lab 2 - Transformation et modélisation ##

### Création des tables DimClient et  DimProduit ###
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
### Dates ###
```sql
SELECT [Dates]
            ,[Year]
            ,[Month]
            ,[MonthName]
            ,[ShortMonthName]
            ,[Quarter]
            ,[QtrText]
            ,[DateId]
FROM [AmazingZoneLH].[dbo].[Calendrier]
```
### Mesures ###
```sql
CA = SUMX(FaitCommande
	, FaitCommande[Quantité] * FaitCommande[PrixUnitaireTTC]
)

CA LY = CALCULATE(
    [CA],
    SAMEPERIODLASTYEAR(DimDate[Dates]
)

Quantité Totale = SUM(FaitCommande[Quantité])
```
## Lab Temps réel ##
```
.create function 
with (docstring = 'SummerLove', folder='Views')
LogsView(){
Logs
| mv-expand records
| extend dateLog = records.['time']
        , Type = records.['Type']
        , Success = records.['Success']
        , ResultCode = records.['ResultCode']
        , DurationMs = records.['DurationMs']
        , ClientCity = records.['ClientCity']
        , ClientCountryOrRegion = records.['ClientCountryOrRegion']
| where Type == "AppRequests"
| project todatetime(dateLog), tostring(Type) ,tostring(ResultCode), todouble(DurationMs), toboolean(Success), ClientCity, ClientCountryOrRegion
}
```
## Lab 3 Visualisation & Copilot ##
![image](./Fab%20summer%20party%20background.png)

## Lab 4 - Data Science ##
[fraud_detection_clean.ipynb](./fraud_detection_clean.ipynb)

