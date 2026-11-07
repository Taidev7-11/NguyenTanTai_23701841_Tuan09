# BÁO CÁO TRẢ LỜI CÂU HỎI BỔ SUNG — BUỔI THỰC HÀNH 9
**Môn học:** Thực hành Lập trình Thiết bị Di động (React Native)  
**Đề tài:** BookStore — Thanh toán, Đơn hàng & Thông báo  
**Sinh viên thực hiện:** Nguyễn Tấn Tài — MSSV: 23701841  

---

### Câu 1: Giá trị dẫn xuất (Derived State)
> **Câu hỏi:** Ở Checkout, tại sao `errors` và `valid` được tính trực tiếp mỗi lần render mà không dùng `useState` riêng? Nếu dùng `useState` thì dễ sai ở chỗ nào?

**Trả lời:**
- **Lý do tính trực tiếp mỗi lần render:** `errors` và `valid` là các *giá trị dẫn xuất* (derived state) phụ thuộc hoàn toàn vào state của form (`name`, `phone`, `address`) và danh sách sản phẩm trong giỏ (`items.length`). Khi tính trực tiếp trong quá trình render, ta đảm bảo nguyên lý *Single Source of Truth* (nguồn chân lý duy nhất). Bất cứ khi nào state đầu vào thay đổi, `errors` và `valid` được tính toán lại ngay lập tức và luôn đồng bộ 100% với dữ liệu người dùng vừa nhập mà không hề có độ trễ chu kỳ render.
- **Nguy cơ lỗi khi dùng `useState` riêng:**
  1. **Lệch pha dữ liệu (Stale state / Desynchronization):** Khi người dùng gõ vào ô nhập, nếu dùng `useState` ta buộc phải nhớ gọi hàm cập nhật lỗi (`setErrors`, `setValid`) ở mọi sự kiện `onChangeText` hoặc phải dùng thêm `useEffect`. `useEffect` sẽ khiến component bị re-render thêm một nhịp thừa và trong nhịp đó giá trị state `valid` vẫn mang giá trị cũ của lần render trước.
  2. **Dễ bỏ sót khi bảo trì:** Nếu sau này form bổ sung thêm trường nhập liệu mới hoặc có tính năng tự động điền (autofill) mà lập trình viên quên cập nhật logic `setValid` ở một nhánh nào đó, nút "Xác nhận đặt hàng" sẽ bị khoá hoặc mở sai trạng thái, dẫn tới bug logic nghiêm trọng.

---

### Câu 2: Thứ tự gọi hàm trong submit
> **Câu hỏi:** Trong `submit`, nếu để `clear()` chạy trước `await createOrder(...)` thì khi mất mạng người dùng gặp chuyện gì? Vì sao đặt `clear()` ngay sau `createOrder` là đúng?

**Trả lời:**
- **Hậu quả nếu gọi `clear()` trước `await createOrder(...)`:**
  - Nếu gọi `clear()` trước, giỏ hàng trong Redux store sẽ bị xoá ngay lập tức. Đồng thời hook `useAutoPersist` sẽ đồng bộ trạng thái rỗng này vào `AsyncStorage`.
  - Khi gặp sự cố mạng (mất kết nối, timeout, server sập), lời gọi `createOrder` sẽ thất bại và ném ra ngoại lệ. Khi đó đơn hàng **chưa hề được tạo trên hệ thống**, nhưng toàn bộ sản phẩm khách hàng đã cất công lựa chọn trong giỏ đều **bị mất sạch**. Khách hàng sẽ phải tìm và thêm lại từng cuốn sách từ đầu, tạo ra trải nghiệm người dùng cực kỳ tồi tệ.
- **Lý do đặt `clear()` ngay sau `await createOrder(...)` là đúng:**
  - Đảm bảo tính toàn vẹn nghiệp vụ giao dịch (transactional flow): Chỉ khi server tiếp nhận và tạo đơn hàng thành công (trả về mã 200/201), ứng dụng mới được phép xoá sạch giỏ hàng.
  - Khi có lỗi mạng hoặc lỗi API, luồng thực thi lập tức nhảy vào khối `catch`, giỏ hàng được bảo toàn nguyên vẹn. Ứng dụng hiển thị thông báo lỗi màu đỏ để người dùng kiểm tra kết nối và có thể bấm "Xác nhận đặt hàng" để thử lại ngay mà không mất dữ liệu.

---

### Câu 3: So sánh PUT và PATCH
> **Câu hỏi:** Nếu dùng `PUT /users/1` với `{ name, phone }` thì `db.json` còn lại gì ở user đó? Vì sao bài này chọn `PATCH`?

**Trả lời:**
- **Kết quả nếu dùng `PUT /users/1` với `{ name, phone }`:**
  - Theo chuẩn HTTP RESTful và cơ chế của `json-server`, `PUT` là phương thức thay thế toàn bộ tài nguyên (full replacement).
  - Do đó, nếu gửi `PUT /users/1` chỉ với `{ name, phone }`, `json-server` sẽ ghi đè toàn bộ bản ghi user cũ, làm mất sạch các trường không có trong payload gửi lên. Trong `db.json`, bản ghi user đó sẽ chỉ còn lại:
    ```json
    {
      "id": 1,
      "name": "Tên mới",
      "phone": "0901234567"
    }
    ```
    Toàn bộ thông tin quan trọng như `email`, `password`, `createdAt`, `avatar` đều bị biến mất vĩnh viễn.
- **Lý do chọn `PATCH`:**
  - `PATCH` là phương thức cập nhật từng phần (partial update). Server sẽ chỉ cập nhật những trường được truyền trong payload (`name` và `phone`), đồng thời giữ nguyên vẹn toàn bộ các trường dữ liệu còn lại (`id`, `email`, `password`,...).
  - Màn hình sửa hồ sơ chỉ cho phép đổi tên và số điện thoại, nên việc dùng `PATCH` là hoàn toàn chính xác, an toàn và bảo vệ tính toàn vẹn của dữ liệu người dùng.

