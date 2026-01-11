# 📥 NHẬP KHO - LOGIC REVIEW

**Date:** 09/01/2026  
**Status:** Active ✅  
**Last Updated:** After Cache Fix

---

## 🎯 HIGH-LEVEL FLOW

```
User Fill Form (NhapKhoForm.tsx)
    ↓
Select Purpose (Mục đích nhập)
    ├─ Nhập Line (OK/NG)
    ├─ Nhập CRO
    ├─ Nhập cải tiến
    └─ Nhập mượn
    ↓
POST /api/nhap-kho/ghi-nhan
    ↓
Backend Process (nhap_kho.js)
    ├─ 1. Validate required fields ✓
    ├─ 2. Insert transaction record (GIAO_DICH_NHAP_KHO) ✓
    ├─ 3. Update master inventory (KHO) - Add to Kho_OK & Tong_ton ✓
    ├─ 4. Update bin location (BIN_VI_TRI) - Add to OK & Stock ✓
    └─ 5. Log activity (Nhật ký) ✓
    ↓
Response: { message: "Ghi nhận nhập kho thành công" }
```

---

## 📋 FORM STRUCTURE (Frontend: NhapKhoForm.tsx)

### Step 1: Search Material
- Input: SS_Code (Mã SS)
- Auto-complete from `/api/vat-tu` endpoint
- Shows suggestions with Item name, Vendor code, Model

### Step 2: Fill Details
**Purpose Selection (Mục đích nhập):**
```jsx
computeLoaiNhap() returns:
├─ "nhap_line_ok"     ← Nhập Line (OK)
├─ "nhap_line_ng"     ← Nhập Line (NG)
├─ "nhap_cai_tien"    ← Nhập Cải tiến
├─ "nhap_cro"         ← Nhập CRO
└─ "nhap_muon"        ← Nhập Mượn
```

**Fields:**
- `Loai_Giao_Dich`: Type of import (nhap_line_ok, nhap_cro, etc)
- `So_Luong`: Quantity to import
- `Vi_Tri`: Warehouse location (Bin code)
- `Nguoi_Thuc_Hien`: Person receiving
- `Noi_Nhan`: Receiving location (Equipment, Factory, etc)
- `Ngay_Nhap`: Date of import
- `UoM`: Unit of measurement
- `Ghi_Chu`: Notes
- `Hang_muc`: Category (optional)
- `Hang_muc_nho`: Sub-category (optional, auto-filled "CRO" if nhap_CRO)

---

## 🔧 BACKEND PROCESSING (nhap_kho.js)

### Endpoint: POST /api/nhap-kho/ghi-nhan

#### STEP 1: Validate Required Fields
```javascript
Required:
✓ SS_Code        - Material code
✓ Vendor_code    - Supplier code
✓ Item           - Material name
✓ So_Luong       - Quantity (must be > 0)
✓ Vi_Tri         - Warehouse location

Optional but used:
- Nguoi_Thuc_Hien
- Noi_Nhan
- Ngay_Giao_Dich
- Loai_Giao_Dich
- UoM
- Ghi_Chu
- MaNV
- Action
- Hang_muc
- Hang_muc_nho
```

#### STEP 2: Insert Transaction Record
**Table:** GIAO_DICH_NHAP_KHO
```sql
INSERT INTO GIAO_DICH_NHAP_KHO (
  Vendor_code, SS_Code, Item, So_Luong, vi_Tri,
  Nguoi_Thuc_Hien, Noi_Nhan, NgayGiaoDich,
  Loai_Giao_Dich, UoM, Ghi_Chu, MaNV, Action,
  Hang_muc, Hang_muc_nho
)
VALUES (...)
```

**Purpose:** Record audit trail of all import transactions

#### STEP 3: Update Master Inventory (KHO table)

**Check if material exists:**
```sql
SELECT * FROM KHO 
WHERE SS_Code = @SS_Code AND Vendor_code = @Vendor_code
```

**If EXISTS → UPDATE:**
```sql
UPDATE KHO 
SET Kho_OK = ISNULL(Kho_OK, 0) + @So_Luong,
    tong_ton = ISNULL(tong_ton, 0) + @So_Luong
WHERE SS_Code = @SS_Code AND Vendor_code = @Vendor_code
```

**If NOT EXISTS → INSERT new:**
```sql
INSERT INTO KHO (SS_Code, Vendor_code, Item, Kho_OK, tong_ton)
VALUES (@SS_Code, @Vendor_code, @Item, @So_Luong, @So_Luong)
```

**Effect:** Increases both OK stock and total stock by import quantity

#### STEP 4: Update Bin Location (BIN_VI_TRI table)

