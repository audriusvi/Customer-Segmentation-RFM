# Customer Segmentation & RFM
## The aim of the project
To perform RFM analysis with a given data set from "turing_data_analytics" database and present analyses with a dashboard.
## Tools used
Big Query, Tableau, Google Spreadsheets.
## Data Processing
SQL code snippet to get summarized customers metrics by segment.

``` sql
-- Data is taken from Turing Data Analytics database, rfm table.
-- Here, I perform RFM (recency, frequency, monetary) analyses and segment customers accordingly.

WITH
  rfm_values AS (
  SELECT
    CustomerID AS customer_id,
-- Recency (r) – period since last purchase, frequency (f) – number of transactions, monetary (m) – total money spent.
    DATE_DIFF('2011-12-01', DATE(MAX(InvoiceDate)), DAY) AS recency,
    COUNT(DISTINCT InvoiceNo) AS frequency,
    SUM(Quantity * UnitPrice) AS monetary
  FROM
    tc-da-1.turing_data_analytics.rfm
  WHERE
    (DATE(InvoiceDate) BETWEEN '2010-12-01' AND '2011-12-01') AND
    CustomerID IS NOT NULL AND
    Quantity > 0 AND
    UnitPrice > 0
  GROUP BY
    CustomerID),

  rmf_metric AS ( 
  SELECT
  *,
-- Each customer is assigned ranks (RANK()) based on recency, frequency, and monetary value.
-- Frequency and monetary ranks are in ascending order, while recency - descending (more recent -> higher rank).
-- These ranks are divided by the total number of rows and multipled by 4, resulting in new ranks between 0 and 4.
-- After rounding up and converting to strings, the ranks range from 1 to 4,
-- ensuring that each rank has a similar number of members and identical values receive the same rank.
-- The new ranks are merged to create the RFM metric.
    CAST(CEILING((RANK() OVER (ORDER BY recency DESC)) / (COUNT(1) OVER (PARTITION BY 1)) * 4) AS STRING) ||
    CAST(CEILING((RANK() OVER (ORDER BY frequency)) / (COUNT(1) OVER (PARTITION BY 1)) * 4) AS STRING) ||
    CAST(CEILING((RANK() OVER (ORDER BY monetary)) / (COUNT(1) OVER (PARTITION BY 1)) * 4) AS STRING) AS rfm_score
  FROM
    rfm_values),

  combinations AS (
  SELECT
    CAST(num AS STRING) AS comb
  FROM
    UNNEST(GENERATE_ARRAY(111, 444)) AS num
  WHERE
    REGEXP_CONTAINS(CAST(num AS STRING), r'^[1234]+$')),

  segmentation AS (
  SELECT
    comb AS rfm_score,
    CASE
      WHEN comb IN ('444', '434') THEN 'Stars'                    
      WHEN comb IN ('344', '334') THEN 'Loyal Spenders'              
      WHEN comb IN ('413', '414', '423', '424', '433') THEN 'Rising Stars'
      WHEN comb IN ('411', '412') THEN 'New'
      WHEN comb IN ('421', '422', '431', '432', '441', '442') THEN 'First Gear'
      WHEN comb IN ('133', '143', '214', '224', '234','243', '244') THEN 'At-Risk'
      WHEN comb IN ('314', '324', '343', '443') THEN 'Potential Loyalists'
      WHEN comb IN ('134', '144') THEN "Can't Lose Them"
      WHEN comb IN ('131', '132', '141','142', '231', '232', '241', '242', '321', '331', '332', '341','342') THEN 'Sleeping Giants'
      WHEN comb IN ('113', '114', '123', '124', '213') THEN 'Ghosts'
      WHEN comb IN ('112', '121', '122', '211', '311') THEN 'Hipernating'
      WHEN comb IN ('212', '221', '222', '223','233','312', '313', '322', '323', '333') THEN 'Ordinary'
      WHEN comb = '111' THEN 'Lost'  END AS segment
  FROM
    combinations)

SELECT
  segment,
  COUNT(1) AS customers,
  ROUND(SUM(monetary),0) AS total_value,
  ROUND(AVG(monetary),0) AS avg_value,
  ROUND(AVG(recency),0) AS avg_recency_days,
  ROUND(AVG(frequency),1) AS avg_frequency
FROM
  rmf_metric
LEFT JOIN
  segmentation
USING
  (rfm_score)
GROUP BY
  segment
ORDER BY
  customers DESC;
```
