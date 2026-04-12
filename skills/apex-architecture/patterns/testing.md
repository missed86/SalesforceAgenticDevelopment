# Testing Patterns

Structured tests with factories, mocking, and bulk coverage.

## TestDataFactory (Centralized)

```apex
@isTest
public class TestDataFactory {

    // ── Accounts ──

    public static Account createAccount() {
        return createAccount('Test Account');
    }

    public static Account createAccount(String name) {
        Account acc = new Account(
            Name = name,
            Industry = 'Technology',
            BillingCountry = 'US'
        );
        insert acc;
        return acc;
    }

    public static List<Account> createAccounts(Integer count) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(new Account(
                Name = 'Test Account ' + i,
                Industry = 'Technology',
                BillingCountry = 'US'
            ));
        }
        insert accounts;
        return accounts;
    }

    // ── Contacts ──

    public static Contact createContact(Id accountId) {
        Contact con = new Contact(
            FirstName = 'Test',
            LastName = 'Contact',
            Email = 'test@example.com',
            AccountId = accountId
        );
        insert con;
        return con;
    }

    public static List<Contact> createContacts(Id accountId, Integer count) {
        List<Contact> contacts = new List<Contact>();
        for (Integer i = 0; i < count; i++) {
            contacts.add(new Contact(
                FirstName = 'Test' + i,
                LastName = 'Contact' + i,
                Email = 'test' + i + '@example.com',
                AccountId = accountId
            ));
        }
        insert contacts;
        return contacts;
    }

    // ── Opportunities ──

    public static Opportunity createOpportunity(Id accountId) {
        Opportunity opp = new Opportunity(
            Name = 'Test Opportunity',
            AccountId = accountId,
            StageName = 'Prospecting',
            CloseDate = Date.today().addDays(30),
            Amount = 10000
        );
        insert opp;
        return opp;
    }

    // ── Orders ──

    public static Order createOrder(Id accountId) {
        Order ord = new Order(
            AccountId = accountId,
            EffectiveDate = Date.today(),
            Status = 'Draft',
            Pricebook2Id = Test.getStandardPricebookId()
        );
        insert ord;
        return ord;
    }

    // ── Users ──

    public static User createStandardUser() {
        Profile stdProfile = [SELECT Id FROM Profile WHERE Name = 'Standard User' LIMIT 1];
        String uniqueId = String.valueOf(DateTime.now().getTime());
        User u = new User(
            Alias = 'tuser',
            Email = 'testuser' + uniqueId + '@example.com',
            EmailEncodingKey = 'UTF-8',
            LastName = 'TestUser',
            LanguageLocaleKey = 'en_US',
            LocaleSidKey = 'en_US',
            ProfileId = stdProfile.Id,
            TimeZoneSidKey = 'America/Los_Angeles',
            Username = 'testuser' + uniqueId + '@example.com.test'
        );
        insert u;
        return u;
    }
}
```

## Test Structure: Arrange-Act-Assert

```apex
@isTest
private class AccountServiceTest {

    @testSetup
    static void setup() {
        List<Account> accounts = TestDataFactory.createAccounts(5);
        for (Account acc : accounts) {
            TestDataFactory.createContacts(acc.Id, 3);
        }
    }

    @isTest
    static void cascadeOwnerToChildren_shouldUpdateContactOwner() {
        // Arrange
        Account acc = [SELECT Id, OwnerId FROM Account LIMIT 1];
        User newOwner = TestDataFactory.createStandardUser();
        acc.OwnerId = newOwner.Id;
        update acc;

        // Act
        Test.startTest();
        AccountService.cascadeOwnerToChildren(new Set<Id>{ acc.Id });
        Test.stopTest();

        // Assert
        List<Contact> contacts = [
            SELECT OwnerId FROM Contact WHERE AccountId = :acc.Id
        ];
        for (Contact c : contacts) {
            Assert.areEqual(newOwner.Id, c.OwnerId,
                'Contact owner should match account owner');
        }
    }

    @isTest
    static void cascadeOwnerToChildren_withEmptySet_shouldDoNothing() {
        // Act
        Test.startTest();
        AccountService.cascadeOwnerToChildren(new Set<Id>());
        Test.stopTest();

        // Assert — no exceptions, no changes
        Integer accountCount = [SELECT COUNT() FROM Account];
        Assert.isTrue(accountCount > 0, 'Accounts should remain unchanged');
    }
}
```

