# sql_sample
Data Analyst Assessment By AJIBOLA OLAITAN DAMMY 

select * from adashi_staging.plans_plan;
select * from adashi_staging.savings_savingsaccount;
select * from adashi_staging.users_customuser;
select * from adashi_staging.withdrawals_withdrawal;

-- Question 1 
SELECT
  u.id                     AS owner_id,
  u.name                   AS name,
  s.savings_count          AS savings_count,
  i.investment_count       AS investment_count,
  ROUND(COALESCE(t.total_deposits, 0) / 100.0, 2) AS total_deposits
FROM users_customuser u
  -- count of savings plans per customer
  JOIN (
    SELECT owner_id, COUNT(*) AS savings_count
    FROM plans_plan
    WHERE is_regular_savings = 1
    GROUP BY owner_id
  ) s ON u.id = s.owner_id
  -- count of investment plans per customer
  JOIN (
    SELECT owner_id, COUNT(*) AS investment_count
    FROM plans_plan
    WHERE is_a_fund = 1
    GROUP BY owner_id
  ) i ON u.id = i.owner_id
  -- sum of all deposit transactions per customer
  LEFT JOIN (
    SELECT owner_id, SUM(confirmed_amount) AS total_deposits
    FROM savings_savingsaccount
    GROUP BY owner_id
  ) t ON u.id = t.owner_id
ORDER BY total_deposits DESC;

-- Question 2 

WITH customer_txn AS (
  SELECT
    owner_id,
    COUNT(*) AS txn_count
  FROM savings_savingsaccount
  WHERE transaction_date >= DATE_SUB(CURDATE(), INTERVAL 12 MONTH)
  GROUP BY owner_id
),
customer_freq AS (
  SELECT
    owner_id,
    txn_count / 12.0 AS avg_txn_per_month,
    CASE
      WHEN txn_count >= 120 THEN 'High Frequency'
      WHEN txn_count BETWEEN 36 AND 119 THEN 'Medium Frequency'
      ELSE 'Low Frequency'
    END AS frequency_category
  FROM customer_txn
)
SELECT
  frequency_category,
  COUNT(*) AS customer_count,
  ROUND(AVG(avg_txn_per_month), 1) AS avg_transactions_per_month
FROM customer_freq
GROUP BY frequency_category
ORDER BY
  CASE frequency_category
    WHEN 'High Frequency' THEN 1
    WHEN 'Medium Frequency' THEN 2
    WHEN 'Low Frequency' THEN 3
  END;
  
  -- Question 3 
  
 SELECT
  p.id AS plan_id,
  p.owner_id,
  CASE
    WHEN p.is_regular_savings = 1 THEN 'Savings'
    ELSE 'Investment'
  END AS type,
  DATE_FORMAT(MAX(s.transaction_date), '%Y-%m-%d') AS last_transaction_date,
  DATEDIFF(CURDATE(), MAX(s.transaction_date)) AS inactivity_days
FROM plans_plan p
LEFT JOIN savings_savingsaccount s
  ON s.plan_id = p.id
-- WHERE p.active = 1   <-- remove or comment out this line
GROUP BY p.id, p.owner_id, p.is_regular_savings
HAVING MAX(s.transaction_date) < DATE_SUB(CURDATE(), INTERVAL 365 DAY);


  -- Question 4
  
SELECT
  u.id AS customer_id,
  u.name AS name,
  GREATEST(
    (YEAR(CURDATE()) * 12 + MONTH(CURDATE())) -
    (YEAR(u.date_joined) * 12 + MONTH(u.date_joined)),
    1
  ) AS tenure_months,
  COUNT(s.id) AS total_transactions,
  ROUND(
    (COUNT(s.id) / 
      GREATEST(
        (YEAR(CURDATE()) * 12 + MONTH(CURDATE())) -
        (YEAR(u.date_joined) * 12 + MONTH(u.date_joined)),
        1
      )
    ) * 12 * (0.001 * AVG(s.confirmed_amount / 100.0)),
    2
  ) AS estimated_clv
FROM users_customuser u
JOIN savings_savingsaccount s ON s.owner_id = u.id
GROUP BY u.id, u.name, u.date_joined
ORDER BY estimated_clv DESC;
