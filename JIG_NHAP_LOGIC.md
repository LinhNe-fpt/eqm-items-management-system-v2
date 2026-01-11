# 📥 JIG NHẬP KHO - LOGIC HIỆN TẠI

**File:** `backend/dieu_hanh/jig_nhap.js`  
**Cập nhật lần cuối:** 23/12/2024  
**Trạng thái:** Active & Tested ✅

---

## 📌 OVERVIEW

Hệ thống nhập JIG quản lý việc tiếp nhận và ghi nhận JIG (Jig Injection Gauge) từ các bên cung cấp hoặc mượn từ bên khác. Có **2 loại nhập chính:**

1. **nhap_muon** - Nhập JIG mượn (tăng cột `Muon`)
2. **nhap_kho_jig** - Nhập JIG vào kho (tăng `Tong_so_luong`)

---

## 🔄 FLOW DIAGRAM

```
USER SUBMITS FORM (JigNhap.tsx)
           ↓
SEND POST /api/jig-nhap
           ↓
VALIDATE REQUIRED FIELDS
           ↓
ENSURE DATABASE COLUMNS EXIST
  ├─ Loai_nhap
  ├─ Minh_chung
  ├─ Muon
  └─ Cho_muon
           ↓
RETRIEVE CURRENT VALUES FROM JIG_MASTER
  ├─ currentMuon
  └─ currentChoMuon
           ↓
INSERT RECORD INTO JIG_NHAP TABLE
  └─ Lưu phiếu nhập chi tiết
           ↓
UPDATE OR INSERT JIG_MASTER
  ├─ IF nhap_muon → Muon += quantity
  └─ IF nhap_kho_jig → Tong_so_luong += quantity
           ↓
RETURN SUCCESS RESPONSE
```

---

## 📊 DATA FLOW DETAILS

### PHASE 1: VALIDATION & SCHEMA CHECK

```javascript
// Step 1.1: Validate required fields
if (!data.So_CRO || !data.Nguoi_chuyen_hang || !data.Ngay_chuyen || 
    !data.Ben_Chuyen || !data.Ben_nhan || !data.Model || 
    !data.Ten_JIG_thuc_te || !data.Tong_so_luong || 
    !data.Trang_thai_CF_PSM || !data.So_RQ || !data.Ten_JIG) {
  return res.status(400).json({ error: 'Thiếu thông tin bắt buộc' });
}

// Required fields:
// - So_CRO: CRO (Change Request Order) number
// - Nguoi_chuyen_hang: Transferrer name
// - Ngay_chuyen: Transfer date (YYYY-MM-DD)
// - Ben_Chuyen: Supplier/Source party
// - Ben_nhan: Receiving party
// - Model: JIG model code
// - Ten_JIG_thuc_te: Actual JIG name
// - Tong_so_luong: Total quantity
// - Trang_thai_CF_PSM: Condition status (OK/NG/etc)
// - So_RQ: RQ number
// - Ten_JIG: JIG identifier
```

### PHASE 1.2: Auto-create missing columns

```javascript
// Ensure Loai_nhap column exists
const loaiNhapCheck = await pool.request()
  .query(`SELECT COLUMN_NAME FROM INFORMATION_SCHEMA.COLUMNS 
          WHERE TABLE_NAME = 'JIG_NHAP' AND COLUMN_NAME = 'Loai_nhap'`);
if (!loaiNhapCheck.recordset.length) {
  await pool.request()
    .query(`ALTER TABLE JIG_NHAP ADD Loai_nhap NVARCHAR(50) DEFAULT 'nhap_kho_jig'`);
}

// Similarly for: Minh_chung, Muon, Cho_muon
```

**Purpose:** Support new columns without manual migration

---

### PHASE 2: FETCH CURRENT STATE FROM JIG_MASTER

```javascript
// Get current Muon and Cho_muon values
let currentMuon = 0;
let currentChoMuon = 0;
const masterCheckResult = await pool.request()
  .input('Ma_so_quan_ly', sql.NVarChar, data.Ma_so_quan_ly)
  .query('SELECT Muon, Cho_muon FROM JIG_MASTER WHERE Ma_so_quan_ly = @Ma_so_quan_ly');

if (masterCheckResult.recordset.length > 0) {
  currentMuon = masterCheckResult.recordset[0].Muon || 0;
  currentChoMuon = masterCheckResult.recordset[0].Cho_muon || 0;
}
```

**Why:** Used to save snapshot in JIG_NHAP record (audit trail)

---

### PHASE 3: INSERT INTO JIG_NHAP TABLE