## Bulk Testing (200+ Records)

```apex
@isTest
static void triggerHandler_bulkInsert_shouldSetDefaults() {
    List<Account> accounts = new List<Account>();
    for (Integer i = 0; i < 200; i++) {
        accounts.add(new Account(Name = 'Bulk Account ' + i));
    }

    Test.startTest();
    insert accounts;
    Test.stopTest();

    List<Account> inserted = [
        SELECT Id, Status__c FROM Account WHERE Name LIKE 'Bulk Account%'
    ];
    Assert.areEqual(200, inserted.size(), 'All 200 records should be inserted');
    for (Account acc : inserted) {
        Assert.areEqual('Active', acc.Status__c,
            'Default status should be set by trigger');
    }
}
```

## Mocking HTTP Callouts

```apex
@isTest
private class PaymentServiceTest {

    private class PaymentGatewayMock implements HttpCalloutMock {
        private final Integer statusCode;
        private final String responseBody;

        public PaymentGatewayMock(Integer statusCode, String responseBody) {
            this.statusCode = statusCode;
            this.responseBody = responseBody;
        }

        public HttpResponse respond(HttpRequest req) {
            HttpResponse res = new HttpResponse();
            res.setStatusCode(statusCode);
            res.setBody(responseBody);
            return res;
        }
    }

    @isTest
    static void processPayment_successfulResponse_shouldMarkPaid() {
        Account acc = TestDataFactory.createAccount();
        Order ord = TestDataFactory.createOrder(acc.Id);

        String mockResponse = '{"status":"approved","transactionId":"TXN-123"}';
        Test.setMock(HttpCalloutMock.class, new PaymentGatewayMock(200, mockResponse));

        Test.startTest();
        PaymentResultDto result = PaymentService.processPayment(ord.Id, 100.00);
        Test.stopTest();

        Assert.isTrue(result.isSuccess, 'Payment should succeed');
        Assert.areEqual('TXN-123', result.transactionId, 'Transaction ID should match');
    }

    @isTest
    static void processPayment_failedResponse_shouldReturnError() {
        Account acc = TestDataFactory.createAccount();
        Order ord = TestDataFactory.createOrder(acc.Id);

        Test.setMock(HttpCalloutMock.class, new PaymentGatewayMock(500, 'Server Error'));

        Test.startTest();
        PaymentResultDto result = PaymentService.processPayment(ord.Id, 100.00);
        Test.stopTest();

        Assert.isFalse(result.isSuccess, 'Payment should fail');
    }
}
```

## Testing Exceptions

```apex
@isTest
static void validateOrder_cancelledOrder_shouldThrow() {
    Order ord = new Order(Status = 'Cancelled', TotalAmount = 100);

    try {
        OrderDomain.validateForProcessing(ord);
        Assert.fail('Expected ValidationException');
    } catch (OrderDomain.ValidationException e) {
        Assert.isTrue(
            e.getMessage().contains('cancelled'),
            'Error should mention cancelled status'
        );
    }
}
```

## StubProvider for Dependency Injection

```apex
@isTest
private class OrderProcessingServiceTest {

    private class OrderRepoStub implements System.StubProvider {
        public Object handleMethodCall(
            Object stubbedObject, String methodName,
            Type returnType, List<Type> paramTypes,
            List<String> paramNames, List<Object> paramValues
        ) {
            if (methodName == 'findActive') {
                return new List<Order>{
                    new Order(OrderNumber = 'ORD-STUB-001', Status = 'Active'),
                    new Order(OrderNumber = 'ORD-STUB-002', Status = 'Active')
                };
            }
            return null;
        }
    }

    @isTest
    static void getActiveOrders_shouldReturnStubbed() {
        IOrderRepository mockRepo = (IOrderRepository)
            Test.createStub(IOrderRepository.class, new OrderRepoStub());

        OrderProcessingService service = new OrderProcessingService(mockRepo);

        Test.startTest();
        List<Order> orders = service.getActiveOrders();
        Test.stopTest();

        Assert.areEqual(2, orders.size(), 'Should return 2 stubbed orders');
        Assert.areEqual('ORD-STUB-001', orders[0].OrderNumber, 'First order should match');
    }
}
```

