# QUICK REFERENCE - JIG Export Logic

## TÓBM TẮT CÁC THAY ĐỔI

### 📊 BEFORE (SAI)
```
xuat_tra (return): CHO_MUON ➕ (SAULT)
xuat_muon: ❌ KHÔNG CÓ (MISSING)
xuat_vendor: TONG_SO_LUONG ➖ (OK)
```

### ✅ AFTER (ĐÚNG)
```
xuat_tra (return):  MUON ➖ (FIXED)
xuat_muon (borrow): MUON ➕ (ADDED)
xuat_vendor:        TONG_SO_LUONG ➖ (OK)
```

---

## 💾 FILES CHANGED

| File | Change | Status |
|------|--------|--------|
| `backend/dieu_hanh/jig_inventory.js` | Logic fixes for xuat_tra & xuat_muon | ✅ Done |
| `src/pages/JigShipping.tsx` | Add xuat_muon option + update condition | ✅ Done |

---

## 🔄 FLOW DIAGRAM

```
IMPORT SIDE (JIG_NHAP)
├── nhap_muon (borrow) → Muon ➕
├── nhap_kho_jig (stock) → Tong_so_luong ➕
└── Display: Report shows import type

EXPORT SIDE (JIG_XUAT) - UPDATED
├── xuat_tra (return) → Muon ➖ (FIXED)
├── xuat_muon (borrow) → Muon ➕ (ADDED)
├── xuat_vendor (sell) → Tong_so_luong ➖
├── production_line → So_luong_ngoai_line ➕
└── disposal/others → Tong_so_luong ➖

DATABASE (JIG_MASTER)
├── Muon: tracks borrowed items
├── Tong_so_luong: total inventory
├── Tong_JIG_vendor: vendor stock
└── So_luong_ngoai_line: items on line
```

---

## ⚙️ TECHNICAL NOTES

### Code Changes
1. **isXuatTra** logic: `Cho_muon += So_luong` → `Muon -= So_luong`
2. **isXuatMuon** logic: NEW condition added
3. **Else** branch: Remains for vendor/disposal

### Edge Cases Handled
- ✅ `Math.max(0, ...)` to prevent negative Muon
- ✅ `ISNULL(Muon, 0)` to handle null values
- ✅ Conditional field display based on loaiXuat type

### Data Validation
```javascript
const newMuon = Math.max(0, currentMuon - So_luong);
// Result: Muon can't go below 0
```

---

## 🎯 USER-FACING CHANGES

### Frontend Dropdown
Before:
```
- Xuất trả
- Xuất ra vendor đối tác
- Thanh lý
- Xuất ra line SX
```

After:
```
- Xuất trả (Trừ mượn) ← Clearer description
- Xuất mượn (Cộng mượn) ← NEW OPTION
- Xuất ra vendor đối tác
- Thanh lý
- Xuất ra line SX
```

---

## 📈 DATA INTEGRITY

All changes respect:
- ✅ No orphaned records
- ✅ No duplicate keys
- ✅ Referential integrity maintained
- ✅ Backward compatible with existing exports

---

## 🚀 DEPLOYMENT CHECKLIST

- [ ] Backup database
- [ ] Replace jig_inventory.js on backend
- [ ] Rebuild frontend (`npm run build`)
- [ ] Restart backend server
- [ ] Clear browser cache
- [ ] Test new xuat_muon option
- [ ] Verify Muon values in reports

---

**Last Updated:** 23/12/2024
**Version:** 1.0 - Ready for Production
