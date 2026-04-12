# DTO & Wrapper Patterns

Structured data transfer between layers. Never expose raw SObjects to external consumers.

## Core Principles

- **DTOs are data containers** — no business logic, no SOQL, no DML
- **`@AuraEnabled`** on every property exposed to LWC
- **Shape for the consumer** — flatten relationships, compute display values
- **Immutable intent** — set in constructor or via deserialization, not mutated after
- **One direction per DTO** — don't reuse the same class for input and output

## DTO Types

| Type | Direction | Naming | Example |
|------|-----------|--------|---------|
| Response DTO | Apex → LWC | `{Entity}Dto` | `AccountDto` |
| Request DTO | LWC → Apex | `{Action}Request` | `CreateOrderRequest` |
| Service Response | Service → Controller | `ServiceResponseDto` | `ServiceResponseDto` |
| Operation Result | Service → Caller | `{Action}ResultDto` | `PaymentResultDto` |
| Page Result | Apex → LWC (paginated) | `PageResultDto` | `PageResultDto` |
| Lookup Option | Apex → LWC (combobox) | `OptionDto` | `OptionDto` |

## Response DTO (Apex → LWC)

Flatten SObject relationships into a clean, UI-ready structure:

```apex
public class AccountDto {
    @AuraEnabled public Id accountId;
    @AuraEnabled public String name;
    @AuraEnabled public String industry;
    @AuraEnabled public Decimal annualRevenue;
    @AuraEnabled public String ownerName;
    @AuraEnabled public String billingLocation;
    @AuraEnabled public Integer contactCount;

    public AccountDto(Account acc) {
        this.accountId = acc.Id;
        this.name = acc.Name;
        this.industry = acc.Industry;
        this.annualRevenue = acc.AnnualRevenue;
        this.ownerName = acc.Owner?.Name;
        this.billingLocation = buildLocation(acc.BillingCity, acc.BillingCountry);
        this.contactCount = acc.Contacts?.size() ?? 0;
    }

    private String buildLocation(String city, String country) {
        List<String> parts = new List<String>();
        if (String.isNotBlank(city)) parts.add(city);
        if (String.isNotBlank(country)) parts.add(country);
        return String.join(parts, ', ');
    }
}
```

Why flatten:
- `acc.Owner.Name` becomes `ownerName` — LWC doesn't traverse relationships cleanly
- Computed fields like `billingLocation` are calculated once server-side
- `contactCount` avoids sending the entire child collection when you only need the count

## Request DTO (LWC → Apex)

Structured input from the frontend:

```apex
public class CreateOrderRequest {
    @AuraEnabled public Id accountId;
    @AuraEnabled public Date requestedDate;
    @AuraEnabled public List<LineItemRequest> lineItems;
    @AuraEnabled public String notes;

    public CreateOrderRequest() {}
}

public class LineItemRequest {
    @AuraEnabled public Id productId;
    @AuraEnabled public Integer quantity;
    @AuraEnabled public Decimal unitPrice;

    public LineItemRequest() {}
}
```

Controller receives as JSON string and deserializes:

```apex
@AuraEnabled
public static ServiceResponseDto createOrder(String requestJson) {
    try {
        CreateOrderRequest request = (CreateOrderRequest)
            JSON.deserialize(requestJson, CreateOrderRequest.class);
        OrderService.createFromRequest(request);
        return ServiceResponseDto.success('Order created');
    } catch (JSONException e) {
        throw new AuraHandledException('Invalid request format.');
    } catch (AppException e) {
        throw new AuraHandledException(e.getMessage());
    }
}
```

LWC side:

```javascript
async handleCreateOrder() {
    const request = {
        accountId: this.selectedAccountId,
        requestedDate: this.requestedDate,
        lineItems: this.lineItems.map(item => ({
            productId: item.productId,
            quantity: item.quantity,
            unitPrice: item.unitPrice
        })),
        notes: this.notes
    };
    const result = await createOrder({ requestJson: JSON.stringify(request) });
}
```

## Service Response DTO (Standardized)

Single response shape for all controller methods:

```apex
public class ServiceResponseDto {
    @AuraEnabled public Boolean isSuccess;
    @AuraEnabled public String message;
    @AuraEnabled public Object data;
    @AuraEnabled public List<String> errors;

    private ServiceResponseDto(Boolean isSuccess, String message, Object data) {
        this.isSuccess = isSuccess;
        this.message = message;
        this.data = data;
        this.errors = new List<String>();
    }

    public static ServiceResponseDto success(String message) {
        return new ServiceResponseDto(true, message, null);
    }

    public static ServiceResponseDto success(String message, Object data) {
        return new ServiceResponseDto(true, message, data);
    }

    public static ServiceResponseDto error(String message) {
        return new ServiceResponseDto(false, message, null);
    }

    public static ServiceResponseDto error(String message, List<String> errors) {
        ServiceResponseDto response = new ServiceResponseDto(false, message, null);
        response.errors = errors;
        return response;
    }
}
```