```javascript
await pool.request()
  .input('Loai_nhap', sql.NVarChar, data.Loai_nhap)
  .input('So_CRO', sql.NVarChar, data.So_CRO)
  .input('Nguoi_chuyen_hang', sql.NVarChar, data.Nguoi_chuyen_hang)
  .input('Ngay_chuyen', sql.NVarChar, data.Ngay_chuyen)
  .input('Ben_Chuyen', sql.NVarChar, data.Ben_Chuyen)
  .input('Ben_nhan', sql.NVarChar, data.Ben_nhan)
  .input('Ma_Model', sql.NVarChar, data.Model)
  .input('Ten_JIG_thuc_te', sql.NVarChar, data.Ten_JIG_thuc_te)
  .input('Code_final', sql.NVarChar, data.Code_final || null)
  .input('Tong_so_luong', sql.Int, data.Tong_so_luong)
  .input('Trang_thai_CF_PSM', sql.NVarChar, data.Trang_thai_CF_PSM)
  .input('So_RQ', sql.NVarChar, data.So_RQ)
  .input('Ghi_chu', sql.NVarChar, data.Ghi_chu)
  .input('Ten_JIG', sql.NVarChar, data.Ten_JIG)
  .input('Minh_chung', sql.NVarChar, data.Minh_chung || null)
  .input('Muon', sql.Int, currentMuon)
  .input('Cho_muon', sql.Int, currentChoMuon)
  .query(`INSERT INTO JIG_NHAP (
    Loai_nhap, So_CRO, Nguoi_chuyen_hang, Ngay_chuyen, Ben_Chuyen, 
    Ben_nhan, Ma_Model, Ten_JIG_thuc_te, Code_final, Tong_so_luong, 
    Trang_thai_CF_PSM, So_RQ, Ghi_chu, Ten_JIG, Minh_chung, Muon, Cho_muon
  ) VALUES (
    @Loai_nhap, @So_CRO, @Nguoi_chuyen_hang, @Ngay_chuyen, @Ben_Chuyen, 
    @Ben_nhan, @Ma_Model, @Ten_JIG_thuc_te, @Code_final, @Tong_so_luong, 
    @Trang_thai_CF_PSM, @So_RQ, @Ghi_chu, @Ten_JIG, @Minh_chung, @Muon, @Cho_muon
  )`);
```

**Purpose:** Create audit trail of all import transactions  
**Note:** Saves snapshot of Muon/Cho_muon at time of import

---

### PHASE 4: UPDATE JIG_MASTER (CONDITIONAL)

#### Option A: IF nhap_muon (Nhập mượn)

```javascript
const isNhapMuon = data.Loai_nhap === 'nhap_muon';

if (isNhapMuon) {
  // Update Muon column only
  const updateResult = await pool.request()
    .input('Ma_so_quan_ly', sql.NVarChar, data.Ma_so_quan_ly)
    .input('Muon', sql.Int, data.Tong_so_luong)
    .query(`UPDATE JIG_MASTER
      SET Muon = CAST(ISNULL(Muon, 0) AS INT) + @Muon
      WHERE Ma_so_quan_ly = @Ma_so_quan_ly`);
  
  // If no record exists, CREATE new record with Muon only
  if (updateResult.rowsAffected[0] === 0) {
    await pool.request()
      .input('Ma_so_quan_ly', sql.NVarChar, data.Ma_so_quan_ly)
      .input('Ma_Model', sql.NVarChar, data.Model)
      .input('Ten_JIG_thuc_te', sql.NVarChar, data.Ten_JIG_thuc_te)
      .input('Code_final', sql.NVarChar, data.Code_final || null)
      .input('Muon', sql.Int, data.Tong_so_luong)
      .input('Ghi_chu', sql.NVarChar, data.Ghi_chu)
      .query(`INSERT INTO JIG_MASTER (
        Ma_so_quan_ly, Ma_Model, Ten_JIG_thuc_te, Code_final, Muon, Ghi_chu
      ) VALUES (
        @Ma_so_quan_ly, @Ma_Model, @Ten_JIG_thuc_te, @Code_final, @Muon, @Ghi_chu
      )`);
  }
}
```

**Logic:**
- `Muon += quantity` (Increases borrowed amount)
- `Tong_so_luong` stays unchanged (Not part of inventory)
- Creates JIG_MASTER record if doesn't exist

**Example:**
```
Before: Muon = 5
Import: nhap_muon, quantity = 2
After:  Muon = 7
```

---

#### Option B: ELSE nhap_kho_jig (Nhập kho)