## Test Method Naming Convention

Pattern: `methodUnderTest_scenario_expectedBehavior`

```apex
@isTest static void calculateDiscount_orderOver10k_shouldApply10Percent() { }
@isTest static void calculateDiscount_orderUnder5k_shouldApplyNoDiscount() { }
@isTest static void validateOrder_nullAmount_shouldThrowValidation() { }
@isTest static void processPayment_gatewayTimeout_shouldReturnError() { }
```

## Testing Async Code

### Testing Queueable

```apex
@isTest
static void enrichmentJob_shouldUpdateAccounts() {
    Account acc = TestDataFactory.createAccount();
    Test.setMock(HttpCalloutMock.class, new EnrichmentMock());

    Test.startTest();
    System.enqueueJob(new AccountEnrichmentJob(new Set<Id>{ acc.Id }));
    Test.stopTest(); // forces synchronous execution

    Account updated = [SELECT Description FROM Account WHERE Id = :acc.Id];
    Assert.isNotNull(updated.Description, 'Description should be enriched');
}
```

### Testing Batch

```apex
@isTest
static void staleCleanupBatch_shouldMarkStaleAccounts() {
    List<Account> accounts = TestDataFactory.createAccounts(50);
    // LastActivityDate is system-managed (read-only). Use a custom
    // LastReviewedDate__c field, or insert old Tasks to set it indirectly.
    List<Task> oldTasks = new List<Task>();
    for (Account acc : accounts) {
        oldTasks.add(new Task(
            WhatId = acc.Id,
            Subject = 'Old Activity',
            Status = 'Completed',
            ActivityDate = Date.today().addMonths(-13)
        ));
    }
    insert oldTasks;

    Test.startTest();
    Database.executeBatch(new StaleAccountCleanupBatch(), 200);
    Test.stopTest();

    List<Account> stale = [SELECT Status__c FROM Account WHERE Id IN :accounts];
    for (Account acc : stale) {
        Assert.areEqual('Stale', acc.Status__c, 'Should be marked stale');
    }
}
```

### Testing Schedulable

```apex
@isTest
static void weeklyScheduler_shouldExecuteBatch() {
    Test.startTest();
    String jobId = System.schedule(
        'Test Weekly Cleanup',
        '0 0 2 ? * SAT',
        new WeeklyCleanupScheduler()
    );
    Test.stopTest();

    CronTrigger ct = [SELECT State FROM CronTrigger WHERE Id = :jobId];
    Assert.areEqual('WAITING', ct.State, 'Job should be scheduled');
}
```

### Testing Platform Events

```apex
@isTest
static void orderEvent_shouldBePublished() {
    Test.startTest();
    OrderService.publishOrderEvent(new Set<Id>{ '801000000000001' });
    Test.stopTest();

    // Platform event triggers fire after Test.stopTest()
    // Assert on side effects of the subscriber trigger
}
```

## @isTest(testFor=...) — Spring '26

Link tests to production classes for `RunRelevantTests`:

```apex
@isTest(testFor=AccountService.class)
private class AccountServiceTest {
    // tests here — auto-selected during deploy of AccountService
}
```

## Key Rules

- **Never** use `seeAllData=true` — always create test data
- Use `@testSetup` for shared data across test methods
- Test bulk (200+ records), positive, negative, and edge cases
- Use `Test.startTest()` / `Test.stopTest()` to reset governor limits
- Name tests: `method_scenario_expected`
- One assertion focus per test method
- Mock all external callouts with `HttpCalloutMock`
- Use `System.runAs()` to test permission-based behavior
- Aim for 90%+ meaningful coverage, not just line coverage
