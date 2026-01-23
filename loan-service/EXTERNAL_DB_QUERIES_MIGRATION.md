# External Database Queries - Migration Documentation

This document lists all database queries made to external databases (databases other than `loan_service`'s own database) that need to be converted to API calls during the migration from easy server to respo server.

---

## 1. Payment Service Database (`payment_service`)

**Database Type:** MySQL  
**Connection Method:** Direct MySQL connection using `mysql.createConnection()` or `db.sequelize.query()`

### 1.1 Queries in `src/controllers/loansPaymentSettlementController.js`

**File:** `src/controllers/loansPaymentSettlementController.js`  
**Function:** `loanSettlement`  
**Line:** 21-30

```javascript
const connection = mysql.createConnection({
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  port: Number(process.env.DB_PORT),
  database: 'payment_service'
})
const [data] = await connectionPromise.query(
  `SELECT * FROM payment_service.razor_pay_payments
   WHERE customer_id = ?`, 
  [customerId]
)
```

**Purpose:** Fetch Razorpay payment details for loan settlement  
**Migration:** Convert to API call to payment-service: `GET /payment-service/api/v2/payments/razorpay?customerId={customerId}`

---

### 1.2 Queries in `src/services/loanAgreement.service.js`

**File:** `src/services/loanAgreement.service.js`  
**Function:** `getCustomerbankDetails`  
**Line:** 720

```javascript
const [data] = await db.sequelize.query(
  'SELECT * FROM payment_service.autopay_subscriptions where customer_id=:customerId AND subscription_status in (:statuses)',
  {
    replacements: { customerId, statuses: VALID_SUBSCRIPTION_STATUSES },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Get customer bank details from autopay subscriptions  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions?customerId={customerId}&status={statuses}`

---

### 1.3 Queries in `src/services/customerCreditProfile.service.js`

**File:** `src/services/customerCreditProfile.service.js`  
**Function:** (around line 479)

```javascript
const [result] = await db.sequelize.query(
  'SELECT COUNT(*) as count FROM payment_service.autopay_subscriptions where customer_id=:customerId AND bank_account_number=:bankAccountNumber AND checkout_status in (:statuses)',
  {
    replacements: { customerId, bankAccountNumber: bankDetails.bankAccountNo, statuses: VALID_DUPLICATE_BANK_ACCOUNT_CHECK_STATUSES },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Check for duplicate bank account in autopay subscriptions  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions/count?customerId={customerId}&bankAccountNumber={bankAccountNumber}&status={statuses}`

---

### 1.4 Queries in `src/services/cashTransfter.service.js`

**File:** `src/services/cashTransfter.service.js`  
**Function:** `getBankDetailsForEnachSuccess`  
**Line:** 1157

```javascript
const [data] = await db.sequelize.query(
  'SELECT * FROM payment_service.autopay_subscriptions where customer_id=:customerId AND subscription_status in (:statuses) order by created_date desc limit 1',
  {
    replacements: { customerId, statuses: VALID_SUBSCRIPTION_STATUSES },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Get latest autopay subscription for eNACH success  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions/latest?customerId={customerId}&status={statuses}`

---

### 1.5 Queries in `src/utils/getMandatePaymentMethodsAvailability.js`

**File:** `src/utils/getMandatePaymentMethodsAvailability.js`  
**Function:** (around line 70)

```javascript
db.sequelize.query(
  'SELECT count(*) as failedCounts FROM payment_service.autopay_subscriptions where customer_id=:customerId AND checkout_status in (:status) AND payment_gateway_provider =:PgProvider AND bank_account_number=:bankAccountNumber',
  {
    replacements: {
      customerId,
      status: ResponseStatus.FAILED,
      PgProvider: MandateProviders.CASHFREE,
      bankAccountNumber
    },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Count failed autopay subscriptions for mandate payment methods availability  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions/failed-count?customerId={customerId}&status={status}&pgProvider={PgProvider}&bankAccountNumber={bankAccountNumber}`

---

### 1.6 Queries in `src/controllers/getCustomerBankDetails.js`

**File:** `src/controllers/getCustomerBankDetails.js`  
**Function:** `getCustomerBankDetails`  
**Line:** 36

```javascript
result = await db.sequelize.query(
  'SELECT * FROM payment_service.autopay_subscriptions WHERE customer_id = :customerId AND subscription_status IN (:statuses) ORDER BY created_date DESC LIMIT 1',
  {
    replacements: { customerId, statuses: VALID_SUBSCRIPTION_STATUSES },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Get customer bank details from autopay subscriptions  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions?customerId={customerId}&status={statuses}`

---

### 1.7 Queries in `src/services/statementSummary.service.js`

**File:** `src/services/statementSummary.service.js`  
**Function:** `getBankDetailsForEnachSuccess`  
**Line:** 296

```javascript
const [data] = await db.sequelize.query(
  'SELECT * FROM payment_service.autopay_subscriptions where customer_id=:customerId AND subscription_status in (:statuses) order by created_date desc limit 1', 
  { 
    replacements: { customerId, statuses: VALID_SUBSCRIPTION_STATUSES }, 
    type: QueryTypes.SELECT 
  }
)
```

**Purpose:** Get latest autopay subscription for statement summary  
**Migration:** Convert to API call: `GET /payment-service/api/v2/autopay/subscriptions/latest?customerId={customerId}&status={statuses}`

---

## 2. File Upload Service Database (`fileupload_service`)

**Database Type:** MySQL  
**Connection Method:** `db.sequelize.query()` or `replicaDb.replicaDbSequelize.query()`

### 2.1 Queries in `src/services/database.service.js`

**File:** `src/services/database.service.js`  
**Function:** `getStatementData`  
**Line:** 180-194

```javascript
const [resultSet] = await replicaDb.replicaDbSequelize.query(
  `
    SELECT uploaded_file_name
    FROM fileupload_service.upload_history
    WHERE customer_id = :customerId AND upload_type = 'statementGeneration'
      AND YEAR(created_date) = :year
      AND MONTH(created_date) = :month
    ORDER BY created_date DESC
    LIMIT 1
  `,
  {
    replacements: { customerId, year, month },
    type: replicaDb.replicaDbSequelize.QueryTypes.SELECT
  }
)
```

**Purpose:** Get uploaded statement file name for a customer by year and month  
**Migration:** Convert to API call: `GET /fileupload-service/api/v1/upload-history?customerId={customerId}&uploadType=statementGeneration&year={year}&month={month}`

---

### 2.2 Queries in `src/services/statementSummary.service.js`

**File:** `src/services/statementSummary.service.js`  
**Function:** `getStatement`  
**Line:** 270-279

```javascript
const s3Path = await db.sequelize.query(
  `SELECT uploaded_file_name
   FROM fileupload_service.upload_history
   WHERE customer_id = :customerId AND upload_type = 'statementGeneration'
     AND YEAR(created_date) = :year
     AND MONTH(created_date) = :month
   ORDER BY created_date DESC
   LIMIT 1`,
  {
    replacements: { customerId, year, month },
    type: QueryTypes.SELECT
  }
)
```

**Purpose:** Get statement file path from upload history  
**Migration:** Convert to API call: `GET /fileupload-service/api/v1/upload-history?customerId={customerId}&uploadType=statementGeneration&year={year}&month={month}`

---

## 3. Bureau Service Database (`bureau_service`)

**Database Type:** MySQL  
**Connection Method:** NBFC MySQL connection pool

### 3.1 Queries in `src/services/database.service.js`

**File:** `src/services/database.service.js`  
**Function:** `getHardPullBureauFileName`  
**Line:** 510-524

```javascript
const connectionPool = getMySQLConnectionPool('respo');
const [rows] = await connectionPool.promise().query(
  `SELECT provider_name, hard_pull_response 
   FROM bureau_service.hard_pull_record 
   WHERE customer_id = ?`,
  [customerId]
);
```

**Purpose:** Get hard pull bureau record (provider name and response) for a customer  
**Migration:** Convert to API call: `GET /bureau-service/api/v1/hard-pull-records?customerId={customerId}`

---

## 4. MongoDB - Attribution Database (`attribution_db`)

**Database Type:** MongoDB  
**Connection Method:** MongoDB client via `getDbConnection().getPartnerCollection()`

### 4.1 Queries in `src/services/voltMoney.service.js`

**File:** `src/services/voltMoney.service.js`  
**Function:** `bulkUpdatePartnerProduct`  
**Line:** 865

```javascript
const data = await getDbConnection().getPartnerCollection().find().toArray()
```

**Purpose:** Fetch all partner product records from MongoDB for bulk update  
**Migration:** Convert to API call: `GET /attribution-service/api/v1/partner-products` (or appropriate service endpoint)

**Note:** This queries the `PARTNER_COLLECTION` collection in the `ATTRIBUTION_DB` database.

---

## 5. NBFC MySQL Databases (External LMS Databases)

**Database Type:** MySQL  
**Connection Method:** NBFC-specific MySQL connection pools via `getNBFCConfig(nbfcId).mysqlPool` or `getNBFCConfig(nbfcId).replicaMysqlPool`

**Note:** These databases use the same database name (`LOAN_SRV_DB_NAME`) but connect to different hosts based on NBFC configuration. They are considered external because they are accessed via different connection pools with NBFC-specific hosts.

### 5.1 Queries in `src/services/database.service.js`

#### 5.1.1 `getAllLoansData` (Line 61-91)

```javascript
const connectionPool = getMySQLConnectionPool(nbfcId)
const [loanData] = await connectionPool.promise().query(
  `SELECT id AS lmsLoanId, 
   (CASE  
     WHEN loan_status_id = 300 THEN "ACTIVE" 
     WHEN loan_status_id = 600 THEN "CLOSED_OBLIGATION_MET" 
     WHEN loan_status_id = 601 THEN "CLOSED_WRITTEN_OFF" 
     WHEN loan_status_id = 602 THEN "CLOSED_RESCHEDULED" 
     WHEN loan_status_id = 700 THEN "OVERPAID" 
   END) AS loanStatus
   FROM loan_service.m_loan
   WHERE client_id = ?
   AND loan_status_id IN (300, 600, 601, 602, 700) 
   AND disbursedon_date IS NOT NULL;`,
  [clientId]
);
```

**Purpose:** Get all loans data for a client  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans?clientId={clientId}&status={statuses}`

---

#### 5.1.2 `getInsuranceData` (Line 216-240)

```javascript
const connectionPool = getMySQLConnectionPool(nbfcId)
const [insuranceData] = await connectionPool.promise().query(
  `
    Select
    ML.id as loanId,
    ML.principal_amount as loanAmount,
    MLC.amount_paid_derived as insuranceAmount,
    MLC.created_date as insuranceDate
    from loan_service.m_loan ML
    left join loan_service.m_loan_charge MLC
    on ML.id = MLC.loan_id
    where ML.client_id = ?
    and MLC.charge_id in (39,44,45)
  `,
  [clientId]
)
```

**Purpose:** Get insurance data for loans  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans/insurance?clientId={clientId}`

---

#### 5.1.3 `getAllReceipts` (Line 322-345)

```javascript
const connectionPool = getMySQLConnectionPool(NBFCIdsMap.RESPO);
const [paymentDetails] = await connectionPool.promise().query(
  `
  SELECT receipt_number
  FROM m_payment_detail
  WHERE receipt_number IN (${transactionIds})
  `,
  transactionIds
);
```

**Purpose:** Get payment details by receipt numbers  
**Migration:** Convert to API call: `POST /lms-service/api/v1/payments/receipts` with body `{ receiptNumbers: [...] }`

---

#### 5.1.4 `getId` (Line 458-480)

```javascript
const connectionPool = getMySQLConnectionPool('respo');
const [rows] = await connectionPool
  .promise()
  .query(
    `
      Select
        id,
        receipt_number
      from
        m_payment_detail
      where
        receipt_number in (?)
    `,
    [paymentIds]
  );
```

**Purpose:** Get payment detail IDs by receipt numbers  
**Migration:** Convert to API call: `POST /lms-service/api/v1/payments/details` with body `{ receiptNumbers: [...] }`

---

#### 5.1.5 `getApportionDetails` (Line 482-508)

```javascript
const connectionPool = getMySQLConnectionPool('respo');
const [rows] = await connectionPool.promise().query(
  `
    Select
      payment_detail_id,
      loan_id,
      transaction_date,
      amount,
      principal_portion_derived,
      interest_portion_derived,
      fee_charges_portion_derived,
      penalty_charges_portion_derived,
      outstanding_loan_balance_derived
    from
      m_loan_transaction 
    where payment_detail_id in (?)
  `,
  [ids]
);
```

**Purpose:** Get loan transaction apportion details by payment detail IDs  
**Migration:** Convert to API call: `POST /lms-service/api/v1/loans/transactions/apportion` with body `{ paymentDetailIds: [...] }`

---

### 5.2 Queries in `src/services/finFlux.service.js`

**Note:** This file contains multiple queries to NBFC LMS databases. All use `getMySQLConnectionPool(nbfcId)` or `getReplicaMySQLConnectionPool(nbfcId)`.

#### 5.2.1 `getLoanDetails` (Line 1518-1547)

```javascript
const connectionPool = getMySQLConnectionPool(nbfcId)
const [loans] = await connectionPool.promise().query(
  `
    SELECT
    id AS loanId,
    approved_principal AS loanAmount,
    annual_nominal_interest_rate AS interestRate,
    number_of_repayments AS loanTenure,
    disbursedon_date AS disburseDate,
    approvedon_date AS approvedOnDate,
    expected_maturedon_date AS maturityDate,
    closedon_date AS closedOnDate,
    product_id AS productId,
    total_charges_due_at_disbursement_derived AS initialCharges,
    principal_net_disbursed_derived  AS disbursementAmount,
    total_excess_amount_derived AS excessAmount,
    loan_status_id AS loanStatusId,
    (CASE WHEN closedon_date < DATE(NOW())  THEN true ELSE false END) AS closed,
    (CASE WHEN closedon_date < DATE(NOW())  THEN maturedon_date ELSE null END) AS maturedOnDate
    FROM loan_service.m_loan
    where client_id=? and loan_status_id !='500'
  `,
  [clientId]
)
```

**Purpose:** Get detailed loan information  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans/details?clientId={clientId}`

---

#### 5.2.2 `getAllLoans` (Line 1549-1604)

```javascript
const connectionPool = getMySQLConnectionPool(nbfcId)
const [loans] = await connectionPool.promise().query(query, queryParams)
// Query selects from loan_service.m_loan with various filters
```

**Purpose:** Get all loans with status filters  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans?clientId={clientId}&status={statuses}`

---

#### 5.2.3 `getLoanInstallements` (Line 1643-1663)

```javascript
const [installements] = await connectionPool.promise().query(
  `
    SELECT 
    installment AS installmentNo,
    dueDate AS lmsDueDate,
    loan_id AS loanId,
    principal_amount AS principalAmount,
    interest_amount AS interestAmount,
    principal_completed_derived AS principalPaid,
    interest_completed_derived AS interestPaid,
    fee_charges_amount  AS feeChargesAmount,
    fee_charges_completed_derived AS feeChargesCompletedDerived
    FROM loan_service.m_loan_repayment_schedule
    WHERE loan_id IN (?)
  `,
  [loanIdList]
)
```

**Purpose:** Get loan installments/repayment schedule  
**Migration:** Convert to API call: `POST /lms-service/api/v1/loans/repayment-schedule` with body `{ loanIds: [...] }`

---

#### 5.2.4 `getLoanCharges` (Line 1665-1682)

```javascript
const [loanCharges] = await connectionPool.promise().query(
  `SELECT
  loan_id AS loanId,
  version AS version,
  charge_id AS chargeId,
  calculation_percentage AS chargePercentage,
  amount AS totalCharge,
  amount_outstanding_derived AS amountOutstanding,
  tax_amount AS gst,
  amount_sans_tax AS charge
  FROM loan_service.m_loan_charge
  WHERE loan_id IN (?) and waived=0`,
  [loanIdList]
)
```

**Purpose:** Get loan charges  
**Migration:** Convert to API call: `POST /lms-service/api/v1/loans/charges` with body `{ loanIds: [...] }`

---

#### 5.2.5 Additional Queries in `finFlux.service.js`

Multiple other queries exist in this file (lines 1785, 1810, 1857, 1870, 1884, 1910, 1962, 2053, 2082, 2332, 2358) that query:
- `loan_service.m_loan` - Various loan queries
- `loan_service.m_loan_repayment_schedule` - Repayment schedule queries
- `loan_service.m_payment_detail` - Payment detail queries

**Migration:** All should be converted to appropriate LMS service API endpoints.

---

### 5.3 Queries in `src/services/settlementReport.service.js`

#### 5.3.1 `getLoanDetails` (Line 120-135)

```javascript
const connectionPool = getNBFCConfig(nbfcId).mysqlPool
const [loans] = await connectionPool.promise().query(`
  SELECT
  id AS loanId
    FROM loan_service.m_loan
    where client_id=?
  `, [clientId])
```

**Purpose:** Get loan IDs for a client  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans?clientId={clientId}`

---

#### 5.3.2 `getTotalOutstandingWithoutAdvancedAmount` (Line 137-159)

```javascript
const [feeChargesAccruable] = await connectionPool.promise().query(`
  SELECT
  principal_amount AS principalAmount,
  interest_amount AS interestAmount,
  principal_completed_derived AS principalPaid,
  interest_completed_derived AS interestPaid
  FROM loan_service.m_loan_repayment_schedule
  WHERE loan_id IN (?) and emi_cleared_on is null
`, [loanIdList])
```

**Purpose:** Get outstanding amounts from repayment schedule  
**Migration:** Convert to API call: `POST /lms-service/api/v1/loans/outstanding` with body `{ loanIds: [...] }`

---

#### 5.3.3 `getPenaltyAmount` (Line 161-172)

```javascript
const [penalties] = await connectionPool.promise().query(`
  SELECT
  penalty_charges_outstanding_derived AS penalty
    FROM loan_service.m_loan
    where loan_status_id = 300 and client_id = ?
  `, [lmsClientId])
```

**Purpose:** Get penalty amount for active loans  
**Migration:** Convert to API call: `GET /lms-service/api/v1/loans/penalties?clientId={lmsClientId}&status=300`

---

## Summary

### External Databases Identified:

1. **payment_service** (MySQL) - 7 query locations
2. **fileupload_service** (MySQL) - 2 query locations
3. **bureau_service** (MySQL) - 1 query location
4. **attribution_db** (MongoDB) - 1 query location
5. **NBFC LMS Databases** (MySQL) - ~20+ query locations across multiple files

### Migration Strategy:

1. **Payment Service:** Create/use payment-service API endpoints for autopay subscriptions and razorpay payments
2. **File Upload Service:** Create/use fileupload-service API endpoints for upload history
3. **Bureau Service:** Create/use bureau-service API endpoints for hard pull records
4. **Attribution Service:** Create/use attribution-service API endpoints for partner products
5. **LMS Service:** Create/use LMS service API endpoints for all loan-related queries (m_loan, m_loan_repayment_schedule, m_loan_charge, m_payment_detail, m_loan_transaction)

### Files Requiring Changes:

1. `src/controllers/loansPaymentSettlementController.js`
2. `src/services/loanAgreement.service.js`
3. `src/services/customerCreditProfile.service.js`
4. `src/services/cashTransfter.service.js`
5. `src/utils/getMandatePaymentMethodsAvailability.js`
6. `src/controllers/getCustomerBankDetails.js`
7. `src/services/statementSummary.service.js`
8. `src/services/database.service.js`
9. `src/services/finFlux.service.js`
10. `src/services/settlementReport.service.js`
11. `src/services/voltMoney.service.js`

### Next Steps:

1. Identify or create API endpoints in each external service
2. Replace direct database queries with HTTP API calls
3. Update error handling to handle API errors
4. Consider caching strategies for frequently accessed data
5. Update tests to mock API calls instead of database connections