**Purpose:** Track stock in specific warehouse bins

**Check if bin exists for this material:**
```sql
SELECT * FROM BIN_VI_TRI 
WHERE SS_code = @SS_Code 
  AND Vendor_code = @Vendor_code 
  AND Bin_Code = @Bin_Code
```

**If EXISTS → UPDATE:**
```sql
UPDATE BIN_VI_TRI 
SET OK = ISNULL(OK, 0) + @So_Luong,
    Stock = ISNULL(Stock, 0) + @So_Luong
WHERE SS_code = @SS_Code 
  AND Vendor_code = @Vendor_code 
  AND Bin_Code = @Bin_Code
```

**If NOT EXISTS → INSERT new:**
```sql
INSERT INTO BIN_VI_TRI (
  Rack, Layout, Bin, Bin_Code, SS_code, Vendor_code,
  Item, OK, Stock, Trang_thai_Bin
)
VALUES (
  @Rack, @Layout, @Bin, @Bin_Code, @SS_code, @Vendor_code,
  @Item, @So_Luong, @So_Luong, 'Occupied'
)
```

**Bin Code Parsing:**
```javascript
// Input: "A-1-5" or "A-5"
parseBinCode("A-1-5") → { rack: "A", layout: "1", bin: "5" }
parseBinCode("A-5")   → { rack: "A", layout: null, bin: "5" }
```

#### STEP 5: Log Activity (ghiNhatKy)

Records import in activity log with:
- `loaiHoatDong`: "NHAP_KHO"
- `module`: "Nhập kho"
- `soLuong`: Quantity imported
- `moTa`: Description including location & type
- `duLieuMoi`: New data fields

---

## 📊 DATA TABLES AFFECTED

### 1. GIAO_DICH_NHAP_KHO (Transaction Log)
**Records:** Import transactions  
**Updated:** ✓ INSERT new record  
**Purpose:** Audit trail

### 2. KHO (Master Inventory)
**Columns Updated:**
- `Kho_OK`: Stock in warehouse (OK condition)
- `tong_ton`: Total stock

**Updated:** ✓ UPDATE existing OR INSERT new  
**Purpose:** Master stock count

### 3. BIN_VI_TRI (Bin Locations)
**Columns Updated:**
- `OK`: Quantity in OK condition
- `Stock`: Total quantity in bin

**Updated:** ✓ UPDATE existing OR INSERT new  
**Purpose:** Track stock in specific locations

### 4. LOG_NGHIEP_VU (Activity Log)
**Updated:** ✓ INSERT new record  
**Purpose:** Audit trail for business operations

---

## ⚡ IMPORT TYPES EXPLAINED

### Type 1: Nhập Line (nhap_line_ok / nhap_line_ng)
```
Purpose: Materials from production line back to warehouse
Loai_Giao_Dich: "nhap_line_ok" or "nhap_line_ng"
Vi_Tri: Auto-set to "San xuat" (Production)
Khu_vuc_nhan: "Equipment"
Effect: Increases Kho_OK and Tong_ton
```

### Type 2: Nhập CRO (nhap_cro)
```
Purpose: Import from CRO (Change Request Order)
Loai_Giao_Dich: "nhap_cro"
Hang_muc_nho: Auto-filled as "CRO"
Effect: Increases Kho_OK and Tong_ton
```

### Type 3: Nhập Cải tiến (nhap_cai_tien)
```
Purpose: Improved/refurbished materials
Loai_Giao_Dich: "nhap_cai_tien"
Effect: Increases Kho_OK and Tong_ton
```

### Type 4: Nhập Mượn (nhap_muon)
```
Purpose: Borrow materials from supplier
Loai_Giao_Dich: "nhap_muon"
Note: Currently same logic as others, increases Kho_OK
TODO: May need separate Muon column if borrowing should be tracked separately
```

---

## 🔍 POTENTIAL ISSUES & OBSERVATIONS

### ⚠️ Issue 1: Multiple Import Types Same Logic
**Current:** All import types (nhap_line_ok, nhap_cro, nhap_cai_tien, nhap_muon) update KHO the same way
**Risk:** Cannot distinguish between import sources in reporting
**Solution:** Consider adding `Loai_Giao_Dich` filter in reports

### ⚠️ Issue 2: No Validation of Bin Format
**Current:** Bin code parsed but no format validation
**Risk:** Invalid bin codes may be accepted
**Suggestion:** Add regex validation for bin format (e.g., "A-1-5" or "A-5")

