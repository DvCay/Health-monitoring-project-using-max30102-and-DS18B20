# 📘 HƯỚNG DẪN CHI TIẾT HỆ THỐNG GIÁM SÁT SỨC KHỎE

## Mục lục
1. [Tổng quan hệ thống](#1-tổng-quan-hệ-thống)
2. [Cấu trúc file và imports](#2-cấu-trúc-file-và-imports)
3. [Cấu hình Firebase](#3-cấu-hình-firebase)
4. [Biểu đồ và visualization](#4-biểu-đồ-và-visualization)
5. [CSS và styling](#5-css-và-styling)
6. [State management](#6-state-management)
7. [Firebase integration](#7-firebase-integration)
8. [Xử lý dữ liệu real-time](#8-xử-lý-dữ-liệu-real-time)
9. [Export Excel](#9-export-excel)
10. [Components và UI](#10-components-và-ui)

---

## 1. Tổng quan hệ thống

### 🎯 Mục đích
Hệ thống giám sát sức khỏe real-time cho phép:
- Đo và hiển thị các chỉ số sinh tồn: **Nhịp tim (BPM)**, **SpO₂**, **Nhiệt độ**
- Lưu trữ dữ liệu vào **Firebase Firestore**
- Xem lịch sử theo ngày
- Xuất báo cáo Excel chi tiết

### 📊 Luồng hoạt động
```
[Cảm biến] → [WebSocket/Giả lập] → [App.jsx] → [Firebase] → [Lịch sử/Excel]
                                         ↓
                                    [Biểu đồ Live]
```

---

## 2. Cấu trúc file và imports

### 📦 Dependencies chính

```jsx
import React, { useState, useEffect, useRef, useCallback } from 'react';
```
- **useState**: Quản lý state (trạng thái) của component
- **useEffect**: Xử lý side effects (Firebase, animations, WebSocket)
- **useRef**: Lưu trữ giá trị không trigger re-render
- **useCallback**: Memoize functions để tránh tạo lại không cần thiết

```jsx
import { Line } from 'react-chartjs-2';
import { Chart as ChartJS, ... } from 'chart.js';
```
- **react-chartjs-2**: Thư viện vẽ biểu đồ dựa trên Chart.js
- Hiển thị sóng PPG (nhịp tim), SpO₂ và nhiệt độ theo thời gian

```jsx
import * as XLSX from 'xlsx-js-style';
```
- Thư viện xuất file Excel với styling (màu sắc, merge cells, borders)

```jsx
import { initializeApp } from "firebase/app";
import { getAuth, ... } from "firebase/auth";
import { getFirestore, ... } from "firebase/firestore";
```
- **Firebase SDK**: Xác thực người dùng và lưu trữ dữ liệu

---

## 3. Cấu hình Firebase

### 🔥 Khởi tạo Firebase

```jsx
const firebaseConfig = {
  apiKey: "...",
  authDomain: "du-lieu-cb.firebaseapp.com",
  projectId: "du-lieu-cb",
  // ...
};

let app, db, auth;
try {
  app = initializeApp(firebaseConfig);
  db = getFirestore(app);
  auth = getAuth(app);
} catch (error) {
  console.error("❌ Lỗi khởi tạo Firebase:", error);
}
```

**Giải thích:**
- `initializeApp()`: Khởi tạo Firebase app với config
- `getFirestore()`: Lấy instance Firestore database
- `getAuth()`: Lấy instance Authentication
- `try-catch`: Xử lý lỗi nếu Firebase không kết nối được

### 🗂️ Cấu trúc dữ liệu Firestore

```
artifacts/
  └── web-monitor/
      └── users/
          └── {userId}/
              └── health_data/
                  └── {date_YYYYMMDD}/
                      └── records: [
                            {
                              timestamp: Timestamp,
                              bpm: 75.5,
                              spo2: 98.2,
                              temp: 36.5,
                              patientName: "Nguyễn Văn A",
                              patientAge: "25",
                              patientGender: "Nam"
                            },
                            ...
                          ]
```

**Tại sao dùng cấu trúc này?**
- Dễ query theo ngày
- Mỗi document chứa array records → giảm số lượng writes
- Mỗi user có dữ liệu riêng biệt

---

## 4. Biểu đồ và visualization

### 📈 Hàm `createInitialChartData(type)`

Tạo dữ liệu ban đầu cho 3 loại biểu đồ:

#### A. Biểu đồ BPM (Sóng mạch PPG)
```jsx
if (type === 'bpm') {
  const ppgDataset = {
    label: 'Sóng mạch (PPG)',
    data: Array(280).fill(0.5), // 280 điểm trên trục X
    borderColor: 'rgba(255, 77, 109, 0.95)', // Màu đỏ hồng
    tension: 0.4, // Độ cong của đường
    // ...
  };
}
```

**Giải thích:**
- **280 điểm**: Đủ để hiển thị ~6-8 đỉnh sóng mạch
- **PPG (Photoplethysmography)**: Đo thể tích máu qua ánh sáng
- Biểu đồ này hiển thị **biên độ sóng**, không phải số BPM

#### B. Biểu đồ SpO₂
```jsx
else if (type === 'spo2') {
  backgroundColor: (context) => {
    const gradient = ctx.createLinearGradient(...);
    gradient.addColorStop(0, 'rgba(220, 53, 69, 0.15)');    // Đỏ: <90%
    gradient.addColorStop(0.33, 'rgba(255, 193, 7, 0.15)'); // Vàng: 90-95%
    gradient.addColorStop(0.66, 'rgba(0, 180, 216, 0.15)'); // Xanh: 95-100%
    // ...
  },
  segment: {
    borderColor: (ctx) => {
      const value = ctx.p1.parsed.y;
      if (value >= 95) return 'rgba(40, 167, 69, 0.9)';   // Xanh lá: Tốt
      if (value >= 90) return 'rgba(255, 193, 7, 0.9)';   // Vàng: Cảnh báo
      return 'rgba(220, 53, 69, 0.9)';                     // Đỏ: Nguy hiểm
    }
  }
}
```

**Giải thích:**
- **Gradient động**: Màu nền thay đổi theo giá trị SpO₂
- **Segment coloring**: Mỗi đoạn đường có màu khác nhau dựa trên giá trị
- **Ngưỡng y học**:
  - `≥95%`: Bình thường (xanh lá)
  - `90-95%`: Cần theo dõi (vàng)
  - `<90%`: Nguy hiểm (đỏ)

#### C. Biểu đồ Nhiệt độ
```jsx
else if (type === 'temp') {
  segment: {
    borderColor: (ctx) => {
      const value = ctx.p1.parsed.y;
      if (value < 35) return 'rgba(0, 180, 216, 0.9)';     // Xanh: Hạ nhiệt
      if (value <= 37) return 'rgba(40, 167, 69, 0.9)';    // Xanh lá: Bình thường
      return 'rgba(220, 53, 69, 0.9)';                      // Đỏ: Sốt
    }
  }
}
```

**Ngưỡng nhiệt độ:**
- `<35°C`: Hạ nhiệt (nguy hiểm)
- `35-37°C`: Bình thường
- `>37°C`: Nhiệt độ cao (nguy hiểm)

**🆕 Thang đo nhiệt độ (cập nhật):**
- **Trước**: 20-42°C (range hẹp, chỉ thấy dải y học)
- **Sau**: **20-42°C** (range rộng, hiển thị cả nhiệt độ phòng)
- **Bước nhảy**: 2°C (thay vì 0.5°C) → dễ đọc hơn
- **Đường kẻ ngưỡng**:
  - Xanh dương tại 35°C (hạ nhiệt)
  - Xanh lá tại 37°C (bình thường)

### 🎨 Hàm `createChartOptions(isLoading, viewMode, activeChart)`

Cấu hình op20,  // 🆕 Thay đổi từ 34 → 20°C
      max: 42,
      stepSize: 2, // 🆕 Thay đổi từ 0.5 → 2°C
```jsx
const createChartOptions = (isLoading, viewMode, activeChart) => {
  let yOptions = {};
  
  if (activeChart === 'temp' && viewMode === 'live') {
    yOptions = { 
      min: 34, 
      max: 42, 
      grid: {
        color: (context) => {
          const value = context.tick.value;
          if (value === 37) return 'rgba(40, 167, 69, 0.4)'; // Đường kẻ tại 37°C
          if (value === 35) return 'rgba(0, 180, 216, 0.4)'; // Đường kẻ tại 35°C
          return 'rgba(0, 0, 0, 0.05)';
        }
      }
    };
  }
  // ...
}
```

**Giải thích:**
- **Dynamic grid lines**: Đường kẻ đậm hơn tại các ngưỡng quan trọng
- **Tooltip callbacks**: Hiển thị thông tin chi tiết khi hover
- **Responsive**: Tự động điều chỉnh theo kích thước màn hình

---

## 5. CSS và styling

### 🎨 GlobalStyles Component

```jsx
const GlobalStyles = () => (
  <style>{`
    :root {
      --bg-gradient: linear-gradient(135deg, #f0f4f8 0%, #e5eef5 100%);
      --card-bg-color: rgba(255, 255, 255, 0.8);
      --color-bpm: #ff4d6d;
      --color-spo2: #00b4d8;
      --color-temp: #ff9f1c;
    }
  `}</style>
);
```

**CSS Variables (Custom Properties):**
- Dễ maintain và thay đổi theme
- Tái sử dụng màu sắc nhất quán
- Hỗ trợ dark mode trong tương lai

### 🌊 Glass Morphism Effect

```css
.glass-card {
  background-color: rgba(255, 255, 255, 0.8);
  backdrop-filter: blur(12px);
  -webkit-backdrop-filter: blur(12px);
  border: 1px solid rgba(255, 255, 255, 0.9);
  box-shadow: 0 5px 15px rgba(100, 108, 120, 0.15);
}
```

**Giải thích:**
- **backdrop-filter**: Làm mờ nền phía sau card
- **Semi-transparent**: Tạo hiệu ứng kính trong suốt
- **Layered shadows**: Tạo độ sâu 3D

### 💓 Animations

```css
@keyframes pulseHeart {
  0%, 100% { 
    transform: scale(1);
    box-shadow: 0 0 0 12px rgba(255, 77, 109, 0.15);
  }
  50% { 
    transform: scale(1.05);
    box-shadow: 0 0 0 15px rgba(255, 77, 109, 0);
  }
}
```

**Animation cho icon tim:**
- Phóng to/thu nhỏ theo nhịp
- Shadow lan tỏa ra ngoài (ripple effect)
- Loop vô hạn với `cubic-bezier` smooth

---

## 6. State management

### 📊 Main States

```jsx
function App() {
  // Chỉ số sinh tồn
  const [bpm, setBpm] = useState(0);
  const [spo2, setSpo2] = useState(0);
  const [temperature, setTemperature] = useState(0);
  
  // Thông tin bệnh nhân (lưu vào LocalStorage)
  const [patientName, setPatientName] = useState(
    () => localStorage.getItem('patientName') || 'Nguyễn Văn A'
  );
  const [patientAge, setPatientAge] = useState(
    () => localStorage.getItem('patientAge') || ''
  );
  const [patientGender, setPatientGender] = useState(
    () => localStorage.getItem('patientGender') || 'Nam'
  );
  
  // Dữ liệu biểu đồ riêng biệt cho từng chỉ số
  const [bpmChartData, setBpmChartData] = useState(createInitialChartData('bpm'));
  const [spo2ChartData, setSpo2ChartData] = useState(createInitialChartData('spo2'));
  const [tempChartData, setTempChartData] = useState(createInitialChartData('temp'));
  
  // UI states
  const [activeChart, setActiveChart] = useState('bpm');
  const [viewMode, setViewMode] = useState('live'); // 'live' hoặc 'historical'
  
  // 🆕 Measurement states - Đo tất cả HOẶC đo từng chỉ số riêng biệt
  const [isMeasuring, setIsMeasuring] = useState(false);        // Đo tất cả
  const [isMeasuringBpm, setIsMeasuringBpm] = useState(false);  // Chỉ đo BPM
  const [isMeasuringSpo2, setIsMeasuringSpo2] = useState(false); // Chỉ đo SpO₂
  const [isMeasuringTemp, setIsMeasuringTemp] = useState(false); // Chỉ đo Nhiệt độ
  
  // Firebase states
  const [userId, setUserId] = useState(null);
  const [saveStatus, setSaveStatus] = useState('waiting');
  
  // ...
}
```

**Tại sao tách biệt 3 chart states?**
- Mỗi biểu đồ cập nhật độc lập
- BPM, SpO₂, Nhiệt độ có tốc độ thay đổi khác nhau
- Tránh re-render không cần thiết

**🆕 Tính năng đo riêng từng chỉ số:**
- **isMeasuring**: Đo tất cả 3 chỉ số cùng lúc (nút chính)
- **isMeasuringBpm**: Chỉ đo nhịp tim (nút đỏ trên card BPM)
- **isMeasuringSpo2**: Chỉ đo SpO₂ (nút xanh dương trên card SpO₂)
- **isMeasuringTemp**: Chỉ đo nhiệt độ (nút cam trên card Nhiệt độ)

**Ứng dụng thực tế:**
- Bệnh nhân chỉ cần đo nhiệt độ → Chỉ ấn nút cam
- Bệnh nhân chỉ cần đo nhịp tim + SpO₂ → Ấn 2 nút đỏ và xanh
- Đo tất cả → Ấn nút chính "▶️ Bắt đầu đo"

### 🔄 LocalStorage Persistence

```jsx
const [patientName, setPatientName] = useState(
  () => localStorage.getItem('patientName') || 'Nguyễn Văn A'
);
useEffect(() => localStorage.setItem('patientName', patientName), [patientName]);
```

**Cách hoạt động:**
1. **Lazy initialization**: Chỉ đọc localStorage lần đầu tiên
2. **useEffect sync**: Mỗi khi state thay đổi → lưu vào localStorage
3. **Kết quả**: Dữ liệu được giữ nguyên khi refresh trang

### 📍 Refs (không trigger re-render)

```jsx
const bpmRef = useRef(0);
const waveformPhaseRef = useRef(0);
const smoothSpo2Ref = useRef(0);
const lastMessageTsRef = useRef(Date.now());
```

**Khi nào dùng ref thay vì state?**
- Giá trị thay đổi liên tục (animation frames)
- Không cần re-render UI khi thay đổi
- Lưu trữ timers, intervals, DOM references

---

## 7. Firebase integration

### 🔐 Authentication

```jsx
useEffect(() => {
  const unsubscribe = onAuthStateChanged(auth, (user) => {
    if (user) {
      setUserId(user.uid);
    } else {
      setUserId(null);
    }
  });
  
  const signIn = async () => {
    await signInAnonymously(auth);
  };
  
  if (!auth.currentUser) {
    signIn();
  }
  
  return () => unsubscribe();
}, []);
```🆕 Kiểm tra điều kiện - Hỗ trợ đo riêng từng chỉ số
  if (!userId || !db || viewMode !== 'live') return;
  
  // Cho phép lưu nếu ĐANG ĐO (bất kỳ chỉ số nào)
  if (!isMeasuring && !isMeasuringBpm && !isMeasuringSpo2 && !isMeasuringTemp) return;
  
  // 🆕 Chỉ lưu khi có ít nhất 1 chỉ số > 0 (không yêu cầu đủ cả 3)
  if (bpmValue <= 0 && spo2Value <= 0 && tempValue <= 0) return;
  
  // Throttle: chỉ lưu mỗi 10 giây
  const now = Date.now();
  if (now - lastSaveTimeRef.current < 10000) return;
  
  lastSaveTimeRef.current = now;
  setSaveStatus('saving');
  
  const newRecord = {
    timestamp: Timestamp.now(),
    bpm: parseFloat(bpmValue.toFixed(1)),
    spo2: parseFloat(spo2Value.toFixed(1)),
    temp: parseFloat(tempValue.toFixed(1)),
    patientName: patientName.trim(),
    patientAge: patientAge.trim(),
    patientGender: patientGender
  };
  
  const docId = getDocId(new Date()); // Format: YYYYMMDD
  const docPath = `artifacts/${appId}/users/${userId}/health_data/${docId}`;
  const docRef = doc(db, docPath);
  
  try {
    await setDoc(docRef, {
      records: arrayUnion(newRecord)
    }, { merge: true });
    
    setSaveStatus('saved');
    setLastSavedTime(new Date());
    setRecordsSavedToday(prev => prev + 1);
  } catch (error) {
    console.error("Lỗi lưu Firestore:", error);
    setSaveStatus('error');
  }
}, [userId, db, viewMode, isMeasuring, isMeasuringBpm, isMeasuringSpo2, isMeasuringTemp
      records: arrayUnion(newRecord)
    }, { merge: true });
    
    setSaveStatus('saved');
    setLastSavedTime(new Date());
    setRecordsSavedToday(prev => prev + 1);
  } catch (error) {
    console.error("Lỗi lưu Firestore:", error);
    setSaveStatus('error');
  }
}, [userId, db, viewMode, isMeasuring, patientName, patientAge, patientGender]);
```

**Chi tiết quan trọng:**

1. **Throttle (10 giây)**: 
   - Tránh spam requests
   - Giảm chi phí Firebase
   - Đủ để theo dõi xu hướng

2. **arrayUnion**:

4. **🆕 Logic đo riêng (OR condition)**:
   - **Trước**: Yêu cầu `bpm > 0 AND spo2 > 0` mới lưu
   - **Sau**: Chấp nhận `bpm > 0 OR spo2 > 0 OR temp > 0`
   - **Lợi ích**: Có thể đo và lưu từng chỉ số độc lập
   
   **Ví dụ:**
   ```jsx
   // Đo chỉ nhiệt độ:
   // bpm = 0, spo2 = 0, temp = 36.5 → ✅ LƯU ĐƯỢC
   
   // Đo BPM + SpO₂:
   // bpm = 75, spo2 = 98, temp = 0 → ✅ LƯU ĐƯỢC
   
   // Chưa có dữ liệu:
   // bpm = 0, spo2 = 0, temp = 0 → ❌ KHÔNG LƯU
   ```
   - Thêm record vào array mà không ghi đè
   - Atomic operation (thread-safe)

3. **merge: true**:
   - Không xóa data cũ
   - Tạo document nếu chưa tồn tại

### 📅 Hàm `getDocId`

```jsx
const getDocId = (date) => {
  const y = date.getFullYear();
  const m = String(date.getMonth() + 1).padStart(2, '0');
  const d = String(date.getDate()).padStart(2, '0');
  return `${y}${m}${d}`; // Ví dụ: "20251218"
};
```

**Tại sao dùng format này?**
- Sortable: Tự động sắp xếp theo ngày
- Compact: Ngắn gọn
- Human-readable: Dễ debug

---

## 8. Xử lý dữ liệu real-time

### 🌊 Animation sóng PPG (BPM)

```jsx
useEffect(() => {
  if (viewMode !== 'live' || activeChart !== 'bpm') return;
  
  let animationFrameId;
  
  const animate = () => {
    const currentBpm = bpmRef.current;
    if (currentBpm === 0) {
      // Không có tín hiệu
      animationFrameId = requestAnimationFrame(animate);
      return;
    }
    
    // Tính amplitude dựa trên BPM
    const targetAmplitude = (currentBpm / 80) * 1.0;
    waveformAmplitudeRef.current += (targetAmplitude - waveformAmplitudeRef.current) * 0.02;
    
    // Tính tần số sóng: BPM → Hz
    const frequency = currentBpm / 60; // Hz
    waveformPhaseRef.current += frequency * (Math.PI * 2) / 60; // Radians per frame
    
    // Tạo sóng PPG
    const phase = waveformPhaseRef.current;
    const amplitude = waveformAmplitudeRef.current;
    
    // HRV (Heart Rate Variability): ±15%
    const hrv = 0.85 + Math.random() * 0.3;
    
    // Dạng sóng PPG thực tế
    let ppgValue = 0.5; // Baseline
    ppgValue += amplitude * hrv * Math.sin(phase); // Systole (tâm thu)
    ppgValue += amplitude * 0.3 * Math.sin(phase * 2 + 0.5); // Dicrotic notch
    ppgValue = Math.max(0.1, Math.min(2.4, ppgValue)); // Clamp
    
    // Cập nhật biểu đồ
    setBpmChartData(prev => {
      const wave = [...prev.datasets[0].data];
      wave.shift(); // Xóa điểm đầu
      wave.push(ppgValue); // Thêm điểm mới
      return { 
        ...prev, 
        datasets: [{ ...prev.datasets[0], data: wave }]
      };
    });
    
    animationFrameId = requestAnimationFrame(animate);
  };
  
  animationFrameId = requestAnimationFrame(animate);
  
  return () => cancelAnimationFrame(animationFrameId);
}, [viewMode, activeChart]);
```

**Giải thích chi tiết:**

1. **requestAnimationFrame**:
   - ~60 FPS (frames per second)
   - Synchronized với refresh rate của màn hình
   - Tạm dừng khi tab không active (tiết kiệm CPU)

2. **Dạng sóng PPG**:
   ```
        /\      ← Systole peak (đỉnh tâm thu)
       /  \
      /    \_   ← Dicrotic notch (sóng phản hồi)
     /       \
   __         \___ ← Diastole (tâm trương)
   ```

3. **HRV (Heart Rate Variability)**:
   - Biến thiên tự nhiên giữa các nhịp tim
   - Mô phỏng: ±15% amplitude
   - Phản ánh sức khỏe tim mạch

### 💨 Smooth interpolation SpO₂

```jsx
useEffect(() => {
  if (viewMode !== 'live') return;
  
  let animationId;
  let lastTime = performance.now();
  
  const animateSpo2 = (time) => {
    const deltaTime = (time - lastTime) / 1000;
    lastTime = time;
    
    // 1. LERP (Linear Interpolation) về giá trị target
    const target = targetSpo2Ref.current;
    const current = smoothSpo2Ref.current;
    const diff = target - current;
    
    if (Math.abs(diff) < 0.05) {
      smoothSpo2Ref.current = target;
    } else {
      smoothSpo2Ref.current += diff * 0.02; // 2% mỗi frame
    }
    
    // 2. Respiratory variation (biến thiên theo hô hấp)
    respiratoryPhaseRef.current += deltaTime * 0.3; // ~3-4s/chu kỳ
    const respiratoryVariation = 0.55 * Math.sin(respiratoryPhaseRef.current);
    
    // 3. Sensor noise
    const noise = (Math.random() - 0.5) * 0.15;
    
    // 4. Final value
    const finalSpo2 = smoothSpo2Ref.current + respiratoryVariation + noise;
    const clampedSpo2 = Math.max(80, Math.min(100, finalSpo2));
    
    // 5. Update chart
    setSpo2ChartData(prev => {
      const data = [...prev.datasets[0].data];
      data.shift();
      data.push(clampedSpo2);
      return { ...prev, datasets: [{ ...prev.datasets[0], data }] };
    });
    
    animationId = requestAnimationFrame(animateSpo2);
  };
  
  animationId = requestAnimationFrame(animateSpo2);
  return () => cancelAnimationFrame(animationId);
}, [viewMode]);
```

**Tại sao cần smooth interpolation?**
- SpO₂ từ cảm biến hay nhảy số
- Tạo animation mượt mà hơn
- Phản ánh biến thiên theo hô hấp (thực tế y học)

### 🌡️ Temperature animation

Tương tự SpO₂ nhưng:
- Thay đổi **CỰC CHẬM** (nhiệt độ cơ thể ít biến động)
- LERP 1% per frame (thay vì 2%)
- Biến thiên ±0.12°C (thay vì ±0.55%)
- Chu kỳ ~10 giây (thay vì ~4 giây)

---

## 9. Export Excel

### 📥 Hàm `exportToCSV`

```jsx
const exportToCSV = (records, filename = 'lich_su_benh_nhan.xlsx') => {
  if (records.length === 0) {
    aler🆕 Tạo data rows - Hiển thị "-" cho giá trị rỗng
  const dataRows = records.map(r => {
    const time = r.timestamp?.seconds
      ? new Date(r.timestamp.seconds * 1000).toLocaleString('vi-VN')
      : '';
    const patient = r.patientName || 'Không có tên';
    const gender = r.patientGender || 'N/A';
    const age = r.patientAge || 'N/A';
    
    // 🆕 Hiển thị "-" cho các chỉ số không đo (= 0 hoặc null)
    const bpmDisplay = (r.bpm && r.bpm > 0) ? r.bpm : '-';
    const spo2Display = (r.spo2 && r.spo2 > 0) ? r.spo2 : '-';
    const tempDisplay = (r.temp && r.temp > 0) ? r.temp : '-';
    
    // 🆕 Status chỉ đánh giá các chỉ số có giá trị
    const status = getStatusText(r.bpm, r.spo2, r.temp);
    
    return [patient, gender, age, time, bpmDisplay, spo2Display, tempDisplay
    ['BÁO CÁO GIÁM SÁT SỨC KHỎE'],
    ['Trung Tâm Giám Sát Sức Khỏe - Đại học Giao thông vận tải'],
    [`Ngày xuất: ${new Date().toLocaleString('vi-VN')}`],
    [`Tổng số bệnh nhân: ${patientCount} | Tổng số bản ghi: ${records.length}`],
    ['Bệnh nhân', 'Giới tính', 'Tuổi', 'Thời gian', 'Nhịp tim (BPM)', 'SpO2 (%)', 'Nhiệt độ (°C)', 'Trạng thái']
  ];
  
  // 3. Tạo data rows
  const dataRows = records.map(r => {
    const time = r.timestamp?.seconds
      ? new Date(r.timestamp.seconds * 1000).toLocaleString('vi-VN')
      : '';
    const patient = r.patientName || 'Không có tên';
    const gender = r.patientGender || 'N/A';
    const age = r.patientAge || 'N/A';
    const status = getStatusText(r.bpm, r.spo2, r.temp);
    return [patient, gender, age, time, r.bpm, r.spo2, r.temp, status];
  });
  
  // 4. Tạo worksheet
  const allData = [...infoData, ...dataRows];
  const ws = XLSX.utils.aoa_to_sheet(allData);
  
  // 5. Set column widths
  ws['!cols'] = [
    { wch: 20 }, // Bệnh nhân
    { wch: 12 }, // Giới tính
    { wch: 8 },  // Tuổi
    { wch: 25 }, // Thời gian
    { wch: 18 }, // BPM
    { wch: 15 }, // SpO2
    { wch: 20 }, // Nhiệt độ
    { wch: 18 }  // Trạng thái
  ];
  
  // 6. Merge cells
  const merges = [
    { s: { r: 0, c: 0 }, e: { r: 0, c: 7 } }, // Title
    { s: { r: 1, c: 0 }, e: { r: 1, c: 7 } }, // Subtitle
    { s: { r: 2, c: 0 }, e: { r: 2, c: 7 } }, // Date
    { s: { r: 3, c: 0 }, e: { r: 3, c: 7 } }  // Summary
  ];
  ws['!merges'] = merges;
  
  // 7. Apply styles
  // ... (styling code) ...
  
  // 8. Create workbook và download
  const wb = XLSX.utils.book_new();
  XLSX.utils.book_append_sheet(wb, ws, 'Dữ liệu sức khỏe');
  XLSX.writeFile(wb, filename);
};
```

### 🎨 Excel Styling

```jsx
// Title style
const titleStyle = {
  font: { 
    name: 'Arial', 
    sz: 18, 
    bold: true, 
    color: { rgb: "FFFFFF" } 
  },
  fill: { 
    fgColor: { rgb: "0066CC" } 
  },
  alignment: { 
    horizontal: "center", 
    vertical: "center" 
  },
  border: borderStyle
};

// Header style (hàng tiêu đề cột)
const headerStyle = {
  font: { 
    sz: 12, 
    bold: true, 
    color: { rgb: "FFFFFF" } 
  },
  fill: { 
    fgColor: { rgb: "4472C4" } 
  },
  alignment: { 
    horizontal: "center", 
    vertical: "center", 
    wrapText: true 
  },
  border: borderStyle
};

// Data style với màu xen kẽ
const dataStyle = {
  font: { sz: 11, color: { rgb: "000000" } },
  fill: { fgColor: { rgb: "FFFFFF" } }, // Trắng
  alignment: { horizontal: "center", vertical: "center" },
  border: borderStyle
};

const dataStyleAlt = {
  ...dataStyle,
  fill: { fgColor: { rgb: "F2F2F2" } } // Xám nhạt
};

// Highlight theo trạng thái
const criticalStyle = {
  ...dataStyle,
  fill: { fgColor: { rgb: "FFE6E6" } }, // Đỏ nhạt
  font: { ...dataStyle.font, bold: true, color: { rgb: "C00000" } }
};

const warningStyle = {
  ...dataStyle,
  fill: { fgColor: { rgb: "FFF9E6" } }, // Vàng nhạt
  font: { ...dataStyle.font, color: { rgb: "C65D00" } }
};
```

**Apply styles:**
```jsx
for (let i = 0; i < dataRows.length; i++) {
  const rowNum = 6 + i;
  const record = records[i];
  const status = getRowStatus(record.bpm, record.spo2, record.temp);
  
  let rowStyle;
  if (status === 'critical') {
    rowStyle = criticalStyle;
  } else if (status === 'warning') {
    rowStyle = warningStyle;
  } else {
    rowStyle = i % 2 === 0 ? dataStyle : dataStyleAlt;
  }
  
  ['A', 'B', 'C', 'D', 'E', 'F', 'G', 'H'].forEach(col => {
    const cellRef = `${col}${rowNum}`;
    if (ws[cellRef]) {
      ws[cellRef].s = rowStyle;
    }
  });
}
```

---

## 10. Components và UI

### 🫀 HeartRateCircle Component

```jsx
const HeartRateCircle = ({ bpm }) => (
  <div className="heart-rate-circle">
    <div className="heart-rate-inner-circle">
      <HeartIcon />
    </div>
  </div>
);
```

**CSS Animation:**
```css
.heart-rate-circle {
  animation: pulseHeart 1s cubic-bezier(0.4, 0, 0.6, 1) infinite;
}

.heart-rate-inner-circle svg {
  animation: beatHeart 1s ease-in-out infinite;
}
```

### 📅 Calendar Component

```jsx
const Calendar = ({ selectedDate, onDateSelect }) => {
  const [currentMonth, setCurrentMonth] = useState(new Date());
  
  // Tạo grid ngày trong tháng
  const daysInMonth = new Date(
    currentMonth.getFullYear(), 
    currentMonth.getMonth() + 1, 
    0
  ).getDate();
  
  const firstDayOfMonth = new Date(
    currentMonth.getFullYear(), 
    currentMonth.getMonth(), 
    1
  ).getDay();
  
  // Render calendar grid...
};
```

### 🎯 Metric Cards

```jsx
<div
  className={`metric-card glass-card ${bpmStatus.className} ${activeChart === 'bpm' ? 'active-chart' : ''}`}
  onClick={() => handleChartSelect('bpm')}
>
  <div className="card-header">
    <div className="header-text">
      <h2>Nhịp tim (BPM)</h2>
      <span className="sensor-name">(MAX30102)</span>
    </div>
    <div className="metric-icon"><HeartIcon /></div>
  
  {/* 🆕 Nút đo riêng từng chỉ số */}
  {viewMode === 'live' && (
    <div className="individual-measure-btn-container">
      {!isMeasuringBpm ? (
        <button 
          className="individual-measure-btn bpm-btn"
          onClick={(e) => { e.stopPropagation(); startMeasuringBpm(); }}
        >
          ▶️ Đo riêng BPM
        </button>
      ) : (
        <button 
          className="individual-measure-btn bpm-btn stop"
          onClick={(e) => { e.stopPropagation(); stopMeasuringBpm(); }}
        >
          ⏹️ Dừng BPM
        </button>
      )}
    </div>
  )}
</div>
```

**Dynamic className:**
- `status-normal`: Xanh lá
- `status-warning`: Vàng
- `status-danger`: Đỏ (có animation pulse)
- `active-chart`: Highlight card đang được chọn

**🆕 Individual Measurement Buttons:**
- **BPM**: Nút đỏ `#ff4d6d`
- **SpO₂**: Nút xanh dương `#00b4d8`
- **Temp**: Nút cam `#ffa500`
- **Vị trí**: Dưới mỗi metric card
- **Logic**: `e.stopPropagation()` để không trigger `onClick` của cardus.text}</span>
  </div>
</div>
```

**Dynamic className:**
- `status-normal`: Xanh lá
- `status-warning`: Vàng
- `status-danger`: Đỏ (có animation pulse)
- `active-chart`: Highlight card đang được chọn

### 🔔 Alert Banner

```jsx
{showAlert && (
  <div className="alert-banner">
    <span className="alert-icon">⚠️</span>
    <span>{alertMessage}</span>
  </div>
)}
```

**Trigger alert:**
```jsx
const checkAndShowAlert = useCallback((b, s, t) => {
  if (viewMode !== 'live') return;
  
  let alerts = [];
  if (b > 100 || b < 60) alerts.push(`Nhịp tim bất thường: ${b} BPM`);
  if (s < 95) alerts.push(`SpO₂ thấp: ${s}%`);
  if (t < 35) alerts.push(`Hạ nhiệt: ${t}°C`);
  if (t > 37) alerts.push(`Nhiệt độ cao: ${t}°C`);
  
  if (alerts.length > 0) {
    setAlertMessage(alerts.join(' | '));
    setShowAlert(true);
    setTimeout(() => setShowAlert(false), 5000);
  }
}, [viewMode]);
```

---

## 🔧 Debugging Tips

### Console logs quan trọng

```jsx
console.log("🔍 Firebase useEffect chạy");
console.log("✅ Đã đăng nhập, User ID:", user.uid);
console.log("💾 Đang lưu: BPM=", bpmValue, "SpO2=", spo2Value);
console.log("🎬 Bắt đầu đo cho bệnh nhân:", patientName);
```

### Check Firebase connection

1. Mở DevTools → Console
2. Tìm log: `"✅ Đã khởi tạo Firebase thành công"`
3. Kiểm tra `userId !== null`

### Check data saving
� Tính năng đặc biệt: Đo riêng từng chỉ số

### Tại sao cần tính năng này?

**Vấn đề trước đây:**
- Phải đo tất cả 3 chỉ số cùng lúc
- Nếu chỉ cần đo nhiệt độ → vẫn phải kết nối sensor BPM/SpO₂
- Lãng phí thời gian và tài nguyên

**Giải pháp:**
- ✅ 3 nút đo riêng biệt trên mỗi card
- ✅ Có thể đo 1, 2, hoặc cả 3 chỉ số
- ✅ Firebase lưu bất kỳ chỉ số nào > 0

### Implementation chi tiết

#### 1. States (Lines 1638-1640 trong App.jsx)
```jsx
const [isMeasuringBpm, setIsMeasuringBpm] = useState(false);
const [isMeasuringSpo2, setIsMeasuringSpo2] = useState(false);
const [isMeasuringTemp, setIsMeasuringTemp] = useState(false);
```

#### 2. Start/Stop Functions (Lines 2407-2471)
```jsx
const startMeasuringBpm = () => {
  if (!patientName.trim()) {
    alert('Vui lòng nhập tên bệnh nhân trước!');
    return;
  }
  if (isMeasuring) setIsMeasuring(false); // Tắt đo tổng
  setIsMeasuringBpm(true);
  console.log(`💓 Bắt đầu đo BPM cho bệnh nhân: ${patientName}`);
};

const stopMeasuringBpm = () => {
  setIsMeasuringBpm(false);
  if (!isMeasuringSpo2 && !isMeasuringTemp && !isMeasuring) {
    setMeasurementStartTime(null); // Reset nếu không còn đo gì
  }
};
```

#### 3. Firebase Save Logic (Lines 1739-1740)
```jsx
// Trước: if (!isMeasuring) return;
// Sau:
if (!isMeasuring && !isMeasuringBpm && !isMeasuringSpo2 && !isMeasuringTemp) return;

// Trước: if (bpmValue <= 0 || spo2Value <= 0) return;
// Sau:
if (bpmValue <= 0 && spo2Value <= 0 && tempValue <= 0) return;
```

#### 4. Dynamic Status Display (Lines 2833-2851)
```jsx
<div className="status-card glass-card">
  <h3>⚙️ Trạng thái hệ thống</h3>
  <div className="status-row">
    <span className="status-label">Chế độ:</span>
    <span className="status-value">
      {(() => {
        if (isMeasuring) return "🔴 Đang đo: 💓 BPM + 🫁 SpO₂ + 🌡️ Nhiệt độ";
        if (isMeasuringBpm && isMeasuringSpo2 && isMeasuringTemp) 
          return "🔴 Đang đo: 💓 BPM + 🫁 SpO₂ + 🌡️ Nhiệt độ";
        if (isMeasuringBpm && isMeasuringSpo2) 
          return "🔴 Đang đo: 💓 BPM + 🫁 SpO₂";
        if (isMeasuringBpm) return "🔴 Đang đo: 💓 BPM";
        if (isMeasuringSpo2) return "🔴 Đang đo: 🫁 SpO₂";
        if (isMeasuringTemp) return "🔴 Đang đo: 🌡️ Nhiệt độ";
        return "⚪ Chưa đo";
      })()}
    </span>
  </div>
</div>
```

#### 5. Excel Export với "-" (Lines 1305-1307)
```jsx
const bpmDisplay = (r.bpm && r.bpm > 0) ? r.bpm : '-';
const spo2Display = (r.spo2 && r.spo2 > 0) ? r.spo2 : '-';
const tempDisplay = (r.temp && r.temp > 0) ? r.temp : '-';
```

**Kết quả Excel:**
```
| Bệnh nhân | Nhịp tim | SpO2 | Nhiệt độ |
|-----------|----------|------|----------|
| Nguyễn A  | 75       | 98   | 36.5     | ← Đo tất cả
| Trần B    | -        | -    | 36.2     | ← Chỉ đo nhiệt độ
| Lê C      | 82       | 96   | -        | ← Chỉ đo BPM + SpO₂
```

---

## 🎓 Kết luận

Hệ thống này kết hợp:
- ✅ Real-time data visualization
- ✅ Cloud storage (Firebase)
- ✅ Professional UI/UX
- ✅ Medical-grade thresholds
- ✅ Excel reporting
- ✅ **🆕 Individual measurement capability**

**Kiến thức cần để hiểu code:**
1. React Hooks (useState, useEffect, useRef, useCallback)
2. JavaScript async/await
3. Chart.js configuration
4. Firebase API
5. CSS animations
6. Browser APIs (requestAnimationFrame, localStorage)

**🆕 Cập nhật mới nhất (Tháng 12/2025):**
1. ✅ Tính năng đo riêng từng chỉ số (BPM, SpO₂, Nhiệt độ)
2. ✅ Firebase save logic linh hoạt (OR condition)
3. ✅ Excel export hiển thị "-" cho giá trị trống
4. ✅ Dynamic status display với emoji
5. ✅ Temperature chart scale 20-42°C (rộng hơn
### Technologies sử dụng

- **React**: https://react.dev/
- **Chart.js**: https://www.chartjs.org/docs/latest/
- **Firebase**: https://firebase.google.com/docs
- **SheetJS (xlsx)**: https://docs.sheetjs.com/

### Medical thresholds

- **BPM**: 60-100 bình thường (người lớn)
- **SpO₂**: ≥95% bình thường, <90% hypoxemia
- **Temperature**: 35-37°C bình thường

### Performance

- **requestAnimationFrame**: 60 FPS (16.67ms/frame)
- **Firebase throttle**: 10s giữa các writes
- **Chart data points**: 280 (tối ưu memory & render)

---

## 🎓 Kết luận

Hệ thống này kết hợp:
- ✅ Real-time data visualization
- ✅ Cloud storage (Firebase)
- ✅ Professional UI/UX
- ✅ Medical-grade thresholds
- ✅ Excel reporting

**Kiến thức cần để hiểu code:**
1. React Hooks (useState, useEffect, useRef, useCallback)
2. JavaScript async/await
3. Chart.js configuration
4. Firebase API
5. CSS animations
6. Browser APIs (requestAnimationFrame, localStorage)

**Next steps để mở rộng:**
- Thêm WebSocket kết nối cảm biến thật
- Implement user authentication (email/password)
- Thêm push notifications khi có cảnh báo
- Tạo admin dashboard để quản lý nhiều bệnh nhân
- Export PDF reports
- Multi-language support

---

📧 **Liên hệ:**
- Đặng Văn Cấy: 211411929
- Đỗ Đức Duy: 211401626

🏫 **Trường:** Đại học Giao thông vận tải
