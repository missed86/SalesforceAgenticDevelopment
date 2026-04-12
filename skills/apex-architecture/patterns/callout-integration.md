# Callout & Integration Patterns

Structured external HTTP communication. Named Credentials, retry logic, and REST API exposure.

## Core Principles

- **Named Credentials** for all endpoints — never hardcode URLs or credentials
- **Centralized callout service** — one class per external system
- **Timeout and retry** — handle transient failures gracefully
- **Log all failures** — callout errors must be visible in production
- **Test with mocks** — use `HttpCalloutMock` for every callout

## Named Credentials

Always use Named Credentials (`callout:`) instead of hardcoded endpoints:

```apex
// CORRECT — Named Credential manages URL + auth
req.setEndpoint('callout:PaymentGateway/v2/charge');

// WRONG — hardcoded, breaks across environments
req.setEndpoint('https://api.payment.com/v2/charge');
```

Named Credentials handle:
- Base URL per environment (sandbox vs production)
- Authentication (OAuth, Basic, API Key) — no credentials in code
- Certificate management for mutual TLS

## Callout Service Pattern

```apex
public with sharing class HttpCalloutService {

    private static final Integer DEFAULT_TIMEOUT_MS = 10000;
    private static final Integer MAX_RETRIES = 2;

    public static HttpResponse doGet(String namedCredential, String path) {
        return doCallout(namedCredential, path, 'GET', null);
    }

    public static HttpResponse doPost(String namedCredential, String path, String body) {
        return doCallout(namedCredential, path, 'POST', body);
    }

    public static HttpResponse doPut(String namedCredential, String path, String body) {
        return doCallout(namedCredential, path, 'PUT', body);
    }

    private static HttpResponse doCallout(
        String namedCredential, String path, String method, String body
    ) {
        HttpRequest req = new HttpRequest();
        req.setEndpoint('callout:' + namedCredential + path);
        req.setMethod(method);
        req.setTimeout(DEFAULT_TIMEOUT_MS);
        req.setHeader('Content-Type', 'application/json');
        req.setHeader('Accept', 'application/json');

        if (body != null) {
            req.setBody(body);
        }

        return executeWithRetry(req);
    }

    private static HttpResponse executeWithRetry(HttpRequest req) {
        HttpResponse res;
        Integer attempts = 0;

        while (attempts <= MAX_RETRIES) {
            try {
                Http http = new Http();
                res = http.send(req);

                if (res.getStatusCode() < 500) {
                    return res;
                }

                Logger.warn('HttpCalloutService', 'executeWithRetry',
                    'Server error ' + res.getStatusCode() + ', attempt ' + (attempts + 1));

            } catch (CalloutException e) {
                if (attempts == MAX_RETRIES) {
                    Logger.error('HttpCalloutService', 'executeWithRetry', e);
                    throw new IntegrationException(
                        'Callout failed after ' + (MAX_RETRIES + 1) + ' attempts: ' + e.getMessage(), e
                    );
                }
            }

            attempts++;
        }

        Logger.error('HttpCalloutService', 'executeWithRetry',
            'Max retries exhausted. Last status: ' + res.getStatusCode(), null);
        throw new IntegrationException(
            'External service unavailable (HTTP ' + res.getStatusCode() + ')'
        );
    }
}
```

## Integration Service (Per External System)

```apex
public with sharing class PaymentGatewayService {

    private static final String CREDENTIAL = 'PaymentGateway';
    private static final String CLASS_NAME = 'PaymentGatewayService';

    public static PaymentGatewayResponse charge(String orderId, Decimal amount, String currency_x) {
        Map<String, Object> payload = new Map<String, Object>{
            'orderId' => orderId,
            'amount' => amount,
            'currency' => currency_x
        };

        HttpResponse res = HttpCalloutService.doPost(
            CREDENTIAL, '/v2/charge', JSON.serialize(payload)
        );

        if (res.getStatusCode() == 200 || res.getStatusCode() == 201) {
            return (PaymentGatewayResponse) JSON.deserialize(
                res.getBody(), PaymentGatewayResponse.class
            );
        }

        Logger.warn(CLASS_NAME, 'charge',
            'Payment API returned ' + res.getStatusCode() + ': ' + res.getBody());

        return new PaymentGatewayResponse(false, 'Payment service error', null);
    }

    public class PaymentGatewayResponse {
        public Boolean isSuccess;
        public String declineReason;
        public String transactionId;

        public PaymentGatewayResponse(Boolean isSuccess, String declineReason, String transactionId) {
            this.isSuccess = isSuccess;
            this.declineReason = declineReason;
            this.transactionId = transactionId;
        }

        public PaymentGatewayResponse() {}
    }
}
```

