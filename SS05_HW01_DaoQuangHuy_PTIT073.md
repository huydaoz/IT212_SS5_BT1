# Bài 1: Phân tích & Lựa chọn Prompt Chain-of-thought

## Đáp án lựa chọn

**Chọn phương án B.**

## Vì sao phương án B tối ưu nhất?

Phương án B là prompt tốt nhất vì nó hướng dẫn AI giải bài toán theo quy trình rõ ràng trước khi sinh mã nguồn. Với bài toán tính thuế TNCN lũy tiến, lỗi thường gặp không nằm ở cú pháp Java mà nằm ở nghiệp vụ: xác định sai thu nhập tính thuế, sai biên bậc thuế, sai cách tính từng phần hoặc dùng kiểu dữ liệu gây sai số tiền tệ.

Phương án B đáp ứng tốt cấu trúc 5 thành phần của một prompt hiệu quả:

1. **Vai trò**: yêu cầu AI đóng vai Java Developer chuyên nghiệp kiêm chuyên gia tài chính. Điều này định hướng AI vừa chú ý chất lượng mã nguồn, vừa chú ý nghiệp vụ tính thuế.

2. **Nhiệm vụ**: viết class Java tính thuế TNCN lũy tiến dựa trên thu nhập sau giảm trừ. Nhiệm vụ không mơ hồ như "viết hàm tính thuế" mà nêu rõ loại thuế và cách tiếp cận.

3. **Ngữ cảnh nghiệp vụ**: prompt nêu đầy đủ giảm trừ bản thân 11 triệu, người phụ thuộc 4.4 triệu/người, bảo hiểm 10.5% tổng thu nhập và bảng thuế 7 bậc. Đây là dữ liệu quan trọng để tránh AI tự suy đoán.

4. **Ràng buộc kỹ thuật**: yêu cầu Java 17 và `BigDecimal`. Đây là ràng buộc rất cần thiết vì tiền tệ không nên tính bằng `double` do có sai số nhị phân.

5. **Định dạng/quy trình đầu ra**: yêu cầu phân tích từng bước, liệt kê công thức, dry-run và cuối cùng mới sinh code. Đây là cách áp dụng Chain-of-thought ở mức kiểm soát được: AI phải trình bày các bước kiểm chứng nghiệp vụ trước khi viết mã.

## Lý do loại trừ các phương án còn lại

### Phương án A

Prompt A quá ngắn và mơ hồ:

- Không nêu rõ bảng thuế gồm những bậc nào.
- Không yêu cầu dùng `BigDecimal`, dễ khiến AI dùng `double`.
- Cụm "tối ưu nhất" không rõ tối ưu về hiệu năng, độ chính xác hay khả năng bảo trì.
- Không yêu cầu dry-run nên khó phát hiện lỗi ranh giới giữa các bậc thuế.
- AI có thể tự dùng biểu thuế khác hoặc giả định sai quy định hiện hành.

### Phương án C

Prompt C định hướng sai trọng tâm:

- Tính thuế lũy tiến là bài toán nghiệp vụ cần chính xác, không phải bài toán cần tối ưu bằng parallel stream.
- Dùng Stream song song cho vài bậc thuế là dư thừa, khó đọc và có thể gây lỗi nếu xử lý sai thứ tự các khoảng.
- Không nhắc tới giảm trừ gia cảnh, bảo hiểm, `BigDecimal` hoặc kiểm thử ranh giới.
- Tập trung vào hiệu năng trước khi đảm bảo đúng logic, không phù hợp với nghiệp vụ tài chính.

## Mã nguồn Java hoàn chỉnh