LWC handling:

```javascript
const result = await submitOrder({ orderId: this.recordId });
if (result.isSuccess) {
    // show success toast with result.message
} else {
    // show error toast with result.message
    // optionally display result.errors as a list
}
```

## Operation Result DTO

For services that return detailed operation outcomes:

```apex
public class PaymentResultDto {
    @AuraEnabled public Boolean isSuccess;
    @AuraEnabled public String message;
    @AuraEnabled public String transactionId;
    @AuraEnabled public Decimal amountCharged;
    @AuraEnabled public Datetime processedAt;

    public PaymentResultDto(Boolean isSuccess, String message, String transactionId) {
        this.isSuccess = isSuccess;
        this.message = message;
        this.transactionId = transactionId;
        this.processedAt = Datetime.now();
    }
}
```

## Pagination DTO

```apex
public class PageResultDto {
    @AuraEnabled public List<Object> records;
    @AuraEnabled public Integer totalRecords;
    @AuraEnabled public Integer currentPage;
    @AuraEnabled public Integer pageSize;
    @AuraEnabled public Integer totalPages;
    @AuraEnabled public Boolean hasNext;
    @AuraEnabled public Boolean hasPrevious;

    public PageResultDto(List<Object> records, Integer totalRecords,
                         Integer currentPage, Integer pageSize) {
        this.records = records;
        this.totalRecords = totalRecords;
        this.currentPage = currentPage;
        this.pageSize = pageSize;
        this.totalPages = (Integer) Math.ceil((Decimal) totalRecords / pageSize);
        this.hasNext = currentPage < this.totalPages;
        this.hasPrevious = currentPage > 1;
    }
}
```

## Lookup / Combobox Option DTO

For picklists and combobox components:

```apex
public class OptionDto {
    @AuraEnabled public String label;
    @AuraEnabled public String value;

    public OptionDto(String label, String value) {
        this.label = label;
        this.value = value;
    }

    public static List<OptionDto> fromSObjects(List<SObject> records,
                                                String labelField, String valueField) {
        List<OptionDto> options = new List<OptionDto>();
        for (SObject record : records) {
            options.add(new OptionDto(
                String.valueOf(record.get(labelField)),
                String.valueOf(record.get(valueField))
            ));
        }
        return options;
    }
}
```

Used in controller:

```apex
@AuraEnabled(cacheable=true)
public static List<OptionDto> getIndustryOptions() {
    List<OptionDto> options = new List<OptionDto>();
    for (AggregateResult ar : [
        SELECT Industry
        FROM Account
        WHERE Industry != null
        WITH SECURITY_ENFORCED
        GROUP BY Industry
        ORDER BY Industry
    ]) {
        String industry = (String) ar.get('Industry');
        options.add(new OptionDto(industry, industry));
    }
    return options;
}
```

## Inner Class vs Separate File

| Approach | When | Example |
|----------|------|---------|
| **Inner class** | Used only by one controller/service | `OrderController.OrderLineDto` |
| **Separate file** | Shared across multiple classes | `AccountDto.cls` |
| **Separate file** | Complex with constructor logic | `PageResultDto.cls` |
| **Separate file** | Standardized (ServiceResponseDto) | `ServiceResponseDto.cls` |

Inner class example:

```apex
public with sharing class OrderController {

    @AuraEnabled(cacheable=true)
    public static List<OrderSummaryDto> getOrderSummaries(Id accountId) {
        // ...
    }

    public class OrderSummaryDto {
        @AuraEnabled public Id orderId;
        @AuraEnabled public String orderNumber;
        @AuraEnabled public Decimal total;
        @AuraEnabled public String status;
    }
}
```

Rule of thumb: if you reference the DTO from more than one class, extract it to its own file.

## JSON Serialization Notes

- `@AuraEnabled` properties serialize automatically to/from JSON for LWC
- Null properties are **included** in the JSON with `null` value
- Apex → LWC: automatic (framework handles it)
- LWC → Apex: pass as `String`, deserialize with `JSON.deserialize()`
- Dates serialize as `YYYY-MM-DD`, DateTimes as ISO 8601
- To exclude a property from JSON: omit `@AuraEnabled` (it won't reach LWC)

## Key Rules

- **One direction per DTO** — don't use the same class for request and response
- **Always `@AuraEnabled`** on properties that LWC needs
- **Parameterless constructor** required on Request DTOs (for JSON deserialization)
- **Constructor from SObject** on Response DTOs (for easy mapping)
- **Flatten relationships** — `ownerName` not `owner.name`
- **Compute display values** server-side — formatting, concatenation, counts
- **Standardize responses** — use `ServiceResponseDto` across all controllers
- **Extract to file** when shared, keep as inner class when scoped to one controller