## Exposing REST API (@RestResource)

```apex
@RestResource(urlMapping='/api/orders/*')
global with sharing class OrderRestService {

    @HttpGet
    global static void doGet() {
        RestResponse res = RestContext.response;
        try {
            String orderId = RestContext.request.requestURI.substringAfterLast('/');

            if (String.isBlank(orderId)) {
                sendError(res, 400, 'Order ID is required');
                return;
            }

            Order ord = OrderSelector.selectById(orderId);
            if (ord == null) {
                sendError(res, 404, 'Order not found');
                return;
            }

            sendSuccess(res, 200, new OrderDto(ord));

        } catch (Exception e) {
            Logger.error('OrderRestService', 'doGet', e);
            sendError(res, 500, 'Internal server error');
        }
    }

    @HttpPost
    global static void doPost() {
        RestResponse res = RestContext.response;
        try {
            CreateOrderRequest request = (CreateOrderRequest) JSON.deserialize(
                RestContext.request.requestBody.toString(), CreateOrderRequest.class
            );

            OrderService.createFromRequest(request);
            sendSuccess(res, 201, ServiceResponseDto.success('Order created'));

        } catch (AppException e) {
            sendError(res, 422, e.getMessage());
        } catch (JSONException e) {
            sendError(res, 400, 'Invalid JSON format');
        } catch (Exception e) {
            Logger.error('OrderRestService', 'doPost', e);
            sendError(res, 500, 'Internal server error');
        }
    }

    // ── Response Helpers ──

    private static void sendSuccess(RestResponse res, Integer statusCode, Object body) {
        res.statusCode = statusCode;
        res.addHeader('Content-Type', 'application/json');
        res.responseBody = Blob.valueOf(JSON.serialize(body));
    }

    private static void sendError(RestResponse res, Integer statusCode, String message) {
        res.statusCode = statusCode;
        res.addHeader('Content-Type', 'application/json');
        res.responseBody = Blob.valueOf(
            JSON.serialize(new Map<String, String>{ 'error' => message })
        );
    }
}
```

## Testing Callouts

```apex
@isTest
private class PaymentGatewayServiceTest {

    private class SuccessMock implements HttpCalloutMock {
        public HttpResponse respond(HttpRequest req) {
            HttpResponse res = new HttpResponse();
            res.setStatusCode(200);
            res.setBody('{"isSuccess":true,"transactionId":"TXN-001","declineReason":null}');
            return res;
        }
    }

    private class ServerErrorMock implements HttpCalloutMock {
        public HttpResponse respond(HttpRequest req) {
            HttpResponse res = new HttpResponse();
            res.setStatusCode(503);
            res.setBody('Service Unavailable');
            return res;
        }
    }

    @isTest
    static void charge_success_shouldReturnTransaction() {
        Test.setMock(HttpCalloutMock.class, new SuccessMock());

        Test.startTest();
        PaymentGatewayService.PaymentGatewayResponse result =
            PaymentGatewayService.charge('ORD-001', 100.00, 'USD');
        Test.stopTest();

        Assert.isTrue(result.isSuccess, 'Should succeed');
        Assert.areEqual('TXN-001', result.transactionId, 'Transaction ID should match');
    }

    @isTest
    static void charge_serverError_shouldReturnFailure() {
        Test.setMock(HttpCalloutMock.class, new ServerErrorMock());

        Test.startTest();
        try {
            PaymentGatewayService.charge('ORD-001', 100.00, 'USD');
            Assert.fail('Should throw IntegrationException');
        } catch (IntegrationException e) {
            Assert.isTrue(e.getMessage().contains('unavailable'), 'Should mention unavailable');
        }
        Test.stopTest();
    }
}
```

## Decision: Sync vs Async Callout

| Context | Approach | Why |
|---------|----------|-----|
| LWC user action (button click) | Sync in `@AuraEnabled` | User waits for response |
| Trigger context | `@future(callout=true)` or Queueable | Callouts forbidden in trigger |
| Bulk operations | Queueable with chaining | Respect callout limits per txn |
| Scheduled sync | Schedulable → Batch with `Database.AllowsCallouts` | Large volume, configurable |

## Key Rules

- **Named Credentials** for every external endpoint — zero hardcoded URLs
- **Central `HttpCalloutService`** — DRY request construction and retry logic
- **One integration service per external system** — `PaymentGatewayService`, `ErpSyncService`
- **Retry only on 5xx** — client errors (4xx) should not be retried
- **Log every failure** — callout errors are invisible without explicit logging
- **Timeout aggressively** — 10-15 seconds max; long timeouts waste CPU time limits
- REST APIs: return proper HTTP status codes, structured JSON errors
- REST APIs: use `global with sharing` — enforce security
