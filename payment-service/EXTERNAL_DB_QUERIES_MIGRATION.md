# External Database Queries Migration Guide

This document lists all database queries made to external databases (databases other than `payment_service`) that need to be converted to API calls during the service migration from easy server to respo server.

---

## 1. LOAN_SERVICE Database Queries

### 1.1 `loan_service.bank_details` Table

#### Query 1: Fetch Bank Details by ID and Customer ID
**Location:** `src/services/loan.service.js` (line 98-105)  
**Function:** `fetchBankCode()`  
**Query:**
```sql
SELECT bank_id as bankId, bank_account_no as bankAccountNumber 
FROM loan_service.bank_details 
WHERE id=:bankDetailsId AND customer_id=:customerId
```
**Purpose:** Get bank ID and account number for a specific bank details record  
**Used In:** Autopay creation flow  
**Migration:** Create API endpoint in loan-service: `GET /api/v1/bank-details/:bankDetailsId?customerId=:customerId`

---

#### Query 2: Fetch Bank Details (Full Record)
**Location:** `src/pgHandlers/RazorpayHandler.js` (line 806)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId, account_type as accountType, bank_name as bankName, 
       bank_account_no as bankAccountNumber, ifsc_code as ifscCode, 
       beneficiary_name as beneficiaryName, bank_id as bankId 
FROM loan_service.bank_details 
WHERE id=:bankDetailsId AND customer_id=:customerId
```
**Purpose:** Get complete bank details for Razorpay autopay creation  
**Migration:** Same as Query 1, but return full bank details object

---

#### Query 3: Fetch Bank Details (Billdesk)
**Location:** `src/pgHandlers/BilldeskHandler.js` (line 228)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId, account_type as accountType, bank_name as bankName, 
       bank_account_no as bankAccountNumber, ifsc_code as ifscCode, 
       beneficiary_name as beneficiaryName, bank_id as bankId 
FROM loan_service.bank_details 
WHERE id=${data.bankDetailsId} AND customer_id=${data.customerId}
```
**Purpose:** Get complete bank details for Billdesk autopay creation  
**Migration:** Same as Query 1

---

#### Query 4: Fetch Bank Details (Cashfree)
**Location:** `src/pgHandlers/CashfreeHandler.js` (line 426)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId, account_type as accountType, bank_name as bankName, 
       bank_account_no as bankAccountNumber, ifsc_code as ifscCode, 
       beneficiary_name as beneficiaryName, bank_id as bankId 
FROM loan_service.bank_details 
WHERE id=${data.bankDetailsId} AND customer_id=${data.customerId}
```
**Purpose:** Get complete bank details for Cashfree autopay creation  
**Migration:** Same as Query 1

---

#### Query 5: Fetch Bank Name by Customer ID and Account Number
**Location:** `src/script/populateBankNameScript.js` (line 31)  
**Query:**
```sql
SELECT bank_name 
FROM loan_service.bank_details 
WHERE customer_id =:customerId AND bank_account_no =:bankAccountNumber
```
**Purpose:** Script to populate bank name  
**Migration:** Create API endpoint: `GET /api/v1/bank-details/bank-name?customerId=:customerId&bankAccountNumber=:bankAccountNumber`

---

### 1.2 `loan_service.bank_list` Table

#### Query 6: Fetch Bank Codes for Razorpay
**Location:** `src/services/loan.service.js` (line 107-114)  
**Function:** `fetchBankCode()`  
**Query:**
```sql
SELECT billDesk_upi_mandate_code as BilldeskUpiBankCode, 
       razorpay_netbanking_code as razorpayNetBankCode, 
       razorpay_debitcard_code as razorpayDebitcardCode, 
       cashfree_debitcard_code as cashfreeDebitcardCode, 
       cashfree_netbanking_code as cashfreeNetBankingCode 
FROM loan_service.bank_list 
WHERE id=:bankId
```
**Purpose:** Get all payment gateway bank codes for a specific bank  
**Used In:** Autopay creation to determine which PG codes to use  
**Migration:** Create API endpoint: `GET /api/v1/bank-list/:bankId/bank-codes`

---

#### Query 7: Fetch Billdesk UPI Bank Code
**Location:** `src/pgHandlers/BilldeskHandler.js` (line 244)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT billDesk_upi_mandate_code as BilldeskUpiBankCode 
FROM loan_service.bank_list 
WHERE id=:bankId
```
**Purpose:** Get Billdesk UPI mandate code for a bank  
**Migration:** Same as Query 6, but can filter by payment gateway type

---

