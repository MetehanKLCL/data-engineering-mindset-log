# 📅 Daily Case: The Right to be Forgotten (Data Governance & PII)

**Date:** 2026-01-30  
**Tags:** #DataGovernance #GDPR #PII #CryptoShredding #Security #Architecture

## 1. Problem Statement
The company is expanding globally and must comply with strict data privacy laws (GDPR/KVKK).
- **Challenge 1:** Marketing needs to analyze user behavior without seeing sensitive PII (Personally Identifiable Information).
- **Challenge 2:** We must implement the **"Right to be Forgotten"**. Deleting a specific user's data from petabytes of immutable Parquet files in the Data Lake is technically difficult and computationally expensive.

## 2. Architect Decisions

### Decision 1: Masking Strategy (Ingestion Level)
**Question:** Where should we hide the sensitive data?
**Decision:** Apply **Dynamic Masking** or **Hashing** at the **Ingestion Layer** (before writing to Bronze/Raw).
**Reasoning:**
- If we store raw PII in the Data Lake, it becomes a permanent security risk.
- Masking at the source ensures that even if the storage is compromised, the data remains unreadable.

### Decision 2: Deletion Strategy (Crypto-Shredding)
**Question:** How to delete a user from immutable files efficiently?
**Decision:** Implement **Crypto-Shredding** (Tokenization).
**Mechanism:**
1.  Separate PII from Activity Data.
2.  Store the mapping (User ID <-> Token) in a secure, ACID-compliant **Key Store**.
3.  Write Activity Data to the Data Lake using only the **Token**.
4.  **To Delete:** Simply delete the key/token from the Key Store. The massive amount of activity data in the Lake becomes instantly orphaned and anonymous (mathematically deleted).

### Decision 3: Access Control (Tag-Based Governance)
**Question:** How to manage permissions at scale?
**Decision:** Use **Metadata-Driven Access Control (ABAC)**.
**Reasoning:**
- Instead of granting permissions table-by-table, we tag columns (e.g., `tag:PII`, `tag:Financial`).
- Policies are defined globally: "Analysts cannot view columns with `tag:PII`".
- This decouples security rules from the physical data structure.

## 3. Conceptual Architecture (Crypto-Shredding)

```plaintext
[Incoming Data] --> {User: "Ali", IP: "127.0.0.1"}
       |
       v
[Tokenization Service] <--> [Secure Key Vault (Oracle/Postgres)]
       |                    (Stores: Ali = Token_999)
       |
       v
[Data Lake Storage]
(Stores: {User: "Token_999", IP: "127.0.0.1"})

-- DELETE REQUEST --
Action: DELETE FROM Key_Vault WHERE User="Ali"
Result: "Token_999" in Data Lake is now meaningless garbage.
```

## 4. Key Learnings
- **Crypto-Shredding:** Don't try to delete data from Big Data; delete the *key* that makes the data readable.
- **Ingestion Masking:** The safest data is the data you never stored in clear text.
- **Policy over Grants:** Manage security via Tags (Metadata), not via individual SQL grants.
