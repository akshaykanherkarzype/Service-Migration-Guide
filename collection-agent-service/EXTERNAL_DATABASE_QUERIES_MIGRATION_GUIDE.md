# External Database Queries Migration Guide

This document lists all database queries made to external databases (databases other than `collection_agent_app`) that need to be converted to API calls during the service migration from easy server to respo server.

**Note:** This codebase does not contain any `payment_service` database queries. The external databases used are: `loan_service`, `profile_service`, `account_service`, and `underwriting_service`.

---

## 1. LOAN_SERVICE Database Queries

### 1.1 `loan_service.nbfc_config` Table

#### Query 1: Fetch All NBFC Configurations
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 3)  
**Function:** `buildNbfcConfigQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/utils.ts` (line 22-26) - `loadNbfcConfig()`  
**Query:**
```sql
SELECT * FROM loan_service.nbfc_config
```
**Purpose:** Load all NBFC configurations (payment gateway credentials, Finflux tokens, etc.) during application startup  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/nbfc-config`  
**Note:** Called during app initialization; consider caching the response

---

### 1.2 `loan_service.customer_credit_profile` Table

#### Query 2: Fetch Customer Credit Profile
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 5-10)  
**Function:** `buildCustomerCreditProfileQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 43-58) - `fetchCustomerCreditProfile()`  
**Query:**
```sql
SELECT 
    nbfc_id as nbfcId,
    lms_client_id as clientId,
    advance_amount as advanceAmount
FROM loan_service.customer_credit_profile
WHERE customer_id = ${userId}
```
**Purpose:** Get customer's NBFC ID, LMS client ID, and advance amount for loan operations  
**Migration:** Create API endpoint: `GET /api/v1/customer-credit-profile/:customerId`

---

#### Query 3: Fetch Bill Date and Due Date (with loan_product_details JOIN)
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 49-58)  
**Function:** `buildBillDateAndDueDateQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 116-123) - `fetchGeneralDetails()`  
**Query:**
```sql
SELECT
    t1.customer_id as customerId,
    t1.product_id as productId,
    t2.bill_date as billDate, 
    t2.due_date as dueDate 
FROM loan_service.customer_credit_profile AS t1  
LEFT JOIN loan_service.loan_product_details AS t2 
ON t1.product_id = t2.product_id 
WHERE t1.customer_id = ${userId} 
GROUP BY t1.customer_id
```
**Purpose:** Get customer's bill date and due date for general details display  
**Migration:** Create API endpoint: `GET /api/v1/customer/:customerId/bill-and-due-date`

---

#### Query 4: Fetch Client ID
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 125-128)  
**Function:** `buildClientIdQuery()`  
**Query:**
```sql
SELECT 
    lms_client_id as clientId
FROM loan_service.customer_credit_profile 
WHERE customer_id = ${userId}
```
**Purpose:** Get LMS client ID for a customer (used for loan overview, approved amount, etc.)  
**Migration:** Can be combined with Query 2 or create: `GET /api/v1/customer-credit-profile/:customerId` (already returns clientId)

---

### 1.3 `loan_service.loan_product_details` Table

**Note:** This table is accessed via JOIN in Query 3 (buildBillDateAndDueDateQuery). See Query 3 above.

---

### 1.4 `loan_service.m_loan` Table

#### Query 5: Fetch Approved Amount (Total Principal)
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 60-63)  
**Function:** `buildApprovedAmountQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 129-136) - `fetchGeneralDetails()`  
**Query:**
```sql
SELECT 
    sum(t2.principal_amount_proposed) AS totalAmount 
FROM loan_service.m_loan AS t2 
WHERE t2.loan_status_id = '200'
AND t2.client_id = ${clientId}
```
**Purpose:** Get total approved loan amount for loans in "Waiting For disbursal" status  
**Migration:** Create API endpoint: `GET /api/v1/loans/approved-amount?clientId=:clientId`

---

### 1.5 `loan_service.bank_account_transfer` Table

#### Query 6: Fetch Loan Disbursal Details
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 130-142)  
**Function:** `buildLoanDisbursalQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 357-375) - `fetchLoanDisbursalDetails()`  
**Query:**
```sql
SELECT 
    amount,
    ifsc_code AS ifscCode,  
    CONCAT(REPEAT('*', 6), SUBSTR(account_no, CHAR_LENGTH(account_no)-4, CHAR_LENGTH(account_no))) AS accountNumber,
    CONVERT_TZ(updated_date,'+00:00','+00:00') AS disbursedDate,
    transfer_status AS transferStatus,
    loan_id AS loanId
