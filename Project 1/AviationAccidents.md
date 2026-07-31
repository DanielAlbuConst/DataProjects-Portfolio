# Project description

IASS (International Alliance for Safe Skies) has a dataset containing approximately 25K aviation accidents.
Their goal is to analyze this data to uncover valuable insights, identify the main risk factors in flight, and ultimately improve aviation safety.
The results of the analysis must be presented in a clear and easy-to-understand dashboard.

**Dataset:** [aviation-accidents.csv](./aviation-accidents-COPY.csv)

**Tools:**
- Power Query
- SQL
- Tableau

# The Analysis Process

**1) Import Data:**

The data is imported from CSV into a SQL database to facilitate the data cleaning process.
A staging table is then created to store the entire dataset, with all columns formatted as VARCHAR.
```sql
-- 1) Staging Table Creation
CREATE TABLE Stage_AviationAccidents(
	NrAccidents VARCHAR(150),
	[Day] VARCHAR(150),
	[Month] VARCHAR(150),
	[Year] VARCHAR(150),
	[Type] VARCHAR(150),
	Registration VARCHAR(150),
	Operator VARCHAR(150),
	Fatalities VARCHAR(150),
	[Location] VARCHAR(150),
	Country VARCHAR(150),
	Category VARCHAR(150)
);

-- 2) Import data into Staging Table
BULK INSERT Stage_AviationAccidents
FROM 'C:\Users\danie\Desktop\aviation-accidents-COPY.csv'
WITH(
	FIRSTROW = 2,
	FIELDTERMINATOR = ','
);
```

**2a) Data Cleaning - Whitespace, BLANK value and NULL:**

Whitespace and BLANK values are identified:

```sql
-- 1) Whitespace identification
SELECT
	[Field],
	LEN([Field]) - LEN(TRIM([Field])) AS [Whitespace]
FROM Stage_AviationAccidents 
WHERE LEN([Field]) - LEN(TRIM([Field])) != 0;

-- 2) BLANK value identification
SELECT
	*
FROM Stage_AviationAccidents
WHERE [Field] = '';
```

No BLANK values were identified, so I proceeded with removing whitespace only:
```sql
-- Whitespace removal
UPDATE Stage_AviationAccidents
SET Operator = TRIM(Operator);
```

**2b) Data Cleaning - Data Consistency:**

I verify that the dates are correctly formatted:
```sql
-- 1) Verify [Day] consistency
SELECT DISTINCT
	[Day]
FROM Stage_AviationAccidents;

-- 2) Verify [Month] consistency
SELECT DISTINCT
	[Month]
FROM Stage_AviationAccidents;

-- 3) Verify [Year] consistency
SELECT DISTINCT
	[Year]
FROM Stage_AviationAccidents;
```

I correct the anomalous Month value: [14]
'14' is searched for as a string because [Month] is formatted as VARCHAR
```sql
UPDATE Stage_AviationAccidents
SET [Month] = NULL
WHERE [Month] = '14';
```

**2c) Data Cleaning - Extra Characters:**

Identification of extra characters within the column [Type], [Registration], [Operator], [Location]:
```sql
-- 1) Identify extra characters in column [Type]
SELECT DISTINCT
	[Type]
FROM Stage_AviationAccidents
WHERE [Type] LIKE '%[^-A-Za-z0-9 .()+?®/''&¿¡]%';

-- 2) Identify extra characters in column [Registration]
SELECT DISTINCT
	Registration
FROM Stage_AviationAccidents
WHERE Registration LIKE '%[^-A-Za-z0-9()+/ ?.]%';

-- 3) Identify extra characters in column [Operator]
SELECT DISTINCT
	[Operator]
FROM Stage_AviationAccidents
WHERE [Operator] LIKE '%[^-A-Za-z0-9 +&.®¦()¡''¬»¿/?!¢©]%';

-- 4) Identify extra characters in column [Location]
SELECT DISTINCT
	[Location]
FROM Stage_AviationAccidents
WHERE [Location] LIKE '%[^-A-Za-z0-9 )(.+®¦¡/''»©¿&*£¢º`]%';
```

[Registration] contains many fully or partially unknown values and is not relevant to the analysis, so it was decided to remove it.
```sql
ALTER TABLE Stage_AviationAccidents
DROP COLUMN [Registration];
```

I replace the identified extra characters with the appropriate ones.
(Since the code is essentially the same for all replacements, only one replacement is shown.)
```sql
UPDATE Stage_AviationAccidents
SET [Type] = REPLACE([Type], '+®', 'é');
```

**2d) Data Cleaning - Final Check:**

-- Check data consistency in the remaining columns
```sql
-- 1) Verify [Fatalities] consistency
SELECT DISTINCT
	[Fatalities]
FROM Stage_AviationAccidents;

-- 2) Verify [Country] consistency
SELECT DISTINCT
	[Country]
FROM Stage_AviationAccidents;

-- 3) Verify [Category] consistency
SELECT DISTINCT
	[Category]
FROM Stage_AviationAccidents;
```

Errors identified:

- [Fatalities] contains values written as sums (e.g. 5+8 instead of 13)
- [Country] contains '?' and 'Unknown country': I decided to standardize both as NULL

STANDARDIZATION OF FIELD [Fatalities]:
```sql
-- 1) Extract values on both sides of '+'
SELECT
	Fatalities,
	LEFT(Fatalities, CHARINDEX('+', Fatalities) - 1) AS [Left],
	SUBSTRING(Fatalities, CHARINDEX('+', Fatalities) + 1, LEN(Fatalities) - CHARINDEX('+', Fatalities)) AS [Right]
