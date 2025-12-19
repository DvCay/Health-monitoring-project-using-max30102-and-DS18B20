# Cấu hình Firestore Security Rules

## Lỗi hiện tại
```
❌ Lỗi lưu Firestore: FirebaseError: Missing or insufficient permissions.
```

## Nguyên nhân
Firestore Security Rules mặc định chặn tất cả các request ghi dữ liệu từ client. Cần cấu hình rules để cho phép ứng dụng lưu dữ liệu.

## Path đang sử dụng
```
artifacts/web-monitor/users/{userId}/health_data/{docId}
```

Trong đó:
- `userId`: Anonymous User ID từ Firebase Auth (ví dụ: `C4mRjE1CjETHnBuZbOHkRHeQf313`)
- `docId`: Document ID theo format ngày (ví dụ: `2025-12-16`)

## Cách sửa

### Bước 1: Mở Firebase Console
1. Vào https://console.firebase.google.com
2. Chọn project của bạn
3. Vào **Firestore Database** → **Rules**

### Bước 2: Cập nhật Security Rules

**Option 1: Cho phép tất cả (chỉ dùng để test)** ⚠️
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /{document=**} {
      allow read, write: if true;
    }
  }
}
```

**Option 2: Chỉ cho phép user đã đăng nhập (khuyến nghị)** ✅
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Cho phép user đã đăng nhập (kể cả anonymous) đọc/ghi dữ liệu của chính họ
    match /artifacts/{appId}/users/{userId}/{document=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

**Option 3: Giới hạn chỉ cho phép ghi vào health_data (an toàn nhất)** 🔒
```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    // Chỉ cho phép user đã đăng nhập ghi vào health_data của chính họ
    match /artifacts/{appId}/users/{userId}/health_data/{docId} {
      allow read: if request.auth != null && request.auth.uid == userId;
      allow write: if request.auth != null 
                   && request.auth.uid == userId
                   && request.resource.data.keys().hasAll(['timestamp', 'bpm', 'spo2', 'temp']);
    }
  }
}
```

### Bước 3: Publish Rules
1. Nhấn **Publish** để áp dụng rules mới
2. Đợi vài giây để Firebase cập nhật
3. Test lại ứng dụng

## Kiểm tra
Sau khi cập nhật rules, bạn sẽ thấy trong console:
- ✅ `💾 Đang lưu: BPM=XX, SpO2=YY, Temp=ZZ`
- ✅ `✅ Lưu thành công!`

Thay vì:
- ❌ `❌ Lỗi lưu Firestore: FirebaseError: Missing or insufficient permissions.`

## Lưu ý
- **Option 1** không an toàn, chỉ dùng để test
- **Option 2** khuyến nghị cho môi trường development
- **Option 3** khuyến nghị cho môi trường production
- Sau khi deploy production, nên sử dụng Option 3 để bảo mật tốt nhất