#### Query 8: Fetch Cashfree Bank Codes
**Location:** `src/pgHandlers/CashfreeHandler.js` (line 443)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT cashfree_netbanking_code as cashfreeNetbankingCode, 
       cashfree_debitcard_code as cashfreeDebitcardCode 
FROM loan_service.bank_list 
WHERE id=:bankId
```
**Purpose:** Get Cashfree bank codes for a bank  
**Migration:** Same as Query 6

---

### 1.3 `loan_service.customer_credit_profile` Table

#### Query 9: Fetch Customer Product ID and NBFC ID
**Location:** `src/pgHandlers/PGHandler.js` (line 293-294)  
**Function:** `getCustomersDueDate()`  
**Query:**
```sql
SELECT product_id as productId, nbfc_id as nbfcId 
FROM loan_service.customer_credit_profile 
WHERE customer_id=:customerId
```
**Purpose:** Get customer's product ID and NBFC ID to fetch due date  
**Used In:** Getting customer's EMI due date  
**Migration:** Already has API: `getCustomerCreditProfile()` in `loan.service.js` (line 55-58), but this specific query might need a dedicated endpoint or the existing API should return productId

---

### 1.4 `loan_service.loan_product_details` Table

#### Query 10: Fetch Customer Due Date
**Location:** `src/pgHandlers/PGHandler.js` (line 297-298)  
**Function:** `getCustomersDueDate()`  
**Query:**
```sql
SELECT due_date as customersDueDate 
FROM loan_service.loan_product_details 
WHERE product_id=:productId 
LIMIT 1
```
**Purpose:** Get customer's EMI due date from product details  
**Used In:** Determining when to schedule autopay charges  
**Migration:** Create API endpoint: `GET /api/v1/loan-product-details/:productId/due-date`  
**Note:** This is called after Query 9, so could be combined into a single API call

---

### 1.5 `loan_service.nbfc_config` Table

#### Query 11: Fetch All NBFC Configurations
**Location:** `src/config/nbfc.config.js` (line 21)  
**Function:** `loadNBFCConfig()`  
**Query:**
```sql
SELECT nbfc_id as nbfcId, config_data as configData 
FROM loan_service.nbfc_config
```
**Purpose:** Load all NBFC configurations (payment gateway credentials, etc.)  
**Used In:** Application startup/config loading  
**Migration:** Create API endpoint: `GET /api/v1/nbfc-config`  
**Note:** This is called during app initialization, consider caching the response

---

## 2. PROFILE_SERVICE Database Queries

### 2.1 `profile_service.customer_details` Table

#### Query 12: Fetch Customer Details by Customer ID (Razorpay)
**Location:** `src/pgHandlers/RazorpayHandler.js` (line 832-839)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId, first_name as firstName, last_name as lastName, 
       email as customerEmail, mobile_number as customerPhone 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer details for Razorpay autopay creation  
**Migration:** Already has API: `getCustomerDetailById()` in `profile.service.js` (line 6-13), but verify it returns all required fields

---

#### Query 13: Fetch Customer Details by Customer ID (Billdesk - Basic)
**Location:** `src/pgHandlers/BilldeskHandler.js` (line 255-265)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Verify customer exists for Billdesk autopay creation  
**Migration:** Use existing `getCustomerDetailById()` API or create a lightweight check endpoint

---

#### Query 14: Fetch Customer Details by Customer ID (Billdesk - Full)
**Location:** `src/pgHandlers/BilldeskHandler.js` (line 869-875)  
**Function:** `verifyMandatePayment()`  
**Query:**
```sql
SELECT mobile_number as phone, email, first_name as firstName, last_name as lastName 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer contact details for mandate verification  
**Migration:** Use existing `getCustomerDetailById()` API

---

#### Query 15: Fetch Customer Details by Customer ID (Cashfree)
**Location:** `src/pgHandlers/CashfreeHandler.js` (line 454-464)  
**Function:** `fetchBankDetails()`  
**Query:**
```sql
SELECT customer_id as customerId, first_name as firstName, last_name as lastName, 
       email as customerEmail, mobile_number as customerPhone, name_as_per_pan as customerName 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer details for Cashfree autopay creation  
**Migration:** Use existing `getCustomerDetailById()` API, ensure it returns `name_as_per_pan`

---

#### Query 16: Fetch Customer Details by Customer ID (Payment Controller)
**Location:** `src/controllers/paymentController.js` (line 59-63)  
**Query:**
```sql
SELECT email as email, first_name as firstName, mobile_number as mobileNumber 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer details for payment processing  
**Migration:** Use existing `getCustomerDetailById()` API