FROM Stage_AviationAccidents
WHERE Fatalities LIKE '%+%';

-- 2) Sum the extracted values
UPDATE Stage_AviationAccidents
SET Fatalities = CAST(LEFT(Fatalities, CHARINDEX('+', Fatalities) - 1) AS INT) +
CAST(SUBSTRING(Fatalities, CHARINDEX('+', Fatalities) + 1, LEN(Fatalities) - CHARINDEX('+', Fatalities)) AS INT)
WHERE Fatalities LIKE '%+%';
```

STANDARDIZATION OF FIELD [Country]:
```sql
-- Replace '?' and 'Unknown country' with NULL
UPDATE Stage_AviationAccidents
SET Country = NULL
WHERE Country = '?' OR Country = 'Unknown country';
```

**3) Data type conversion:**

After cleaning the data, it is trasferred into a Target Table, with a new [Date] column created from the [Day], [Month], and [Year] columns.
The first step is to create the Target Table:
```sql
-- Create Target table
CREATE TABLE Target_AviationAccidents(
NrAccidents INT NOT NULL IDENTITY(1, 1) PRIMARY KEY,
[Day] INT,
[Month] INT,
[Year] INT,
[Date] DATE,
[Type] VARCHAR(150),
[Operator] VARCHAR(150),
Fatalities INT,
[Location] VARCHAR(150),
Country VARCHAR(50),
Category CHAR(2)
);
```

Before transfer the data, the [Month] values are converted from text to numeric format, as the [Month] column in the Target Table is defined as INTEGER:
```sql
-- Convert dates to numeric format:
UPDATE Stage_AviationAccidents
SET [Month] =
CASE
	WHEN [Month] = 'JAN' THEN 01
	WHEN [Month] = 'FEB' THEN 02
	WHEN [Month] = 'MAR' THEN 03
	WHEN [Month] = 'APR' THEN 04
	WHEN [Month] = 'MAY' THEN 05
	WHEN [Month] = 'JUN' THEN 06
	WHEN [Month] = 'JUL' THEN 07
	WHEN [Month] = 'AUG' THEN 08
	WHEN [Month] = 'SEP' THEN 09
	WHEN [Month] = 'OCT' THEN 10
	WHEN [Month] = 'NOV' THEN 11
	WHEN [Month] = 'DEC' THEN 12
END;
```

Now we can transer the data:
```sql
-- Data transfer
INSERT INTO Target_AviationAccidents
SELECT
	 [Day],
	 [Month],
	 [Year],
	 DATEFROMPARTS([Year], [Month], [Day]) AS [Date],
	 [Type],
	 Operator,
	 Fatalities,
	 [Location],
	 Country,
	 Category
FROM Stage_AviationAccidents
ORDER BY CAST([Year] AS INT), CAST([Month] AS INT), CAST([Day] AS INT);
```

# Data visualization
Let's define the chart type for each insight:

- Accidents By Country: **Choropleth map**
- NrAccidents By Year: **Line chart**
- Top 5 Acc By Operator: **Horizontal bar chart**
- Top 5 Acc By Aircraft: **Horizontal bar chart**
- AVG Fatalities By Category: **Pie Chart**
- NrAcc By Season: **Pie Chart**

**Visualizing the following insights through the dashboard**

![png](Dashboard)

# Conclusions

The choropleth map on the left highlights the countries where the majority of accidents occur through darker color gradients. The United States accounts for the highest number of accidents. However, without data on the total number of flights, we cannot determine whether this reflects a higher risk associated with U.S. flights or simply a significantly higher number of flights in the United States compared to other countries. Nevertheless, this is still a relevant finding worth reporting to the AISS.

The same consideration applies to the number of accidents by airline operator and aircraft. We cannot determine whether the high number of accidents is due to lower safety standards or simply to a higher total number of flights.

The line chart at the top, showing the trend in accidents by year, indicates that the number of accidents has remained relatively stable over time, with the notable exception of the period around World War II.

Again, determining whether aviation safety has improved or deteriorated over the years would require data on the total number of flights. However, obtaining such data would be extremely difficult, if not impossible, especially considering the discontinuation of many former airlines whose historical records are no longer available.

In this case, although relying on external knowledge is generally a risky approach in data analysis, we can take into account a well-established fact that does not come directly from our dataset: the number of flights has increased significantly over the years. Since the number of accidents has remained relatively stable over time and has even declined since 2020, we can reasonably conclude that **aviation safety has improved significantly over the years**.

We also investigated whether there was a particularly dangerous season. However, the pie chart shows that accidents are relatively evenly distributed across the four seasons.

Although H1 accidents (Hijackings) occur much less frequently than other accident categories, they have the highest average number of fatalities, with as many as **86 fatalities per hijacking on average**. This highlights the importance of continuously improving and never treating pre-flight security checks superficially, as an error at this stage could have devastating consequences and potentially result in a major tragedy.

This is further supported by filtering the dataset by the number of fatalities: the accident with the highest number of fatalities belongs precisely to category H1, namely the infamous **September 11 attacks**, with an exceptionally high confirmed death toll of **1,692 victims**.
