# Credit_card_transactions
# This project shows the spending habits of people using creditcard in India

-- the dataset has been downloaded from
https://www.kaggle.com/datasets/thedevastator/analyzing-credit-card-spending-habits-in-india

select * from credit_trnx

---1.top 5 cities with max spend
select top 5 city, sum(amount) as total_spend
from credit_trnx
group by city
order by total_spend desc

-- or
WITH cte1 AS (
  SELECT city, SUM(CAST(amount AS BIGINT)) AS total_spend
  FROM credit_trnx
  GROUP BY city
),
total_spend AS (
  SELECT SUM(CAST(amount AS BIGINT)) AS total_amount
  FROM credit_trnx
)

SELECT TOP 5 cte1.*, total_spend.total_amount, total_spend*1.0/total_amount * 100 as perc_contribution
FROM cte1, total_spend
ORDER BY cte1.total_spend DESC;

------------------------------------
--2. highest spend month and amount spent in that month for each card type
WITH cte AS (
  SELECT 
    card_type,
    DATEPART(YEAR, transaction_date) AS yt,
    DATEPART(MONTH, transaction_date) AS mt,
    SUM(amount) AS total_spend
  FROM credit_trnx
  GROUP BY 
    card_type, 
    DATEPART(YEAR, transaction_date), 
    DATEPART(MONTH, transaction_date)
)
SELECT * FROM (
  SELECT *,RANK() OVER (PARTITION BY card_type ORDER BY total_spend DESC) AS rn
  FROM cte) AS ranked
WHERE rn = 1;

--3- write a query to print the transaction details(all columns from the table) for each card type when
--it reaches a cumulative of  1,000,000 total spends(We should have 4 rows in the o/p one for each card type)

with cte as (
select *, sum(amount) over(partition by card_type order by transaction_date, transaction_id) as total_spend
from credit_trnx
)
select *  from (select *, rank() over(partition by card_type order by total_spend) as rn
from cte where total_spend >=1000000)a where rn = 1

--4- write a query to find city which had lowest percentage spend for gold card type
with cte as (
select city, card_type, sum(amount) as amount,
sum(case when card_type = 'Gold' then amount end) as gold_amt
from credit_trnx
group by city, card_type)

select top 1 city, sum(gold_amt)*1.0/sum(amount) as gold_ratio
from cte
group by city
having sum(gold_amt) is not null
order by gold_ratio 

--write a query to print 3 coloums: city, hihest_expense_type, lowest_expense_type (ex: delhi, bills, fuel)
with cte as(
select city,exp_type,sum(amount)as aount from credit_trnx
group by city,exp_type)

select city, max(case when rnk_asc = 1 then exp_type end) as lowest_exp,
max(case when rnk_desc = 1 then exp_type end) as highest_exp
from
(select *,
RANK() over (partition by city order by exp_type desc) as rnk_desc 
,RANK() over (partition by city order by exp_type asc) as rnk_asc
from cte) a
group by city

--6- write a query to find percentage contribution of spends by females for each expense type

select exp_type,sum(case when gender ='F' then amount else 0 end)*1.0 /sum(amount) as f_percent_contri 
from credit_trnx
group by exp_type
order by f_percent_contri desc

--7- which card and expense type combination saw highest month over month growth in Jan-2014

with cte as (
SELECT card_type,exp_type,DATEPART(YEAR, transaction_date) AS yt,DATEPART(MONTH, transaction_date) AS mt,
SUM(amount) AS total_spend
FROM credit_trnx
GROUP BY card_type,exp_type,DATEPART(YEAR, transaction_date),DATEPART(MONTH, transaction_date)
)

select top 1*, total_spend-prv_month_spend as mom_growth
from
(select *
,lag(total_spend,1) over(partition by card_type, exp_type order by yt,mt) as prv_month_spend
from cte) a

where prv_month_spend is not null and yt=2014 and mt=1
order by mom_growth desc

--8- during weekends which city has highest total spend to total no of transcations ratio 
select city, sum(amount)*1.0/count(amount) as ratio
from credit_trnx
where DATEPART(weekday, transaction_date) in (1,7)
group by city
order by ratio desc

--9- which city took least number of days to reach its
--500th transaction after the first transaction in that city

with cte as (
select *, 
row_number() over(partition by city order by transaction_date, transaction_id) as rn -- we use row num coz it doesnt repeats
from credit_trnx
)
select top 1 city,datediff(day,min(transaction_date),max(transaction_date)) as datefiff1
from cte 
where rn = 1 or rn = 500
group by city
having count(*) = 2
order by datefiff1
