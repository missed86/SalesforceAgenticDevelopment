# Configuration Patterns

Custom Metadata Types, Platform Cache, and lightweight Dependency Injection. Zero hardcoding.

## Custom Metadata Types (CMDT) — Primary Config Mechanism

CMDTs are deployable, cached, read-only, and don't count against SOQL limits.

### Accessing CMDT Records

```apex
// Single record by DeveloperName
AppConfig__mdt config = AppConfig__mdt.getInstance('Default');
String apiEndpoint = config.ApiEndpoint__c;
Integer maxRetries = (Integer) config.MaxRetries__c;

// All records of a type
Map<String, AppConfig__mdt> allConfigs = AppConfig__mdt.getAll();

// Filtered by a field
List<DiscountRule__mdt> rules = [
    SELECT DeveloperName, MinAmount__c, DiscountPercentage__c
    FROM DiscountRule__mdt
    WHERE IsActive__c = true
    ORDER BY MinAmount__c DESC
];
```

### CMDT-Driven Business Logic

```apex
public with sharing class DiscountEngine {

    private static List<DiscountRule__mdt> rules {
        get {
            if (rules == null) {
                rules = [
                    SELECT DeveloperName, MinAmount__c, DiscountPercentage__c
                    FROM DiscountRule__mdt
                    WHERE IsActive__c = true
                    ORDER BY MinAmount__c DESC
                ];
            }
            return rules;
        }
        private set;
    }

    public static Decimal calculateDiscount(Decimal orderAmount) {
        for (DiscountRule__mdt rule : rules) {
            if (orderAmount >= rule.MinAmount__c) {
                return orderAmount * (rule.DiscountPercentage__c / 100);
            }
        }
        return 0;
    }
}
```

### Feature Toggles With CMDT

```apex
// Custom Metadata Type: FeatureToggle__mdt
// Fields: IsEnabled__c (Checkbox), Description__c (Text)

public with sharing class FeatureToggle {

    public static Boolean isEnabled(String featureName) {
        FeatureToggle__mdt toggle = FeatureToggle__mdt.getInstance(featureName);
        return toggle != null && toggle.IsEnabled__c;
    }
}

// Usage anywhere:
if (FeatureToggle.isEnabled('NewPricingEngine')) {
    NewPricingService.calculate(orders);
} else {
    LegacyPricingService.calculate(orders);
}
```

## CMDT vs Custom Settings vs Custom Labels

| Feature | CMDT | Custom Settings | Custom Labels |
|---------|------|----------------|---------------|
| Deployable | Yes | No (data) | Yes |
| SOQL count | No | Depends | No |
| Editable at runtime | No | Yes | No |
| Supports relationships | Yes | No | No |
| Best for | Config, rules, mappings | User preferences, runtime flags | UI labels, messages |

**Rule of thumb**: Use CMDT for application config. Use Custom Settings only when runtime writes are needed.

## Platform Cache

### Org Cache for Shared Data

```apex
public with sharing class CacheService {

    private static final String PARTITION = 'local.MainPartition';

    public static Object get(String key) {
        return Cache.Org.get(PARTITION + '.' + key);
    }

    public static void put(String key, Object value, Integer ttlSeconds) {
        Cache.Org.put(PARTITION + '.' + key, value, ttlSeconds);
    }

    public static void remove(String key) {
        Cache.Org.remove(PARTITION + '.' + key);
    }

    public static Boolean contains(String key) {
        return Cache.Org.contains(PARTITION + '.' + key);
    }
}
```

### CacheBuilder for Auto-Population

```apex
public class AccountCacheBuilder implements Cache.CacheBuilder {

    public Object doLoad(String key) {
        return [
            SELECT Id, Name, Industry
            FROM Account
            WHERE Id = :key
            WITH SECURITY_ENFORCED
            LIMIT 1
        ];
    }
}

// Usage — cache auto-populates on miss:
Account acc = (Account) Cache.Org.get(
    AccountCacheBuilder.class, accountId
);
```

### Cache Decision Table

| Data Type | Cache? | TTL | Why |
|-----------|--------|-----|-----|
| CMDT records | Optional (already cached) | N/A | CMDT has built-in caching |
| Picklist values | Yes | 1 hour | Rarely changes, frequently read |
| Org configuration | Yes | 30 min | Avoids repeated queries |
| User session data | Session Cache | Session | Per-user, temporary |
| Frequently queried records | Yes | 5-15 min | Reduces SOQL usage |

## Lightweight Dependency Injection (No Framework)

Use DI when you need to mock dependencies in tests without hitting the database.
Most services use static methods (see [service-layer.md](service-layer.md)). Switch to
instance methods with DI only when testability of external dependencies justifies it.

### Dual Constructor Pattern

```apex
public with sharing class OrderProcessingService {

    private final IOrderRepository repository;

    public OrderProcessingService() {
        this(new OrderRepository());
    }

    @TestVisible
    private OrderProcessingService(IOrderRepository repository) {
        this.repository = repository;
    }

    public List<Order> getActiveOrders() {
        return repository.findActive();
    }
}

public interface IOrderRepository {
    List<Order> findActive();
}

public with sharing class OrderRepository implements IOrderRepository {
    public List<Order> findActive() {
        return [
            SELECT Id, OrderNumber, Status, TotalAmount
            FROM Order
            WHERE Status = 'Active'
            WITH SECURITY_ENFORCED
        ];
    }
}
```

In tests, inject a mock:

```apex
@isTest
static void getActiveOrders_shouldReturnFromRepository() {
    IOrderRepository mockRepo = (IOrderRepository)
        Test.createStub(IOrderRepository.class, new OrderRepoStub());

    OrderProcessingService service = new OrderProcessingService(mockRepo);
    List<Order> orders = service.getActiveOrders();

    Assert.areEqual(2, orders.size(), 'Should return stubbed orders');
}
```

### CMDT-Based Service Resolution

```apex
// CMDT: ServiceConfig__mdt with field ImplementationClass__c

public with sharing class ServiceFactory {

    public static Object resolve(String serviceName) {
        ServiceConfig__mdt config = ServiceConfig__mdt.getInstance(serviceName);
        if (config == null) {
            throw new AppException('No service configured for: ' + serviceName);
        }
        Type serviceType = Type.forName(config.ImplementationClass__c);
        return serviceType.newInstance();
    }
}

// Usage:
IPaymentGateway gateway = (IPaymentGateway) ServiceFactory.resolve('PaymentGateway');
```

## Key Rules

- CMDT > Custom Settings for application configuration
- Cache config data that's read frequently and changes rarely
- Use CacheBuilder to auto-populate on cache miss
- For DI, prefer the dual constructor pattern — simple and testable
- Use CMDT-based resolution only when you need runtime-swappable implementations
- Never hardcode IDs, endpoints, thresholds, or feature flags