### ⚠️ Issue 3: Nguoi_Thuc_Hien Not Required
**Current:** Can import without specifying who received
**Risk:** Cannot track responsibility for received materials
**Suggestion:** Make `Nguoi_Thuc_Hien` mandatory in validation

### ⚠️ Issue 4: Double-Counting in KHO vs BIN_VI_TRI
**Current:** Same quantity added to both KHO and BIN_VI_TRI
**Risk:** If reporting queries both tables, may get double-counts
**Solution:** Use KHO as primary, BIN_VI_TRI as location breakdown only

### ⚠️ Issue 5: No Stock Level Validation
**Current:** Can import unlimited quantity
**Risk:** No sanity check for unrealistic quantities
**Suggestion:** Add warning if quantity > expected max

---

## 📱 FRONTEND FORM FLOW (NhapKhoForm.tsx)

```
┌─────────────────────────────────┐
│  START: Enter Material Code     │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│  STEP 1: Search Material        │
│  - Type SS_Code                 │
│  - Get suggestions              │
│  - Select or Continue (new)     │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│  STEP 2: Fill Import Details    │
│  - Select Mục đích nhập         │
│    ├─ Nhập Line                 │
│    ├─ Nhập CRO                  │
│    ├─ Nhập Cải tiến             │
│    └─ Nhập Mượn                 │
│  - Enter Số lượng               │
│  - Select Vi_Tri (Bin)          │
│  - Select Người thực hiện       │
│  - Enter Date                   │
│  - Optional: Ghi chú            │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│  SUBMIT: POST /api/nhap-kho/ghi │
│  -nhan                          │
└────────────┬────────────────────┘
             │
             ↓
┌─────────────────────────────────┐
│  SUCCESS: Show confirmation     │
│  - Display inventory change     │
│  - Reset form for next input    │
└─────────────────────────────────┘
```

---

## 🔄 BULK IMPORT MODE (NhapKhoForm.tsx)

**Alternative flow for importing multiple materials:**

```
1. Paste material codes (one per line)
2. System searches and loads all materials
3. Display table with:
   - Material info (SS_Code, Item, Vendor)
   - Current bin location
   - Quantity field for each
4. Allow edit quantities
5. Submit all together to backend
6. Backend processes each one individually
```

**Advantage:** Faster than one-by-one entry

---

## 💾 DATA PERSISTENCE

### On Success:
✅ GIAO_DICH_NHAP_KHO record created  
✅ KHO updated (Kho_OK, tong_ton)  
✅ BIN_VI_TRI updated (OK, Stock)  
✅ LOG_NGHIEP_VU record created  

### On Error:
❌ Transaction rolled back  
❌ User sees error message with details  
❌ No partial updates  

---

## 🎨 UI/UX OBSERVATIONS

### Strengths:
✅ Clear step-by-step process  
✅ Auto-suggestions reduce typing  
✅ Import type clearly explained  
✅ Bulk import option for efficiency  
✅ Toast notifications for feedback  

### Potential Improvements:
- Add quantity validation (min/max warning)
- Show current stock before importing
- Add barcode scanning option
- Confirm before submit if quantity seems unusual
- Show import history sidebar

---

## 📈 REPORTING QUERIES

### Current Transaction History
```
GET /api/nhap-kho/lich-su
Returns: TOP 200 GIAO_DICH_NHAP_KHO sorted by date DESC
```

### Suggested Reports Needed
1. **Import by Type** - Group by Loai_Giao_Dich
2. **Import by Location** - Group by Vi_Tri
3. **Import by Supplier** - Group by Vendor_code
4. **Daily Import Summary** - Total by date
5. **Material Import History** - For specific material

---

## ✅ VALIDATION CHECKLIST

- [x] Required fields validated on submit
- [x] Database inserts with proper error handling
- [x] Bin code parsing implemented
- [ ] Quantity validation (unrealistic values)
- [ ] Bin format validation
- [ ] Người_Thuc_Hien mandatory check
- [ ] Duplicate import check (prevent duplicates in same transaction)

---

## 🚀 NEXT STEPS (RECOMMENDATIONS)

1. **Add Input Validation**
   - Quantity must be > 0 and < 10000
   - Bin code format validation
   - Date must not be in future

2. **Enhance Error Messages**
   - Show which field failed
   - Suggest corrections

3. **Add Confirmation Dialog**
   - Show summary before submit
   - Allow user to cancel

4. **Implement Stock Preview**
   - Show current stock before import
   - Show post-import stock after action

5. **Cache Invalidation** ✅ DONE
   - Service Worker doesn't cache `/api/nhap-kho` endpoints
   - Cache version bumped to v2

---

## 📞 CONTACT
For questions about logic implementation, contact development team.
