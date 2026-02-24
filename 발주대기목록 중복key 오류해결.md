## **📋 문제 상황**

### **React 에러 메시지**

```
Encountered two children with the same key, `RFQ-2026-0001`.
Keys should be unique so that components maintain their identity across updates.

```

### **발생 원인**

- 데이터베이스에 `RFQ-2026-0001`에 **2개의 품목**이 존재
- 프론트엔드에서 품목별로 행을 분리하여 표시
- `keyField="rfqNo"` 사용 시 같은 key가 중복 발생

---

**1개 RFQ(견적 요청)에 여러 품목이 있는 것은 매우 일반적**

```
예시:
RFQ-2026-0001 "사무용품 구매 견적"
├─ 품목1: 27인치 모니터 × 10대
├─ 품목2: 무선 키보드 × 10개
├─ 품목3: 무선 마우스 × 10개
└─ 품목4: USB 허브 × 5개

```

실제 구매 업무에서 **한 번의 견적 요청으로 여러 품목을 함께 요청**하는 것이 효율적

## **🔍 근본 원인 분석**

### **데이터베이스 상태**

```
-- RFQHD (견적 헤더)
RFQ-2026-0001 | 모니터 구매 견적

-- RFQDT (견적 상세)
RFQ-2026-0001, LINE_NO=1 | 27인치 모니터
RFQ-2026-0001, LINE_NO=2 | 무선 키보드 마우스 세트  ← 같은 RFQ!

```

### **백엔드 분석 결과**

PurchaseOrderService.java (35-57줄)

**백엔드는 이미 올바르게 구현되어 있었습니다!** ✅

```java
publicList<RfqSelectedDTO>getRfqSelectedList(...) {
// 1. RFQ 헤더 목록 조회
List<RfqSelectedDTO> list= purchaseOrderMapper.selectRfqSelectedList(params);

// 2. 각 RFQ에 대해 품목 상세 조회하여 추가
for (RfqSelectedDTO rfq: list) {
List<RfqSelectedItemDTO> items= purchaseOrderMapper.selectRfqSelectedItems(rfq.getRfqNo());
        rfq.setItems(items);// ✅ 품목을 배열로 설정
    }

return list;
}

```

**백엔드 응답 구조:**

```tsx
[
  {
"rfqNo":"RFQ-2026-0001",
"rfqName":"모니터 구매 견적",
"items": [
      {"itemCode":"ITM20250001","itemName":"27인치 모니터" },
      {"itemCode":"ITM20250001","itemName":"무선 키보드 마우스 세트" }
    ]
  }
]

```

### **프론트엔드 분석 결과**

page.tsx (89-110줄)

프론트엔드가 **의도적으로 품목별로 행을 분리**:

```tsx
result.forEach((rfq)=> {
  rfq.items.forEach((item)=> {
    transformed.push({
      rfqNo: rfq.rfqNo,// ← 같은 rfqNo
      itemCode: item.itemCode,// ← 다른 itemCode
...
    });
  });
});

```

**변환 후 데이터:**

```
행1: rfqNo="RFQ-2026-0001", itemCode="ITM20250001"
행2: rfqNo="RFQ-2026-0001", itemCode="ITM20250001"  ← 같은 rfqNo!

```

**문제:** `keyField="rfqNo"` 사용 시 중복 key 발생

---

## **✅ 해결 방법**

### **수정 파일**

/order/pending/ page.tsx

### **1️⃣ PendingOrderRow 인터페이스에 `uniqueKey` 추가**

```tsx
interfacePendingOrderRow {
  uniqueKey:string;// ✅ 고유 key (rfqNo + itemCode 조합)
  rfqNo:string;
  itemCode:string;
...
}

```

### **2️⃣ 데이터 변환 시 고유 key 생성**

```tsx
// Before (중복 key 발생)
transformed.push({
  rfqNo: rfq.rfqNo||'',
  itemCode: item.itemCode||'',
...
});

// After (고유 key 생성)
transformed.push({
  uniqueKey:`${rfq.rfqNo}-${item.itemCode}`,// ✅ 고유 key
  rfqNo: rfq.rfqNo||'',
  itemCode: item.itemCode||'',
...
});

```

### **3️⃣ DataGrid의 `keyField` 변경**

```tsx
// Before
<DataGrid
keyField="rfqNo"// ❌ 중복 발생
...
/>

// After
<DataGrid
keyField="uniqueKey"// ✅ 고유 key 사용
...
/>

```

---

## **🎯 결과**

- `rfqNo`만 사용: ❌ 중복 key 발생
- `rfqNo + itemCode`: ✅ 고유 key 보장

### **변환 후 데이터 (고유 key 적용)**

```
행1: uniqueKey="RFQ-2026-0001-ITM20250001", rfqNo="RFQ-2026-0001"
행2: uniqueKey="RFQ-2026-0001-ITM20250001", rfqNo="RFQ-2026-0001"

```

✅ **각 행이 고유한 key를 가지게 되어 React 경고 해결**

---

## **📌 왜 프론트엔드에서 수정했나?**

### **백엔드 수정 vs 프론트엔드 수정**

| **방법** | **장점** | **단점** |
| --- | --- | --- |
| **백엔드 수정** | 데이터 구조 명확 | 이미 올바르게 구현됨 |
| **프론트엔드 수정** ⭐ | 간단, 빠른 해결 | - |

**결론:** 백엔드는 이미 올바르게 구현되어 있었으므로, 프론트엔드에서 고유 key만 생성하면 해결됩니다.

---

## **💡 결론**

### **문제 해결 접근법**

1. **근본 원인 파악**: 데이터베이스 → 백엔드 → 프론트엔드 순서로 분석
2. **백엔드 우선 검증**: 백엔드가 올바른 데이터 구조를 반환하는지 확인
3. **최소 수정 원칙**: 이미 올바르게 구현된 부분은 건드리지 않음

### **1개 견적요청에 여러 품목은 정상**

- 실제 비즈니스에서 **1개 견적에 여러 품목**은 일반적
- 백엔드는 이를 올바르게 처리 (RFQ별 그룹화)
- 프론트엔드는 표시 목적으로 품목별 행 분리
- **고유 key만 올바르게 설정하면 문제없음**