FROM loan_service.bank_account_transfer 
WHERE customer_id = ${userId} 
AND loan_id = ${loanId}
AND purpose='loandisbursal' 
AND transfer_status='settlementcompl'
```
**Purpose:** Get loan disbursal details (amount, IFSC, masked account, disbursed date) for loan overview  
**Migration:** Create API endpoint: `GET /api/v1/loans/:loanId/disbursal-details?customerId=:customerId`

---

## 2. PROFILE_SERVICE Database Queries

### 2.1 `profile_service.customer_references` Table

#### Query 7: Fetch Customer References
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 13-14)  
**Function:** `buildReferenceDetailsQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 202-209) - `fetchKycDetails()`  
**Query:**
```sql
SELECT * FROM profile_service.customer_references
WHERE customer_id = ${userId}
```
**Purpose:** Get customer reference details for KYC details display  
**Migration:** Create API endpoint: `GET /api/v1/customer/:customerId/references`

---

### 2.2 `profile_service.customer_details` Table (in General Details JOIN)

#### Query 8: Fetch General Details (Multi-Table JOIN)
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 16-47)  
**Function:** `buildGeneralDetailsQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 98-105) - `fetchGeneralDetails()`  
**Query:**
```sql
SELECT 
    t1.dob,
    t1.email,
    t1.mobile_number as mobileNumber,
    t1.os_name as osVersion,
    t4.nbfc_id as nbfcId,
    CONCAT(address_line,',',city,',',state,',',country,',',pincode) as gmapAddress,
    t9.created_date as registeredDate, 
    t9.l1_status_date as l1StatusDate, 
    t9.l2_status_date as l2StatusDate, 
    CASE 
        WHEN t6.bank_statement_id IS NULL THEN 'NO'
        ELSE 'YES'
    END AS bankStatement,
    CASE 
        WHEN t7.id THEN 'YES'
        ELSE 'NO'
    END AS isBookmarked
FROM profile_service.customer_details AS t1
LEFT JOIN profile_service.experian_details AS t3 
    ON t3.customer_id = t1.customer_id
LEFT JOIN loan_service.customer_credit_profile AS t4 
    ON t4.customer_id = t1.customer_id
LEFT JOIN profile_service.auto_gmap_address AS t5 
    ON t5.customer_id = t1.customer_id
LEFT JOIN account_service.finbox_statement_data AS t6 
    ON t6.customer_id = t1.customer_id 
LEFT JOIN underwriting_service.bureau_details AS t7 
    ON t7.customer_id = t1.customer_id
LEFT JOIN profile_service.user_states AS t9
    ON t9.customer_id = t1.customer_id
WHERE t1.customer_id = ${userId}
```
**Purpose:** Get comprehensive customer general details including DOB, email, mobile, OS version, NBFC ID, address, registration dates, L1/L2 status dates, bank statement flag, and bureau bookmark flag  
**Migration:** This is a complex multi-service JOIN. Options:
- Create a composite API in a gateway/BFF: `GET /api/v1/customer/:customerId/general-details`
- Or split into multiple API calls and merge in collection-agent-service

**Tables involved:**
- `profile_service.customer_details` (t1)
- `profile_service.experian_details` (t3)
- `loan_service.customer_credit_profile` (t4)
- `profile_service.auto_gmap_address` (t5)
- `account_service.finbox_statement_data` (t6)
- `underwriting_service.bureau_details` (t7)
- `profile_service.user_states` (t9)

---

## 3. ACCOUNT_SERVICE Database Queries

### 3.1 `account_service.finbox_statement_data` Table

#### Query 9: Check Bank Statement Existence (via General Details JOIN)
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 41)  
**Function:** `buildGeneralDetailsQuery()` - LEFT JOIN with t6  
**Used In:** Part of Query 8 - `fetchGeneralDetails()`  
**Query (relevant portion):**
```sql
LEFT JOIN account_service.finbox_statement_data AS t6 
ON t6.customer_id = t1.customer_id 
-- Used in: CASE WHEN t6.bank_statement_id IS NULL THEN 'NO' ELSE 'YES' END AS bankStatement
```
**Purpose:** Determine if customer has bank statement (bankStatement: YES/NO)  
**Migration:** Create API endpoint in account-service: `GET /api/v1/customer/:customerId/bank-statement-status`  
**Alternative:** Include in composite general-details API

---

## 4. UNDERWRITING_SERVICE Database Queries

### 4.1 `underwriting_service.customer_underwriting_data` Table

#### Query 10: Fetch Reject Reason
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 65-124)  
**Function:** `buildRejectReasonQuery()`  
**Used In:** `src/libs/databases/src/lib/v1/index.ts` (line 137-144) - `fetchGeneralDetails()`  
**Query:**
```sql
SELECT  
    CASE WHEN error_code IS NOT NULL THEN 
        CASE error_code 
            WHEN "L1_latest30dpd_reject" THEN "Negative Loan Performance"
            WHEN "L1_device_check_reject" THEN "Suspected fraud" 
            WHEN "L1_bounce_sms_reject" THEN "Negative Loan Performance" 
            -- ... (full error_code mapping - see queryBuilder.ts lines 68-116)
            ELSE @default_var 
        END 
    END AS rejectReason 
