# External Database Queries - Migration Documentation

This document lists all database listing queries used in the batch service that query external databases (databases other than the batch service's own database) for migration from **easy** server to **respo** server.

---

## 1. PAYMENT_SERVICE Database Queries

### 1.1 `payment_service.loan_payments` Table

#### Query 1: Get Filtered Initiated Payments

**Location:** `jobs/handlers/reconciliationHandler.js` (line 8-14)  
**Function:** `getFilteredInitiatedPayments()`  

**Query (Easy Source):**
```sql
SELECT lp.payment_id
FROM payment_service.loan_payments lp
JOIN loan_service.customer_nbfc_id cni
  ON lp.customer_id = cni.customer_id
WHERE cni.nbfc_id = 'respo'
  AND lp.settlement_status = 'INITIATED'
  AND lp.created_date BETWEEN (NOW() - INTERVAL 10 DAY) AND (NOW() - INTERVAL 90 MINUTE);
```

**Query (Respo Source - for comparison):**
```sql
SELECT receipt_number
FROM m_payment_detail
WHERE receipt_number IN (:paymentIdList);
```

**Purpose:** Get payment IDs from easy source that need to be migrated to respo, filtering out payments that already exist in respo  
**Used In:** Payment settlement reconciliation flow  
**Migration:** Create API endpoint in payment-service: `GET /api/v1/payments/initiated?nbfcId=respo&startDate=:startDate&endDate=:endDate`  
**Note:** This query compares data between easy and respo sources to identify missing payments

---

#### Query 2: Get Payment Details for Settlement

**Location:** `jobs/handlers/reconciliationHandler.js` (line 43-50)  
**Function:** `getPaymentDetailsForSettlement(paymentIds)`  

**Query:**
```sql
SELECT
  payment_id AS paymentId,
  amount_paid AS amount,
  customer_id AS customerId,
  payment_date AS transactionTime
FROM loan_payments
WHERE payment_id IN (:formattedIds);
```

**Purpose:** Fetch detailed payment information for settlement processing  
**Used In:** Payment settlement reconciliation flow  
**Migration:** Create API endpoint in payment-service: `POST /api/v1/payments/details` with body `{ paymentIds: [] }`  
**Note:** This query is called after Query 1 to get full payment details

---

#### Query 3: Get Loan Payment Record

**Location:** `services/db.js` (line 182-184)  
**Function:** `getLoanPaymentRecord(customerId, amount, paymentId)`  

**Query:**
```sql
SELECT 
  id, 
  customer_id AS customerId, 
  payment_id AS paymentId, 
  amount_paid AS amountPaid 
FROM payment_service.loan_payments
WHERE customer_id = :customerId 
  AND payment_id = :paymentId 
  AND settlement_status != 'COMPLETED' 
  AND status = 'SUCCESS' 
  AND amount_paid = :amount;
```

**Purpose:** Retrieve a specific loan payment record by customer ID, payment ID, and amount  
**Used In:** Payment validation and settlement  
**Migration:** Create API endpoint in payment-service: `GET /api/v1/payments/validate?customerId=:customerId&paymentId=:paymentId&amount=:amount`

---

### 1.2 `payment_service.autopay_subscriptions` Table

#### Query 4: Get Customers with Null LMS Client ID

**Location:** `services/db.js` (line 214-227)  
**Function:** `getCustomersWithNullLmsClientId()`  

**Query:**
```sql
SELECT 
  ccp.customer_id AS customerId, 
  aps.subscription_status, 
  ccp.is_active AS isActive, 
  ccp.lms_client_id, 
  aps.created_date
FROM payment_service.autopay_subscriptions AS aps
LEFT JOIN loan_service.customer_credit_profile AS ccp 
  ON aps.customer_id = ccp.customer_id
WHERE aps.subscription_status IN ('ACTIVE', 'BANK_APPROVAL_PENDING')
  AND ccp.lms_client_id IS NULL
  AND DATE(aps.created_date) >= CURDATE() - INTERVAL 2 DAY
ORDER BY aps.created_date DESC;
```

**Purpose:** Find customers with active autopay subscriptions but missing LMS client ID  
**Used In:** Autopay subscription reconciliation  
**Migration:** Create API endpoint in payment-service: `GET /api/v1/autopay-subscriptions/null-lms-client-id`  
**Note:** This query joins with loan_service, so may need coordination between services

---

## 2. LOAN_SERVICE Database Queries

### 2.1 `loan_service.customer_credit_profile` Table

#### Query 5: Get Customers to Generate Statements

**Location:** `services/db.js` (line 134-137)  
**Function:** `getCustomersToGenerateStatements(productId)`  

**Query:**
```sql
SELECT customer_id AS customerId 
FROM loan_service.customer_credit_profile 
WHERE loan_service.customer_credit_profile.product_id = :productId  
  AND max_cash_limit != 0 
  AND max_cash_limit - available_max_cash_limit > 0;
```

**Purpose:** List customers eligible for statement generation based on product ID and cash limit  
**Used In:** Statement generation batch job  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/customer-credit-profile/eligible-for-statements?productId=:productId`

---

#### Query 6: Fetch Customers to Reconcile Limit

**Location:** `services/db.js` (line 159-160)  
**Function:** `fetchCustomersToReconcileLimit()`  

**Query:**
```sql
SELECT customer_id AS customerId 
FROM loan_service.customer_credit_profile
WHERE max_cash_limit != 0 
  AND max_cash_limit - available_max_cash_limit > 0;
```

**Purpose:** Get customers whose cash limits need reconciliation  
**Used In:** Limit reconciliation batch job  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/customer-credit-profile/needs-limit-reconciliation`

---

#### Query 7: Get Customers to Generate E-NACH Order

**Location:** `services/db.js` (line 170-172)  
**Function:** `getCustomersToGenerateEnachOrder(productId)`  

**Query:**
```sql
SELECT customer_id AS customerId 
FROM loan_service.customer_credit_profile 
WHERE loan_service.customer_credit_profile.product_id = :productId  
  AND max_cash_limit != 0 
  AND max_cash_limit - available_max_cash_limit > 0;
```

**Purpose:** List customers eligible for E-NACH order generation  
**Used In:** E-NACH order generation batch job  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/customer-credit-profile/eligible-for-enach?productId=:productId`  
**Note:** Similar to Query 5, consider combining into a single endpoint with a parameter

---

#### Query 8: Get Credit Profile

**Location:** `services/migrationService.js` (line 64)  
**Function:** `validateCustomer(customerId)`  

**Query:**
```sql
SELECT * 
FROM loan_service.customer_credit_profile 
WHERE customer_id = :customerId;
```

**Purpose:** Retrieve customer credit profile information for migration validation  
**Used In:** Customer migration validation  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/customer-credit-profile/:customerId`  
**Note:** This may already exist, verify if it returns all required fields

---

### 2.2 `loan_service.bank_account_transfer` Table

#### Query 9: Fetch Loan IDs

**Location:** `services/db.js` (line 148-150)  
**Function:** `fetchLoanIds()`  

**Query:**
```sql
SELECT loan_id, transfer_status 
FROM loan_service.bank_account_transfer 
WHERE (transfer_status = 'pending' 
  OR transfer_status = 'accepted' 
  OR transfer_status = 'initialized' 
  OR transfer_status = 'settlementinpro') 
  AND purpose = 'loandisbursal';
```

**Purpose:** Get loan IDs with pending transfer status for loan disbursal  
**Used In:** Loan disbursal reconciliation  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/bank-account-transfers/pending-disbursal`

---

### 2.3 `loan_service.loan_product_details` Table

#### Query 10: Store Product Mapper

**Location:** `services/db.js` (line 204)  
**Function:** `storeProductMapper()`  

**Query:**
```sql
SELECT * 
FROM loan_service.loan_product_details;
```

**Purpose:** Cache loan product details for quick lookup  
**Used In:** Application startup/config loading  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/loan-product-details`  
**Note:** This is called during app initialization, consider caching the response

---

### 2.4 `loan_service.m_loan` Table

#### Query 11: Get Loan Details

**Location:** `services/migrationService.js` (line 66-68)  
**Function:** `validateCustomer(customerId)`  

**Query:**
```sql
SELECT * 
FROM loan_service.m_loan 
WHERE client_id = :clientId
  AND (loan_status_id = 200 OR loan_status_id = 300);
```

**Purpose:** Fetch loan details for customers with specific loan status (200 or 300)  
**Used In:** Customer migration validation  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/loans?clientId=:clientId&loanStatusId=200,300`

---

### 2.5 `loan_service.customer_nbfc_id` Table

#### Query 12: Join with Customer NBFC ID (Used in Query 1)

**Location:** `jobs/handlers/reconciliationHandler.js` (line 10-11)  
**Function:** `getFilteredInitiatedPayments()`  

**Query:**
```sql
JOIN loan_service.customer_nbfc_id cni
  ON lp.customer_id = cni.customer_id
WHERE cni.nbfc_id = 'respo'
```

**Purpose:** Filter payments by NBFC ID (used in conjunction with Query 1)  
**Used In:** Payment settlement reconciliation  
**Migration:** This is part of Query 1, should be handled by the payment-service API endpoint

---

## 3. PROFILE_SERVICE Database Queries

### 3.1 `profile_service.customer_details` Table

#### Query 13: Get Customer Details

**Location:** `services/db.js` (line 28-31)  
**Function:** `getCustomerDetails(customerId)`  

**Query:**
```sql
SELECT 
  first_name AS firstName, 
  last_name AS lastName, 
  mobile_number AS phone, 
  email
FROM profile_service.customer_details
WHERE customer_id = :customerId;
```

**Purpose:** Retrieve basic customer information by customer ID  
**Used In:** Data deletion and customer operations  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/customer-details/:customerId`  
**Note:** This may already exist, verify if it returns all required fields

---

#### Query 14: Fetch Customers Data from SQL

**Location:** `services/db.js` (line 432-436)  
**Function:** `fetchCustomersDataFromSQL()`  

**Query:**
```sql
SELECT 
  pan_number AS panNumber, 
  mobile_number AS phoneNumber,
  customer_id AS customerId, 
  app_version AS appVersion, 
  os_version AS osVersion, 
  uid AS deviceId
FROM profile_service.customer_details
WHERE created_date BETWEEN :formattedStartingDate AND :formattedEndingDate;
```

**Purpose:** Get customer registration data within a specific time range for attribution reconciliation  
**Used In:** Attribution data reconciliation  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/customer-details/by-date-range?startDate=:startDate&endDate=:endDate`

---

### 3.2 `profile_service.customer_data_deletion_request` Table

#### Query 15: Get Customers Having Revoke In Progress

**Location:** `services/db.js` (line 14-17)  
**Function:** `getCustomersHavingRevokeInProgress(daysDelay)`  

**Query:**
```sql
SELECT customer_id AS customerId
FROM profile_service.customer_data_deletion_request
WHERE DATE(created_at) = DATE_SUB(CURDATE(), INTERVAL :daysDelay DAY)
  AND status = "REVOKING_IN_PROGRESS";
```

**Purpose:** Find customers with data deletion requests in revoking status  
**Used In:** Data deletion batch job  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/customer-data-deletion-request/revoking-in-progress?daysDelay=:daysDelay`

---

#### Query 16: Get Customers Deletion Requests

**Location:** `services/db.js` (line 42-46)  
**Function:** `getCustomersDeletionRequests(daysDelay)`  

**Query:**
```sql
SELECT *
FROM profile_service.customer_data_deletion_request
WHERE DATE(created_date) <= DATE_SUB(CURDATE(), INTERVAL :daysDelay DAY)
  AND status = "PENDING" 
  AND gdpr_request_id IS NULL;
```

**Purpose:** List pending customer data deletion requests  
**Used In:** Data deletion batch job  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/customer-data-deletion-request/pending?daysDelay=:daysDelay`

---

#### Query 17: Fetch Eligible Customers for Data Deletion

**Location:** `services/db.js` (line 57-66)  
**Function:** `fetchEligibleCustomersForDataDeletion(daysDelay)`  

**Query:**
```sql
SELECT 
  c.customer_id AS customerId, 
  c.name, 
  c.email, 
  u.employment_type AS employmentType, 
  u.is_kyc_failed AS isKycFailed, 
  u.l1_status AS l1Status, 
  u.l2_status AS l2Status 
FROM profile_service.customer_data_deletion_request c
LEFT JOIN profile_service.user_states u
  ON c.customer_id = u.customer_id
WHERE DATE(c.created_date) = DATE_SUB(CURDATE(), INTERVAL :daysDelay DAY)
  AND status = 'PENDING'
  AND gdpr_request_id IS NOT NULL;
```

**Purpose:** Get customers eligible for GDPR data deletion  
**Used In:** Data deletion batch job  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/customer-data-deletion-request/eligible?daysDelay=:daysDelay`  
**Note:** This query joins with user_states table, ensure API returns all required fields

---

### 3.3 `profile_service.user_states` Table

#### Query 18: Get Customers to Generate Communication Events

**Location:** `services/db.js` (line 347-349)  
**Function:** `getCustomersToGenerateCommunicationEvents()`  

**Query:**
```sql
SELECT customer_id AS customerId 
FROM profile_service.user_states
WHERE is_home_screen_reached = 1;
```

**Purpose:** List customers who have reached the home screen  
**Used In:** Communication events batch job  
**Migration:** Create API endpoint in profile-service: `GET /api/v1/user-states/home-screen-reached`

---

## 4. MIGRATION SERVICE Queries

### 4.1 `profile_service.customer_details_rpn` Table

#### Query 19: Get Target Table Columns

**Location:** `services/migrationService.js` (line 17-23)  
**Function:** `getTargetTableColumns()`  

**Query (Respo Source):**
```sql
SELECT COLUMN_NAME 
FROM INFORMATION_SCHEMA.COLUMNS 
WHERE TABLE_SCHEMA = 'profile_service' 
  AND TABLE_NAME = 'customer_details_rpn'
ORDER BY ORDINAL_POSITION;
```

**Purpose:** Dynamically fetch column names from the target migration table  
**Used In:** Customer migration process  
**Migration:** This is a schema introspection query, may need to be handled differently in respo or create API endpoint: `GET /api/v1/migration/target-table-columns?tableName=customer_details_rpn`

---

#### Query 20: Build Customer Details Move Query

**Location:** `services/migrationService.js` (line 38-45)  
**Function:** `buildCustomerDetailsMoveQuery()`, `moveCustomerDetails(customerId)`  

**Query (Easy Source - SELECT):**
```sql
SELECT :columnList 
FROM profile_service.customer_details
WHERE customer_id = :customerId;
```

**Query (Respo Source - INSERT):**
```sql
INSERT INTO profile_service.customer_details_rpn (:columnList)
SELECT :columnList 
FROM profile_service.customer_details
WHERE customer_id = :customerId;
```

**Purpose:** Dynamically build INSERT query to move customer details from easy to respo  
**Used In:** Customer migration process  
**Migration:** Create API endpoint in profile-service: `POST /api/v1/migration/move-customer-details/:customerId`  
**Note:** This is a migration-specific operation, may need special handling

---

## Base URL Configuration for Migration

When migrating from direct database queries to API calls, you need to configure service-specific base URLs. Replace direct database access with HTTP API calls using the appropriate base URL for each service.

### Required Environment Variables

Add the following environment variables to your `.env` file or configuration:

#### Payment Service
```bash
PAYMENT_SERVICE_BASE_URL=https://payment-service.respo.example.com
# or for easy server
PAYMENT_SERVICE_BASE_URL=https://payment-service.easy.example.com
```

#### Loan Service
```bash
LOAN_SERVICE_BASE_URL=https://loan-service.respo.example.com
# or for easy server
LOAN_SERVICE_BASE_URL=https://loan-service.easy.example.com
```

#### Profile Service
```bash
PROFILE_SERVICE_BASE_URL=https://profile-service.respo.example.com
# or for easy server
PROFILE_SERVICE_BASE_URL=https://profile-service.easy.example.com
```

#### Other Services (if used)
```bash
# Risk Service
RISK_SERVICE_BASE_URL=https://risk-service.respo.example.com

# KYC Service
KYC_SERVICE_BASE_URL=https://kyc-service.respo.example.com

# Account Service
ACCOUNT_SERVICE_BASE_URL=https://account-service.respo.example.com

# Reward Service
REWARD_SERVICE_BASE_URL=https://reward-service.respo.example.com

# S3/File Upload Service
S3_SERVICE_URL=https://s3-service.respo.example.com

# Interest Benefit Service
INTEREST_BENEFIT_BASE_URL=https://interest-benefit-service.respo.example.com
```

### Migration Pattern

**Before (Direct Database Query):**
```javascript
const { executeQuery } = require('../../utils/dbUtil');

const query = `SELECT * FROM payment_service.loan_payments WHERE customer_id = :customerId`;
const results = await executeQuery(query, 'payment_service', 'easy');
```

**After (API Call):**
```javascript
const axios = require('axios');

const response = await axios.get(
  `${process.env.PAYMENT_SERVICE_BASE_URL}/api/v1/payments?customerId=${customerId}`
);
const results = response.data;
```

### Base URL Usage by Service

#### Payment Service Base URL
- **Environment Variable**: `PAYMENT_SERVICE_BASE_URL`
- **Used For**: Queries 1-4 (Payment and Autopay operations)
- **Example Endpoints**:
  - `/api/v1/payments/initiated`
  - `/api/v1/payments/details`
  - `/api/v1/payments/validate`
  - `/api/v1/autopay-subscriptions/null-lms-client-id`

#### Loan Service Base URL
- **Environment Variable**: `LOAN_SERVICE_BASE_URL`
- **Used For**: Queries 5-12 (Loan, Credit Profile, and Product operations)
- **Example Endpoints**:
  - `/api/v1/customer-credit-profile/eligible-for-statements`
  - `/api/v1/customer-credit-profile/needs-limit-reconciliation`
  - `/api/v1/customer-credit-profile/:customerId`
  - `/api/v1/bank-account-transfers/pending-disbursal`
  - `/api/v1/loan-product-details`
  - `/api/v1/loans`

#### Profile Service Base URL
- **Environment Variable**: `PROFILE_SERVICE_BASE_URL`
- **Used For**: Queries 13-18 (Customer Details, Data Deletion, User States)
- **Example Endpoints**:
  - `/api/v1/customer-details/:customerId`
  - `/api/v1/customer-details/by-date-range`
  - `/api/v1/customer-data-deletion-request/revoking-in-progress`
  - `/api/v1/customer-data-deletion-request/pending`
  - `/api/v1/customer-data-deletion-request/eligible`
  - `/api/v1/user-states/home-screen-reached`
  - `/api/v1/migration/move-customer-details/:customerId`

### Code Changes Required

1. **Replace `executeQuery` calls** with `axios` HTTP requests
2. **Update imports** from `utils/dbUtil` to `axios`
3. **Replace query parameters** with HTTP query parameters or request body
4. **Update error handling** to handle HTTP errors instead of database errors
5. **Add authentication headers** if required by the API endpoints

### Example Migration

**File**: `services/db.js`  
**Function**: `getLoanPaymentRecord()`

**Before:**
```javascript
const { getDbInstance } = require('../utils/dbUtil');

const getLoanPaymentRecord = async (customerId, amount, paymentId) => {
  const db = getDbInstance(process.env.DB_PAYMENT_SERVICE);
  const data = await db.query(`
    SELECT id, customer_id AS customerId, payment_id AS paymentId, amount_paid AS amountPaid 
    FROM payment_service.loan_payments
    WHERE customer_id = :customerId 
      AND payment_id = :paymentId 
      AND settlement_status != 'COMPLETED' 
      AND status = 'SUCCESS' 
      AND amount_paid = :amount
  `, {
    replacements: { customerId, amount, paymentId },
    type: db.QueryTypes.SELECT
  });
  return data.length > 0 ? data[0] : null;
};
```

**After:**
```javascript
const axios = require('axios');

const getLoanPaymentRecord = async (customerId, amount, paymentId) => {
  try {
    const response = await axios.get(
      `${process.env.PAYMENT_SERVICE_BASE_URL}/api/v1/payments/validate`,
      {
        params: {
          customerId,
          paymentId,
          amount
        }
      }
    );
    return response.data;
  } catch (error) {
    if (error.response && error.response.status === 404) {
      return null;
    }
    throw error;
  }
};
```

### Configuration Checklist

- [ ] Add `PAYMENT_SERVICE_BASE_URL` environment variable
- [ ] Add `LOAN_SERVICE_BASE_URL` environment variable
- [ ] Add `PROFILE_SERVICE_BASE_URL` environment variable
- [ ] Update all `executeQuery` calls to use API endpoints
- [ ] Update all `getDbInstance` calls to use API endpoints
- [ ] Replace database error handling with HTTP error handling
- [ ] Add authentication/authorization headers if required
- [ ] Update unit tests to mock API calls instead of database queries
- [ ] Test all endpoints in staging environment before production migration

---

## Summary

### Total Queries by Database:

- **payment_service**: 4 queries → Use `PAYMENT_SERVICE_BASE_URL`
- **loan_service**: 8 queries → Use `LOAN_SERVICE_BASE_URL`
- **profile_service**: 6 queries → Use `PROFILE_SERVICE_BASE_URL`
- **migration_service**: 2 queries → Use `PROFILE_SERVICE_BASE_URL`
- **Total**: 20 queries

### Base URL Environment Variables:

| Service | Environment Variable | Queries |
|---------|---------------------|---------|
| Payment Service | `PAYMENT_SERVICE_BASE_URL` | 1-4 |
| Loan Service | `LOAN_SERVICE_BASE_URL` | 5-12 |
| Profile Service | `PROFILE_SERVICE_BASE_URL` | 13-18, 19-20 |

### Recommended API Endpoints to Create:

#### Payment Service APIs:

1. `GET /api/v1/payments/initiated` - Get initiated payments by NBFC ID and date range
2. `POST /api/v1/payments/details` - Get payment details by payment IDs
3. `GET /api/v1/payments/validate` - Validate payment record
4. `GET /api/v1/autopay-subscriptions/null-lms-client-id` - Get autopay subscriptions with null LMS client ID

#### Loan Service APIs:

1. `GET /api/v1/customer-credit-profile/eligible-for-statements` - Get customers eligible for statements
2. `GET /api/v1/customer-credit-profile/needs-limit-reconciliation` - Get customers needing limit reconciliation
3. `GET /api/v1/customer-credit-profile/eligible-for-enach` - Get customers eligible for E-NACH
4. `GET /api/v1/customer-credit-profile/:customerId` - Get customer credit profile
5. `GET /api/v1/bank-account-transfers/pending-disbursal` - Get pending loan disbursals
6. `GET /api/v1/loan-product-details` - Get all loan product details
7. `GET /api/v1/loans` - Get loans by client ID and status

#### Profile Service APIs:

1. `GET /api/v1/customer-details/:customerId` - Get customer details by ID
2. `GET /api/v1/customer-details/by-date-range` - Get customers by date range
3. `GET /api/v1/customer-data-deletion-request/revoking-in-progress` - Get revoking deletion requests
4. `GET /api/v1/customer-data-deletion-request/pending` - Get pending deletion requests
5. `GET /api/v1/customer-data-deletion-request/eligible` - Get eligible customers for deletion
6. `GET /api/v1/user-states/home-screen-reached` - Get customers who reached home screen
7. `POST /api/v1/migration/move-customer-details/:customerId` - Move customer details for migration

### Migration Priority:

1. **High Priority** (Used in critical flows):
   - Payment queries (Queries 1-3) - Used in payment settlement reconciliation
   - Customer credit profile queries (Queries 5-8) - Used in statement generation and reconciliation
   - Customer details queries (Queries 13-14) - Used throughout batch operations

2. **Medium Priority**:
   - Autopay subscription queries (Query 4) - Used in subscription reconciliation
   - Data deletion queries (Queries 15-17) - Used in GDPR compliance
   - Loan queries (Queries 9-11) - Used in loan disbursal and migration validation

3. **Low Priority** (Startup/Config):
   - Product mapper query (Query 10) - Called during app initialization, can be cached
   - Migration-specific queries (Queries 19-20) - Used only during migration process

### Notes:

- **Base URL Configuration**: All database queries must be replaced with API calls using service-specific base URLs (`PAYMENT_SERVICE_BASE_URL`, `LOAN_SERVICE_BASE_URL`, `PROFILE_SERVICE_BASE_URL`)
- Some queries join across multiple services (e.g., Query 4 joins payment_service with loan_service) - ensure API endpoints handle this or coordinate between services
- Multiple queries fetch similar data with different filters - consider standardizing API responses with optional filters
- Queries 5 and 7 are identical except for usage context - consider combining into a single endpoint with a parameter
- Query 20 is a migration-specific operation that may need special handling in the respo environment
- All queries use parameterized statements to prevent SQL injection - ensure API endpoints maintain this security practice
- **HTTP vs Database**: Replace `executeQuery` and `getDbInstance` calls with `axios` HTTP requests
- **Error Handling**: Update error handling from database errors to HTTP status codes (404, 500, etc.)
- **Authentication**: Add authentication headers (API keys, tokens) if required by the service APIs
- Database connections are pooled and cached per database/source combination (this will no longer be needed after migration)
- Maximum 25 connections per database pool (this will no longer be needed after migration)
- Connection idle timeout: 30 seconds (this will no longer be needed after migration)

---

## Related Documentation

- [DATA_MODEL.md](./DATA_MODEL.md) - Database schemas and data structures
- [CONFIGURATION.md](./CONFIGURATION.md) - Environment variables and configuration
- [CODE_WALKTHROUGH.md](./CODE_WALKTHROUGH.md) - Code flow and implementation details