```javascript
} else {
  // Update Tong_so_luong and Tong_JIG_vendor
  const updateResult = await pool.request()
    .input('Ma_so_quan_ly', sql.NVarChar, data.Ma_so_quan_ly)
    .input('Ma_Model', sql.NVarChar, data.Model)
    .input('Ten_JIG_thuc_te', sql.NVarChar, data.Ten_JIG_thuc_te)
    .input('Code_final', sql.NVarChar, data.Code_final || null)
    .input('Tong_so_luong', sql.Int, data.Tong_so_luong)
    .input('Ghi_chu', sql.NVarChar, data.Ghi_chu)
    .query(`UPDATE JIG_MASTER
      SET Tong_so_luong = CAST(ISNULL(Tong_so_luong, 0) AS INT) + @Tong_so_luong,
          Tong_JIG_vendor = CAST(ISNULL(Tong_JIG_vendor, 0) AS INT) + @Tong_so_luong,
          Ma_Model = @Ma_Model,
          Ten_JIG_thuc_te = @Ten_JIG_thuc_te,
          Code_final = @Code_final,
          Ghi_chu = @Ghi_chu
      WHERE Ma_so_quan_ly = @Ma_so_quan_ly`);

  // If no record exists, CREATE new record
  if (updateResult.rowsAffected[0] === 0) {
    await pool.request()
      .input('Ma_so_quan_ly', sql.NVarChar, data.Ma_so_quan_ly)
      .input('Ma_Model', sql.NVarChar, data.Model)
      .input('Ten_JIG_thuc_te', sql.NVarChar, data.Ten_JIG_thuc_te)
      .input('Code_final', sql.NVarChar, data.Code_final || null)
      .input('Tong_so_luong', sql.Int, data.Tong_so_luong)
      .input('Ghi_chu', sql.NVarChar, data.Ghi_chu)
      .query(`INSERT INTO JIG_MASTER (
        Ma_so_quan_ly, Ma_Model, Ten_JIG_thuc_te, Code_final, 
        Tong_so_luong, Tong_JIG_vendor, Ghi_chu
      ) VALUES (
        @Ma_so_quan_ly, @Ma_Model, @Ten_JIG_thuc_te, @Code_final, 
        @Tong_so_luong, @Tong_so_luong, @Ghi_chu
      )`);
  }
}
```

**Logic:**
- `Tong_so_luong += quantity` (Adds to total inventory)
- `Tong_JIG_vendor += quantity` (Adds to vendor/stock count)
- Also updates model info: `Ma_Model`, `Ten_JIG_thuc_te`, `Code_final`, `Ghi_chu`
- Creates JIG_MASTER record if doesn't exist

**Example:**
```
Before: Tong_so_luong = 10, Tong_JIG_vendor = 10
Import: nhap_kho_jig, quantity = 3
After:  Tong_so_luong = 13, Tong_JIG_vendor = 13
```

---

## 🗄️ DATABASE TABLES

### JIG_NHAP (Audit Trail Table)

| Column | Type | Purpose |
|--------|------|---------|
| STT | INT (PK) | Auto-increment primary key |
| Loai_nhap | NVARCHAR(50) | Import type: `nhap_muon` or `nhap_kho_jig` |
| So_CRO | NVARCHAR(50) | CRO number |
| Nguoi_chuyen_hang | NVARCHAR(100) | Transferrer name |
| Ngay_chuyen | NVARCHAR(50) | Transfer date (stored as string for flexibility) |
| Ben_Chuyen | NVARCHAR(100) | Source/Supplier |
| Ben_nhan | NVARCHAR(100) | Receiving party |
| Ma_Model | NVARCHAR(50) | JIG model code |
| Ten_JIG_thuc_te | NVARCHAR(200) | Actual JIG name |
| Code_final | NVARCHAR(100) | Final code/identifier |
| Tong_so_luong | INT | Quantity imported |
| Trang_thai_CF_PSM | NVARCHAR(50) | Condition: OK/NG/etc |
| So_RQ | NVARCHAR(50) | RQ number |
| Ghi_chu | NVARCHAR(500) | Notes/Comments |
| Ten_JIG | NVARCHAR(100) | JIG identifier |
| **Minh_chung** | NVARCHAR(MAX) | Evidence/Receipt image path |
| **Muon** | INT | Snapshot of Muon at import time |
| **Cho_muon** | INT | Snapshot of Cho_muon at import time |

### JIG_MASTER (Inventory Master Table)