FROM underwriting_service.customer_underwriting_data  
WHERE customer_id = ${userId}
AND error_code IS NOT NULL 
ORDER BY updated_at DESC 
LIMIT 1
```
**Purpose:** Get human-readable reject reason for rejected customers  
**Migration:** Create API endpoint: `GET /api/v1/customer/:customerId/reject-reason`

---

### 4.2 `underwriting_service.bureau_details` Table (in General Details JOIN)

#### Query 11: Check Bureau Bookmark (isBookmarked)
**Location:** `src/libs/databases/src/lib/v1/queryBuilder.ts` (line 43-44)  
**Function:** `buildGeneralDetailsQuery()` - LEFT JOIN with t7  
**Used In:** Part of Query 8 - `fetchGeneralDetails()`  
**Query (relevant portion):**
```sql
LEFT JOIN underwriting_service.bureau_details AS t7 
ON t7.customer_id = t1.customer_id
-- Used in: CASE WHEN t7.id THEN 'YES' ELSE 'NO' END AS isBookmarked
```
**Purpose:** Determine if customer has bureau details bookmarked  
**Migration:** Create API endpoint: `GET /api/v1/customer/:customerId/bureau-bookmark-status`  
**Alternative:** Include in composite general-details API

---

## Summary

### Total Queries by Database:
- **loan_service**: 6 queries (across 5 tables)
- **profile_service**: 2 queries (customer_references + general details JOIN)
- **account_service**: 1 query (finbox_statement_data in JOIN)
- **underwriting_service**: 2 queries (reject reason + bureau_details in JOIN)
- **Total**: 11 distinct query patterns

### Payment Service:
**No `payment_service` database queries exist in this codebase.**

---

### Recommended API Endpoints to Create:

#### Loan Service APIs:
1. `GET /api/v1/nbfc-config` - Get all NBFC configurations (startup/config)
2. `GET /api/v1/customer-credit-profile/:customerId` - Get nbfcId, clientId, advanceAmount
3. `GET /api/v1/customer/:customerId/bill-and-due-date` - Get bill date and due date
4. `GET /api/v1/loans/approved-amount?clientId=:clientId` - Get total approved amount
5. `GET /api/v1/loans/:loanId/disbursal-details?customerId=:customerId` - Get loan disbursal details

#### Profile Service APIs:
1. `GET /api/v1/customer/:customerId/references` - Get customer references
2. `GET /api/v1/customer/:customerId/general-details` - Composite endpoint (or split into profile + other services)

#### Account Service APIs:
1. `GET /api/v1/customer/:customerId/bank-statement-status` - Check if bank statement exists

#### Underwriting Service APIs:
1. `GET /api/v1/customer/:customerId/reject-reason` - Get human-readable reject reason
2. `GET /api/v1/customer/:customerId/bureau-bookmark-status` - Check if bureau bookmarked

---

### Migration Priority:

1. **High Priority** (Critical flows - General Details, Loan Overview):
   - General Details composite query (Query 8) - Used in customer details view
   - Bill date and due date (Query 3)
   - Customer credit profile (Query 2)
   - Loan disbursal details (Query 6)
   - Approved amount (Query 5)
   - Reject reason (Query 10)

2. **Medium Priority** (KYC, Config):
   - Customer references (Query 7)
   - NBFC config (Query 1) - Startup; can be cached

3. **Low Priority** (Part of composite):
   - Bank statement status (Query 9)
   - Bureau bookmark status (Query 11)

---

### Notes:

- **Security:** Queries use string interpolation (`${userId}`) - ensure migrated APIs use parameterized inputs
- **General Details:** The `buildGeneralDetailsQuery` is a complex cross-service JOIN. Consider a BFF/gateway composite API or orchestration layer
- **Existing API Usage:** The service already uses several external APIs (e.g., `ApiEndpointsEnum.GENERAL_DETAILS`, `ApiEndpointsEnum.USER_STATE`, `ApiEndpointsEnum.BUREAU_DETAILS`) - some DB queries run in parallel with API calls; migration should consolidate to API-only where possible
- **Database Version:** Queries use `NbfcAdapter.NbfcProviderEnum.RPN` database connection; RESPO may have different schema/APIs
