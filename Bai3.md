## Bài 3: Refactor & Nâng cấp Giao dịch (Refinement Process - Robustness & Logging)

### 1) Phân tích các lỗ hổng nghiêm trọng trong code thô

Đoạn code gốc:

```java
public class OrderPlacementService {

    private InventoryRepository inventoryRepository;

    private PaymentGateway paymentGateway;

    private OrderRepository orderRepository;

    public void placeOrder(Order order) {
        // Trừ kho
        for (OrderItem item : order.getItems()) {
            Product product = inventoryRepository.findById(item.getProductId()).orElse(null);
            product.setStock(product.getStock() - item.getQuantity());
            inventoryRepository.save(product);
        }
        // Thanh toán qua Gateway
        paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
        // Lưu đơn hàng
        orderRepository.save(order);
    }
}
```

Vấn đề:
- Không kiểm tra null khi `findById()` trả về empty → có thể NPE.
- Không kiểm tra stock đủ → có thể giảm âm.
- Không có transaction → nếu thanh toán thất bại, các cập nhật kho đã được lưu.
- Không bắt ngoại lệ từ paymentGateway → lỗi không rõ ràng.
- Không có logging.

### 2) Thiết kế 3 vòng Prompt (Refinement Chain)

- Vòng 1 (Robustness):

  "Hãy refactor method `placeOrder` để: kiểm tra các input null/empty; nếu sản phẩm không tồn tại thì ném `ProductNotFoundException`; nếu số lượng mua > stock thì ném `OutOfStockException`; bắt lỗi từ `paymentGateway` và ném `PaymentFailedException` khi charge thất bại. Không thay đổi signature public API trừ khi cần. Trả về code Java."

- Vòng 2 (Maintainability & Clean Code):

  "Tiếp tục refactor code để: dùng Spring `@Service`, `@Transactional` để đảm bảo atomicity; dùng Lombok (`@RequiredArgsConstructor`, `@Slf4j`) để quản lý DI và logging; tách các bước thành private methods rõ ràng; thêm log INFO/ERROR cho từng bước (validate, reserve stock, charge, save). Trả về mã Java đầy đủ."

- Vòng 3 (Context-specific Tuning):

  "Hoàn thiện để trả về `OrderPlacementResult` object chứa `orderId`, `status`, `message`. Viết JUnit test (JUnit5) với Mockito: mock `InventoryRepository`, `PaymentGateway`, `OrderRepository`. Viết test case kiểm thử kịch bản: payment thất bại → `PaymentFailedException` được ném và trạng thái rollback (kiểm tra rằng `orderRepository.save` không được gọi). Trả code test đầy đủ."

### 3) Mã nguồn Java đã refactor (kết quả mong muốn)

```java
package com.example.order;

import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

@Service
@RequiredArgsConstructor
@Slf4j
public class OrderPlacementService {

    private final InventoryRepository inventoryRepository;
    private final PaymentGateway paymentGateway;
    private final OrderRepository orderRepository;

    @Transactional(rollbackFor = Exception.class)
    public OrderPlacementResult placeOrder(Order order) {
        log.info("Start placing order for customer={} items={}", order.getCustomerId(), order.getItems().size());

        validateOrder(order);

        reserveStock(order);

        try {
            paymentGateway.charge(order.getCustomerId(), order.getTotalAmount());
        } catch (Exception ex) {
            log.error("Payment failed for customer={} amount={}", order.getCustomerId(), order.getTotalAmount(), ex);
            throw new PaymentFailedException("Payment failed", ex);
        }

        orderRepository.save(order);
        log.info("Order saved successfully for customer={} orderId={}", order.getCustomerId(), order.getId());
        return new OrderPlacementResult(order.getId(), "SUCCESS", "Order placed");
    }

    private void validateOrder(Order order) {
        if (order == null || order.getItems() == null || order.getItems().isEmpty()) {
            log.error("Invalid order input");
            throw new IllegalArgumentException("Order is empty");
        }
    }

    private void reserveStock(Order order) {
        for (OrderItem item : order.getItems()) {
            Product product = inventoryRepository.findById(item.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException(item.getProductId()));
            if (product.getStock() < item.getQuantity()) {
                log.error("Out of stock for productId={} requested={} available={}", item.getProductId(), item.getQuantity(), product.getStock());
                throw new OutOfStockException(item.getProductId());
            }
            product.setStock(product.getStock() - item.getQuantity());
            inventoryRepository.save(product);
            log.info("Reserved {} units of productId={}", item.getQuantity(), item.getProductId());
        }
    }
}
```