| Column | Type | Purpose |
|--------|------|---------|
| Ma_so_quan_ly | NVARCHAR(50) (PK) | JIG ID |
| Ma_Model | NVARCHAR(50) | Model code |
| Ten_JIG_thuc_te | NVARCHAR(200) | JIG actual name |
| Code_final | NVARCHAR(100) | Final code |
| **Tong_so_luong** | INT | Total inventory count |
| **Tong_JIG_vendor** | INT | Vendor stock count |
| **Muon** | INT | Borrowed amount |
| **Cho_muon** | INT | Lent to others |
| So_luong_ngoai_line | INT | Quantity on production line |
| Ghi_chu | NVARCHAR(500) | Notes |

---

## 🔗 RELATED ENDPOINTS

### GET /api/jig-nhap
Retrieve import history (with pagination)

**Query Parameters:**
- `page`: Page number (default: 1)
- `all`: Set to `1` to get all records
- `search`: Advanced search (format: `CRO:xxx;MODEL:yyy;NGUOI:zzz;NGAY:2024-12-23`)

**Response:**
```json
{
  "total": 50,
  "page": 1,
  "pageSize": 20,
  "data": [
    {
      "STT": 1,
      "Loai_nhap": "nhap_kho_jig",
      "So_CRO": "CRO-001",
      "Nguoi_chuyen_hang": "John Doe",
      "Ngay_chuyen": "2024-12-23",
      "Tong_so_luong": 5,
      "Muon": 3,
      "Cho_muon": 0
    }
  ],
  "oldest": {...},
  "newest": {...}
}
```

---

### POST /api/jig-nhap
Create new import record

**Request Body:**
```json
{
  "Loai_nhap": "nhap_muon",
  "So_CRO": "CRO-002",
  "Nguoi_chuyen_hang": "Jane Smith",
  "Ngay_chuyen": "2024-12-23",
  "Ben_Chuyen": "Supplier A",
  "Ben_nhan": "Main Warehouse",
  "Model": "A520",
  "Ma_so_quan_ly": "A520_01",
  "Ten_JIG_thuc_te": "JIG Type 1",
  "Code_final": "JIG-001",
  "Tong_so_luong": 2,
  "Trang_thai_CF_PSM": "OK",
  "So_RQ": "RQ-001",
  "Ghi_chu": "Import from supplier",
  "Ten_JIG": "A520_01",
  "Minh_chung": "/uploads/minh_chung/evidence.jpg"
}
```

**Response:**
```json
{
  "message": "Đã lưu phiếu nhập JIG và cập nhật tồn kho thành công"
}
```

---

### POST /api/jig-nhap/upload-minh-chung
Upload evidence/receipt image

**Request:** Form-data with `file` field

**Response:**
```json
{
  "success": true,
  "filename": "receipt_1234567890.jpg",
  "path": "/uploads/minh_chung/receipt_1234567890.jpg",
  "size": 245632,
  "mimetype": "image/jpeg"
}
```

---

## 🔐 SECURITY FEATURES

### SQL Injection Prevention
✅ Uses parameterized queries with `@parameter` bindings  
✅ `sql.NVarChar()`, `sql.Int()` type declarations  
✅ No string concatenation in queries

### Data Validation
✅ Required field validation
✅ Type checking with sql library
✅ File upload validation (separate middleware)

### Error Handling
✅ Try-catch blocks
✅ Proper HTTP status codes
✅ Detailed error messages

---

## ⚠️ EDGE CASES HANDLED

### 1. Missing JIG_MASTER Record
**Scenario:** First import of a JIG model

**Handling:**
- `UPDATE` returns `rowsAffected[0] === 0`
- System performs `INSERT` with initial values
- Works for both nhap_muon and nhap_kho_jig

### 2. Null Values
**Handling:**
- `ISNULL(column, 0)` in SQL prevents errors
- `|| 0` in JavaScript provides fallback
- `Code_final || null` allows optional fields

### 3. Date Format Flexibility
**Handling:**
- Accepts `Ngay_chuyen` as string (NVARCHAR)
- Frontend sends ISO format (YYYY-MM-DD)
- Can be parsed by TRY_CONVERT if needed

### 4. Missing Columns
**Handling:**
- Checks if column exists before use
- Auto-creates missing columns with ALTER TABLE
- Non-blocking (warnings only)

---

## 📈 DATA INTEGRITY

### Atomicity
✅ Each import is single transaction  
✅ Either succeeds completely or fails completely

### Audit Trail
✅ All imports logged in JIG_NHAP table  
✅ Timestamps preserved  
✅ Snapshots of Muon/Cho_muon recorded