---

#### Query 17: Fetch Customer Details by Customer ID (Payment Controller - Second Location)
**Location:** `src/controllers/paymentController.js` (line 188-192)  
**Query:**
```sql
SELECT email, first_name as firstName, mobile_number as mobileNumber 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer details for payment processing  
**Migration:** Use existing `getCustomerDetailById()` API

---

#### Query 18: Fetch Customer Email and Phone (Razorpay)
**Location:** `src/pgHandlers/RazorpayHandler.js` (line 1217-1224)  
**Function:** `createAutopayCharge()`  
**Query:**
```sql
SELECT email, mobile_number as customerPhone 
FROM profile_service.customer_details 
WHERE customer_id=:customerId
```
**Purpose:** Get customer email and phone for charge notification  
**Migration:** Use existing `getCustomerDetailById()` API

---

#### Query 19: Verify Customer by UID (Middleware)
**Location:** `src/middlewares/verifyCustomer.js` (line 7-14)  
**Function:** `verifyCustomer()` middleware  
**Query:**
```sql
SELECT uid, customer_id as customerId 
FROM profile_service.customer_details 
WHERE uid=:uid
```
**Purpose:** Verify customer authentication by UID  
**Migration:** Create API endpoint: `GET /api/v1/customer/verify-by-uid/:uid` or use existing customer details API with UID filter

---

#### Query 20: Fetch Customer ID by Mobile Number (BBPS)
**Location:** `src/controllers/bbpsPaymentEventController.js` (line 42-48)  
**Function:** `bbpsPaymentEvent()`  
**Query:**
```sql
SELECT customer_id 
FROM profile_service.customer_details 
WHERE mobile_number = :customerMobileNumber
```
**Purpose:** Get customer ID from mobile number for BBPS payment event  
**Migration:** Create API endpoint: `GET /api/v1/customer/by-mobile/:mobileNumber` or enhance existing API to support mobile number lookup

---

#### Query 21: Fetch Customer Details by Mobile Number (BBPS Callback)
**Location:** `src/controllers/bbpsPaymentEventController.js` (line 105-111)  
**Function:** `bbpsPaymentEventCallBack()`  
**Query:**
```sql
SELECT * 
FROM profile_service.customer_details 
WHERE mobile_number = :mobileNumber
```
**Purpose:** Get full customer details from mobile number for BBPS callback  
**Migration:** Same as Query 20

---

## Summary

### Total Queries by Database:
- **loan_service**: 11 queries
- **profile_service**: 10 queries
- **Total**: 21 queries

### Recommended API Endpoints to Create:

#### Loan Service APIs:
1. `GET /api/v1/bank-details/:bankDetailsId` - Get bank details by ID (with customerId validation)
2. `GET /api/v1/bank-list/:bankId/bank-codes` - Get all PG bank codes for a bank
3. `GET /api/v1/loan-product-details/:productId/due-date` - Get due date for a product
4. `GET /api/v1/nbfc-config` - Get all NBFC configurations
5. Enhance `GET /api/v1/customer-credit-profile/:customerId` to include `productId` if not already included

#### Profile Service APIs:
1. Enhance `GET /api/v1/internalcall/customerDetails/:customerId` to ensure it returns all required fields:
   - `customerId`, `firstName`, `lastName`, `email`, `mobileNumber`, `name_as_per_pan`, `uid`
2. `GET /api/v1/customer/by-mobile/:mobileNumber` - Get customer by mobile number
3. `GET /api/v1/customer/verify-by-uid/:uid` - Verify customer by UID (or enhance existing endpoint)

### Migration Priority:
1. **High Priority** (Used in critical flows):
   - Bank details queries (Queries 1-5) - Used in autopay creation
   - Bank list queries (Queries 6-8) - Used in autopay creation
   - Customer details queries (Queries 12-18) - Used throughout payment flows

2. **Medium Priority**:
   - Customer credit profile and due date queries (Queries 9-10) - Used for scheduling
   - Customer verification queries (Queries 19-21) - Used in authentication and BBPS

3. **Low Priority** (Startup/Config):
   - NBFC config query (Query 11) - Called during app initialization, can be cached

### Notes:
- Some queries are already covered by existing API calls (e.g., `getCustomerDetailById()`, `getCustomerCreditProfile()`)
- Multiple queries fetch the same data with different field selections - consider standardizing the API response
- Some queries use string interpolation instead of parameterized queries (BilldeskHandler, CashfreeHandler) - ensure API calls are secure
- The `verifyCustomer` middleware query should be converted to an API call for security and consistency