```java
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.List;
import java.util.Objects;

public class PersonalIncomeTaxCalculator {

    private static final BigDecimal PERSONAL_DEDUCTION = new BigDecimal("11000000");
    private static final BigDecimal DEPENDENT_DEDUCTION = new BigDecimal("4400000");
    private static final BigDecimal INSURANCE_RATE = new BigDecimal("0.105");
    private static final BigDecimal ZERO = BigDecimal.ZERO;

    private static final List<TaxBracket> TAX_BRACKETS = List.of(
            new TaxBracket(new BigDecimal("5000000"), new BigDecimal("0.05")),
            new TaxBracket(new BigDecimal("5000000"), new BigDecimal("0.10")),
            new TaxBracket(new BigDecimal("8000000"), new BigDecimal("0.15")),
            new TaxBracket(new BigDecimal("14000000"), new BigDecimal("0.20")),
            new TaxBracket(new BigDecimal("20000000"), new BigDecimal("0.25")),
            new TaxBracket(new BigDecimal("28000000"), new BigDecimal("0.30")),
            new TaxBracket(null, new BigDecimal("0.35"))
    );

    public BigDecimal calculateTax(BigDecimal grossIncome, int numberOfDependents, boolean includeInsurance) {
        validateInput(grossIncome, numberOfDependents);

        BigDecimal taxableIncome = calculateTaxableIncome(grossIncome, numberOfDependents, includeInsurance);
        if (taxableIncome.compareTo(ZERO) <= 0) {
            return ZERO.setScale(0, RoundingMode.HALF_UP);
        }

        BigDecimal remainingIncome = taxableIncome;
        BigDecimal totalTax = ZERO;

        for (TaxBracket bracket : TAX_BRACKETS) {
            if (remainingIncome.compareTo(ZERO) <= 0) {
                break;
            }

            BigDecimal incomeInBracket = bracket.limit() == null
                    ? remainingIncome
                    : remainingIncome.min(bracket.limit());

            totalTax = totalTax.add(incomeInBracket.multiply(bracket.rate()));
            remainingIncome = remainingIncome.subtract(incomeInBracket);
        }

        return totalTax.setScale(0, RoundingMode.HALF_UP);
    }

    public BigDecimal calculateTaxableIncome(BigDecimal grossIncome, int numberOfDependents, boolean includeInsurance) {
        validateInput(grossIncome, numberOfDependents);

        BigDecimal dependentDeduction = DEPENDENT_DEDUCTION.multiply(BigDecimal.valueOf(numberOfDependents));
        BigDecimal insurance = includeInsurance ? grossIncome.multiply(INSURANCE_RATE) : ZERO;

        BigDecimal taxableIncome = grossIncome
                .subtract(PERSONAL_DEDUCTION)
                .subtract(dependentDeduction)
                .subtract(insurance);

        return taxableIncome.max(ZERO).setScale(0, RoundingMode.HALF_UP);
    }

    private void validateInput(BigDecimal grossIncome, int numberOfDependents) {
        Objects.requireNonNull(grossIncome, "Gross income must not be null");

        if (grossIncome.compareTo(ZERO) < 0) {
            throw new IllegalArgumentException("Gross income must not be negative");
        }

        if (numberOfDependents < 0) {
            throw new IllegalArgumentException("Number of dependents must not be negative");
        }
    }

    private record TaxBracket(BigDecimal limit, BigDecimal rate) {
    }

    public static void main(String[] args) {
        PersonalIncomeTaxCalculator calculator = new PersonalIncomeTaxCalculator();

        BigDecimal tax = calculator.calculateTax(
                new BigDecimal("30000000"),
                1,
                false
        );

        System.out.println("Tax amount: " + tax + " VND");
    }
}
```

## Dry-run ví dụ

Với tổng thu nhập 30,000,000 VND, 1 người phụ thuộc, không tính bảo hiểm:

- Thu nhập tính thuế = 30,000,000 - 11,000,000 - 4,400,000 = 14,600,000 VND.
- Bậc 1: 5,000,000 x 5% = 250,000 VND.
- Bậc 2: 5,000,000 x 10% = 500,000 VND.
- Bậc 3: 4,600,000 x 15% = 690,000 VND.
- Tổng thuế = 1,440,000 VND.
