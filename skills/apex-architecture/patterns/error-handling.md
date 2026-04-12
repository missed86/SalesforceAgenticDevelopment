# Error Handling Patterns

Structured exception handling. Custom exceptions for domains. Standardized responses for LWC.

## Custom Exception Hierarchy

```apex
// Base application exception
public class AppException extends Exception {}

// Domain-specific exceptions
public class OrderException extends AppException {}
public class PaymentException extends AppException {}
public class IntegrationException extends AppException {}

// Granular exceptions when needed
public class OrderValidationException extends OrderException {}
public class OrderProcessingException extends OrderException {}
```

## Exception Usage In Services

```apex
public with sharing class PaymentService {

    public static PaymentResultDto processPayment(Id orderId, Decimal amount) {
        try {
            Order ord = OrderSelector.selectById(orderId);

            if (ord == null) {
                throw new OrderException('Order not found: ' + orderId);
            }

            OrderDomain.validateForPayment(ord);

            PaymentGatewayResponse response = callPaymentGateway(ord, amount);

            if (!response.isSuccess) {
                throw new PaymentException(
                    'Payment declined: ' + response.declineReason
                );
            }

            update new Order(Id = orderId, PaymentStatus__c = 'Paid');

            return new PaymentResultDto(true, 'Payment processed', response.transactionId);

        } catch (PaymentException e) {
            return new PaymentResultDto(false, e.getMessage(), null);
        } catch (DmlException e) {
            return new PaymentResultDto(false, 'Failed to update order: ' + e.getDmlMessage(0), null);
        }
    }
}
```

## AuraHandledException for LWC

Always wrap exceptions for the frontend with `AuraHandledException`:

```apex
public with sharing class OrderController {

    @AuraEnabled
    public static OrderResultDto submitOrder(Id orderId) {
        try {
            return OrderService.processOrder(orderId);
        } catch (OrderException e) {
            throw new AuraHandledException(e.getMessage());
        } catch (Exception e) {
            throw new AuraHandledException(
                'An unexpected error occurred. Please contact support.'
            );
        }
    }
}
```

On the LWC side, catch the error:

```javascript
import submitOrder from '@salesforce/apex/OrderController.submitOrder';

async handleSubmit() {
    try {
        const result = await submitOrder({ orderId: this.recordId });
        // handle success
    } catch (error) {
        this.errorMessage = error.body.message;
    }
}
```

## Standardized Response DTO

Use `ServiceResponseDto` for consistent responses across controllers and services.
Full implementation and usage: see [dto-wrapper.md](dto-wrapper.md#service-response-dto-standardized).

## Database.SaveResult Handling

```apex
public static List<String> processSaveResults(List<Database.SaveResult> results) {
    List<String> errors = new List<String>();

    for (Database.SaveResult sr : results) {
        if (!sr.isSuccess()) {
            for (Database.Error err : sr.getErrors()) {
                errors.add(sr.getId() + ': ' + err.getStatusCode() + ' - ' + err.getMessage());
            }
        }
    }

    return errors;
}
```

## Error Handling Decision Table

| Context | Technique | Why |
|---------|-----------|-----|
| Before trigger validation | `addError()` on field/record | Blocks DML, shows in UI |
| Service layer failure | Throw custom exception | Caller decides how to handle |
| LWC controller | Wrap in `AuraHandledException` | Clean client-side errors |
| Batch execute | `Database.update(records, false)` | Partial success, collect errors |
| REST API | Return `ServiceResponseDto` | Structured JSON response |
| Callout failure | Catch `CalloutException` specifically | Retry or log |

## Key Rules

- **Catch specific** exception types, not generic `Exception`
- **Never swallow** exceptions silently — always log or rethrow
- **Use `AuraHandledException`** as the last layer before LWC
- **Aggregate errors** in batch — don't throw on first failure
- **Include context** in messages: IDs, field names, values
- **Sanitize messages** for end users — no stack traces or internal details