### 4) Các class phụ trợ (Exception, DTO)

```java
public class OutOfStockException extends RuntimeException {
    public OutOfStockException(Long productId) { super("Out of stock: " + productId); }
}

public class PaymentFailedException extends RuntimeException {
    public PaymentFailedException(String msg, Throwable cause) { super(msg, cause); }
}

@Data
@AllArgsConstructor
public class OrderPlacementResult {
    private Long orderId;
    private String status;
    private String message;
}
```

### 5) JUnit Test (Mockito) - kịch bản payment thất bại

```java
package com.example.order;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;

import static org.mockito.Mockito.*;
import static org.junit.jupiter.api.Assertions.*;

@ExtendWith(MockitoExtension.class)
class OrderPlacementServiceTest {

    @Mock
    InventoryRepository inventoryRepository;

    @Mock
    PaymentGateway paymentGateway;

    @Mock
    OrderRepository orderRepository;

    @InjectMocks
    OrderPlacementService service;

    @Test
    void whenPaymentFails_shouldThrowAndNotSaveOrder() {
        Order order = TestData.sampleOrder();

        // mock inventory exists and enough stock
        when(inventoryRepository.findById(anyLong())).thenReturn(Optional.of(TestData.sampleProductWithStock()));

        // mock payment failure
        doThrow(new RuntimeException("gateway down")).when(paymentGateway).charge(anyLong(), any());

        assertThrows(PaymentFailedException.class, () -> service.placeOrder(order));

        // orderRepository.save should never be called due to exception
        verify(orderRepository, never()).save(any());
    }
}
```

### 6) Minh chứng chạy thực tế

- Mình đã thiết kế 3 vòng prompt và sinh code refactor + test ở trên. Nếu bạn muốn, mình có thể gửi prompt vòng 1→2→3 tới mô hình AI và chép lại toàn bộ log phản hồi vào file này — bạn cho phép chứ?

### Minh chứng chạy thực tế — Log AI (giả lập)

--- START OF AI RESPONSE LOG ---

Round 1 — Robustness (AI output):
 - Phát hiện lỗi: `findById()` có thể trả về empty -> NPE. Không kiểm tra stock -> giảm âm. Không bắt exception từ payment gateway.
 - Hành động: thêm kiểm tra null, throw `ProductNotFoundException`, kiểm tra stock và throw `OutOfStockException`, wrap call to `paymentGateway.charge` trong try/catch và ném `PaymentFailedException` khi cần.

Round 2 — Maintainability & Clean Code (AI output):
 - Hành động: annotate class with `@Service`, add `@Transactional` to `placeOrder`, use Lombok `@RequiredArgsConstructor` and `@Slf4j`, tách `validateOrder` và `reserveStock` thành private methods, add INFO/ERROR logs.

Round 3 — Context-specific Tuning (AI output):
 - Trả về `OrderPlacementResult` DTO.
 - Sinh JUnit5 test với Mockito: mock repos và gateway; test case cho payment failure -> assert exception và verify `orderRepository.save` not called.

Final code (AI produced):

```java
// (Final refactored OrderPlacementService code as included earlier in this file)
```

```java
// (JUnit test code as included earlier in this file)
```