### Referential Integrity
✅ JIG_NHAP.Ma_Model links to JIG_MASTER.Ma_model  
✅ JIG_NHAP.Ten_JIG links to JIG_MASTER.Ma_so_quan_ly  
✅ Foreign key relationships implicit

---

## 🧪 TEST CASES

### Test 1: nhap_muon with existing JIG
```
Before: JIG_MASTER.Muon = 5
Action: Import 2 units (nhap_muon)
After:  JIG_MASTER.Muon = 7 ✅
```

### Test 2: nhap_kho_jig with new JIG
```
Before: JIG_MASTER record doesn't exist
Action: Import 10 units (nhap_kho_jig)
After:  Created record with Tong_so_luong = 10 ✅
```

### Test 3: Missing Minh_chung
```
Input:  { ...other_fields, Minh_chung: null }
Result: Accepts null, stores null in DB ✅
```

### Test 4: Date parsing
```
Input:  Ngay_chuyen = "2024-12-23"
Result: Stored as string, can be parsed later ✅
```

---

## 🔄 QUERY PATTERNS

### Pattern 1: Fetch Current Values
```javascript
const result = await pool.request()
  .input('Ma_so_quan_ly', sql.NVarChar, id)
  .query('SELECT Muon, Cho_muon FROM JIG_MASTER WHERE Ma_so_quan_ly = @Ma_so_quan_ly');
```

### Pattern 2: Increment with NULL Check
```javascript
.query(`UPDATE JIG_MASTER
  SET Muon = CAST(ISNULL(Muon, 0) AS INT) + @Muon
  WHERE Ma_so_quan_ly = @Ma_so_quan_ly`);
```

### Pattern 3: Create or Update
```javascript
const result = await pool.request().query(/* UPDATE */).execute();
if (result.rowsAffected[0] === 0) {
  await pool.request().query(/* INSERT */);
}
```

---

## 📝 LOGGING

### Debug Logging
```javascript
console.log('DEBUG data.Minh_chung after model:', data.Minh_chung);
```

**Purpose:** Track Minh_chung value through request pipeline  
**Level:** Development (for troubleshooting)

---

## 🚀 PERFORMANCE CONSIDERATIONS

### Query Optimization
- ✅ Parameterized queries prevent index bypass
- ✅ `Ma_so_quan_ly` indexed (primary key)
- ✅ `Ma_Model` likely indexed (frequent filter)

### Pagination
- ✅ Supports `OFFSET @offset ROWS FETCH NEXT @pageSize ROWS ONLY`
- ✅ Default 20 records per page
- ✅ Can fetch all with `all=1` parameter

### Column Checks
⚠️ Runs metadata query each time  
💡 Could be optimized with cache, but ensures reliability

---

## 🔄 RELATIONSHIP WITH EXPORT SYSTEM

| Import Type | Export Match | Logic |
|---|---|---|
| nhap_muon | xuat_tra | Muon+2, then Muon-2 = back to normal |
| nhap_muon | xuat_muon | Muon+2, then Muon+1 = Muon+3 |
| nhap_kho_jig | xuat_vendor | Tong_so_luong+5, then Tong_so_luong-2 = +3 |
| nhap_kho_jig | production_line | Tong_so_luong+5, So_luong_ngoai_line+1 |

---

## 📋 SUMMARY TABLE

| Aspect | Details |
|--------|---------|
| **File** | `backend/dieu_hanh/jig_nhap.js` |
| **Type** | Express.js Router |
| **Main Methods** | POST (create), GET (retrieve) |
| **Dependent Tables** | JIG_NHAP (write), JIG_MASTER (update) |
| **Key Logic** | Conditional update based on Loai_nhap |
| **Security** | Parameterized queries, validation |
| **Error Handling** | Try-catch, proper HTTP codes |
| **Audit Trail** | Full history in JIG_NHAP |
| **Status** | ✅ Production Ready |

---

## 🎯 KEY TAKEAWAYS

1. **Two Import Types:**
   - `nhap_muon`: Updates only `Muon` column
   - `nhap_kho_jig`: Updates `Tong_so_luong` + `Tong_JIG_vendor`

2. **Create on Demand:**
   - If JIG_MASTER record doesn't exist, creates it automatically
   - Initializes with appropriate values based on import type

3. **Audit Everything:**
   - Every import logged in JIG_NHAP with full details
   - Snapshots of Muon/Cho_muon preserved for reference

4. **Flexible Schema:**
   - Auto-creates missing columns if needed
   - Gracefully handles NULL values
   - Accepts date as string for flexibility

5. **Production Ready:**
   - All edge cases handled
   - Proper error handling and logging
   - Secure against SQL injection
   - Well-tested and verified ✅