---

### Câu 4: Cập nhật lạc quan (Optimistic Update)
> **Câu hỏi:** Ở màn Thông báo, nếu bỏ đoạn hoàn lại `read: false` trong `catch` thì xảy ra lỗi gì khi mất mạng? Hãy nêu một chức năng khác trong app nên dùng cách làm này và một chức năng không nên (ví dụ thanh toán).

**Trả lời:**
- **Lỗi xảy ra nếu bỏ hoàn lại `read: false` trong `catch`:**
  - Khi mất mạng, giao diện client đã cập nhật lạc quan sang `read: true` (nền chuyển trắng, mất chấm đỏ), nhưng yêu cầu `PATCH /notifications/:id` gửi lên server bị thất bại (trên database thông báo vẫn là `read: false`).
  - Người dùng lầm tưởng thông báo đã được lưu trạng thái đã đọc thành công. Tuy nhiên, khi họ chuyển tab, kéo làm mới hoặc mở lại app, dữ liệu tải lại từ server sẽ khiến thông báo đó đột ngột "chưa đọc" trở lại. Sự không nhất quán này gây khó hiểu và mất lòng tin của người dùng. Việc rollback về `read: false` trong khối `catch` giúp đồng bộ chính xác giao diện với thực tế dữ liệu trên server.
- **Chức năng NÊN dùng cập nhật lạc quan:**
  - *Thêm/xoá sách khỏi danh sách yêu thích (Wishlist ❤️ - Thả tim).* Đây là thao tác UI có tần suất cao, rủi ro thấp. Khi bấm tim, icon tim sáng lên ngay lập tức giúp app mượt mà (responsive). Nếu API lỗi thì hoàn lại trạng thái tim cũ.
- **Chức năng KHÔNG NÊN dùng cập nhật lạc quan:**
  - *Thanh toán / Đặt hàng (Checkout) hoặc Trừ tiền ví điện tử.* Đây là thao tác có tính ràng buộc nghiệp vụ và tài chính nghiêm ngặt. Hệ thống bắt buộc phải chờ máy chủ xác nhận giao dịch thành công (kiểm tra tồn kho, khoá thanh toán, tạo mã đơn) thì mới được hiển thị màn hình chúc mừng đặt hàng thành công.

---

### Câu 5: Code bổ sung — Bộ lọc trạng thái đơn hàng (Bonus Feature)
> **Câu hỏi:** Thêm bộ lọc trạng thái ở màn Đơn hàng (Tất cả / Chờ xác nhận / Đang giao / Hoàn tất / Đã huỷ) bằng state và một chỗ sửa trong dữ liệu truyền cho `FlatList`. Chụp màn hình kết quả.

**Giải pháp mã nguồn triển khai:**

1. **Định nghĩa kiểu và state lọc trong `OrdersScreen.tsx`:**
```tsx
type FilterStatus = "all" | OrderStatus;

const STATUS_TABS: { key: FilterStatus; label: string }[] = [
  { key: "all", label: "Tất cả" },
  { key: "pending", label: "Chờ xác nhận" },
  { key: "shipping", label: "Đang giao" },
  { key: "completed", label: "Hoàn tất" },
  { key: "cancelled", label: "Đã hủy" },
];

// Khai báo state
const [selectedStatus, setSelectedStatus] = useState<FilterStatus>("all");
```

2. **Lọc dữ liệu truyền vào `FlatList` bằng `useMemo`:**
```tsx
const filteredOrders = useMemo(() => {
  if (selectedStatus === "all") return orders;
  return orders.filter((o) => o.status === selectedStatus);
}, [orders, selectedStatus]);
```

3. **Giao diện thanh bộ lọc (Filter Chips Bar):**
```tsx
<View style={styles.filterContainer}>
  <ScrollView horizontal showsHorizontalScrollIndicator={false} contentContainerStyle={styles.filterScroll}>
    {STATUS_TABS.map((tab) => {
      const active = selectedStatus === tab.key;
      return (
        <Pressable
          key={tab.key}
          style={[styles.filterChip, active && styles.filterChipActive]}
          onPress={() => setSelectedStatus(tab.key)}
        >
          <Text style={[styles.filterChipText, active && styles.filterChipTextActive]}>
            {tab.label}
          </Text>
        </Pressable>
      );
    })}
  </ScrollView>
</View>
```

4. **Truyền dữ liệu đã lọc vào `FlatList`:**
```tsx
<FlatList
  data={filteredOrders}
  keyExtractor={(o) => String(o.id)}
  // ... các props khác giữ nguyên
/>
```

5. **Style định dạng cho thanh bộ lọc:**
```tsx
const styles = StyleSheet.create({
  // ...
  filterContainer: { backgroundColor: "#fff", paddingVertical: 8, borderBottomWidth: 1, borderBottomColor: colors.line },
  filterScroll: { paddingHorizontal: 12, gap: 8 },
  filterChip: { paddingHorizontal: 14, paddingVertical: 6, borderRadius: radius.pill, backgroundColor: "#F3F4F6", borderWidth: 1, borderColor: colors.line },
  filterChipActive: { backgroundColor: colors.indigo, borderColor: colors.indigo },
  filterChipText: { fontSize: 13, fontWeight: "600", color: colors.muted },
  filterChipTextActive: { color: "#fff", fontWeight: "700" },
});
```

*(Sinh viên có thể chụp màn hình giao diện ứng dụng khi chạy trên Expo Go hoặc Trình duyệt web với các tab lọc đơn hàng để đính kèm vào báo cáo nộp bài).*

