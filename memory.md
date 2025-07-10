# Salesforce Development Knowledge Bank

## Core Enterprise Patterns (fflib-apex-common)

### **Domain Layer Pattern** 
- **Purpose**: Encapsulates business logic and validation specific to SObject records
- **Key Classes**: `fflib_SObjectDomain`
- **Best Practice**: All business logic should be implemented in Domain classes, NOT in triggers
- **Usage**: Extend `fflib_SObjectDomain` for each custom object (e.g., `AccountDomain`, `OpportunityDomain`)

### **Service Layer Pattern**
- **Purpose**: Provides application services and orchestrates business operations
- **Key Classes**: `fflib_SObjectService`
- **Best Practice**: Complex business processes should be orchestrated through Service classes
- **Usage**: Create service classes for major business operations (e.g., `AccountService`, `OpportunityService`)

### **Selector Layer Pattern (Data Mapper)**
- **Purpose**: Centralized data access and query logic
- **Key Classes**: `fflib_SObjectSelector`
- **Best Practice**: All SOQL queries should be encapsulated in Selector classes
- **Usage**: Create selector classes for each SObject (e.g., `AccountSelector`, `OpportunitySelector`)

### **Unit of Work Pattern**
- **Purpose**: Manages DML operations and maintains transaction boundaries
- **Key Classes**: `fflib_SObjectUnitOfWork`
- **Best Practice**: Use Unit of Work to manage related DML operations as a single transaction
- **Usage**: Bulkify DML operations and handle governor limits efficiently

## 🔧 Critical Implementation Guidelines

### **User Mode vs System Mode (2022 Update)**
- **IMPORTANT**: Use `USER_MODE` and `SYSTEM_MODE` explicitly instead of `with sharing`/`without sharing`
- **Performance**: Measurable CPU usage reduction with explicit mode declarations
- **Selector Classes**: Use `dataAccess` constructors instead of deprecated `enforceCRUD` and `enforceFLS` flags
- **DML Classes**: Use `fflib_SObjectUnitOfWork.UserModeDML` instead of deprecated `SimpleDML`

### **Trigger Architecture**
- **NEVER** put business logic directly in triggers
- **ALWAYS** delegate trigger logic to Domain classes
- **Pattern**: 
  ```apex
  // In trigger
  new MyObjectDomain(Trigger.new).onAfterInsert();
  
  // In Domain class
  public override void onAfterInsert() {
      // Business logic here
  }
  ```

### **Separation of Concerns**
- **Controllers**: Handle user interface logic only
- **Services**: Orchestrate business operations
- **Domains**: Manage record-specific business logic
- **Selectors**: Handle data access patterns
- **Unit of Work**: Manage DML transactions

## 📋 Coding Standards & Best Practices

### **Governor Limits Management**
- **Bulkify**: Always design for bulk operations (200+ records)
- **SOQL Limits**: Use Selector pattern to centralize and optimize queries
- **DML Limits**: Use Unit of Work to batch DML operations
- **CPU Limits**: Leverage USER_MODE/SYSTEM_MODE for performance optimization

### **Testing Patterns**

#### 🧪 **Comprehensive Test Class Framework**

**Test Categories and Strategy**

**Integration Tests (Traditional Salesforce)**
- **Purpose**: Full-stack testing with real DML operations
- **When**: At least one test per trigger, REST API endpoint, and batch job
- **Coverage**: Ensures deployment success and basic functionality
- **Performance**: Slower execution due to DML operations

**Unit Tests (ApexMocks)**
- **Purpose**: Isolated testing of business logic without DML
- **When**: Majority of test methods for Domain, Service, and Selector classes  
- **Coverage**: Focused on logic paths, edge cases, and method interactions
- **Performance**: Fast execution (3-5x faster than integration tests)

**Domain Class Testing Patterns**
```apex
@IsTest
private class AccountDomainTest {
    
    @IsTest
    static void newInstanceTest() {
        // Test factory method without DML
        IAccounts domain = Accounts.newInstance(
            new Set<Id>{ fflib_IDGenerator.generate(Account.SObjectType) }
        );
        System.assertNotEquals(null, domain);
    }
    
    @IsTest
    static void validateCreditRatingTest() {
        // Test business logic with mock data
        List<Account> accounts = new List<Account>{
            new Account(Name = 'Test', AnnualRevenue = 100000),
            new Account(Name = 'Test2', AnnualRevenue = 50000)
        };
        
        IAccounts domain = Accounts.newInstance(accounts);
        domain.validateCreditRating();
        
        // Verify business logic executed correctly
        System.assertEquals('A', accounts[0].Credit_Rating__c);
        System.assertEquals('B', accounts[1].Credit_Rating__c);
    }
    
    @IsTest
    static void integrationTestWithDML() {
        // One integration test for trigger coverage
        List<Account> accounts = TestDataFactory.createAccounts(3);
        
        Test.startTest();
        insert accounts;
        update accounts;
        Test.stopTest();
        
        // Verify trigger-based business logic
        accounts = [SELECT Id, Credit_Rating__c FROM Account WHERE Id IN :accounts];
        System.assert(!accounts.isEmpty());
    }
}
```

**Service Class Testing with ApexMocks**
```apex
@IsTest
private class AccountServiceImplTest {
    
    @IsTest
    static void createAccountsSuccessTest() {
        // Given - Mock Setup
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        fflib_ISObjectUnitOfWork mockUow = new fflib_SObjectMocks.SObjectUnitOfWork(mocks);
        IAccountsSelector mockSelector = (IAccountsSelector) mocks.mock(IAccountsSelector.class);
        
        List<Account> testAccounts = TestDataFactory.createAccounts(2);
        
        // Stubbing
        mocks.startStubbing();
        mocks.when(mockSelector.selectById(new Set<Id>())).thenReturn(testAccounts);
        mocks.stopStubbing();
        
        // Dependency Injection
        Application.UnitOfWork.setMock(mockUow);
        Application.Selector.setMock(mockSelector);
        
        // When - Execute Service Method
        Test.startTest();
        AccountService.createAccounts(new Set<Id>{'001000000000001'});
        Test.stopTest();
        
        // Then - Verify Interactions
        ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).registerNew(fflib_Match.anyObject());
        ((fflib_ISObjectUnitOfWork) mocks.verify(mockUow, 1)).commitWork();
    }
    
    @IsTest
    static void createAccountsExceptionTest() {
        // Test exception handling
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        fflib_ISObjectUnitOfWork mockUow = new fflib_SObjectMocks.SObjectUnitOfWork(mocks);
        
        mocks.startStubbing();
        ((fflib_ISObjectUnitOfWork) mocks.doThrowWhen(new DmlException(), mockUow)).commitWork();
        mocks.stopStubbing();
        
        Application.UnitOfWork.setMock(mockUow);
        
        try {
            Test.startTest();
            AccountService.createAccounts(new Set<Id>{'001000000000001'});
            Test.stopTest();
            System.assert(false, 'Expected DmlException');
        } catch (DmlException e) {
            System.assert(e.getMessage().contains('error'));
        }
    }
}
```

**Selector Class Testing Patterns**
```apex
@IsTest
private class AccountSelectorTest {
    
    @IsTest
    static void selectByIdTest() {
        // Given - Mock data without DML
        List<Account> mockAccounts = new List<Account>{
            new Account(Id = fflib_IDGenerator.generate(Account.SObjectType), Name = 'Test 1'),
            new Account(Id = fflib_IDGenerator.generate(Account.SObjectType), Name = 'Test 2')
        };
        
        // When - Test selector logic (not actual query)
        IAccountsSelector selector = new AccountsSelector();
        String query = selector.newQueryFactory().toSOQL();
        
        // Then - Verify query structure
        System.assert(query.contains('SELECT Id, Name'), 'Query should contain required fields');
        System.assert(query.contains('FROM Account'), 'Query should be from Account');
    }
    
    @IsTest
    static void selectByIndustryTest() {
        // Test custom selector methods
        List<Account> mockAccounts = TestDataFactory.createAccounts(5);
        
        // Integration test for actual query (minimal)
        insert mockAccounts;
        
        Test.startTest();
        List<Account> results = new AccountsSelector().selectByIndustry('Technology');
        Test.stopTest();
        
        System.assert(results.size() >= 0);
    }
}
```

**Test Data Factory Patterns**
```apex
@IsTest
public class TestDataFactory {
    
    public static List<Account> createAccounts(Integer count) {
        List<Account> accounts = new List<Account>();
        for (Integer i = 0; i < count; i++) {
            accounts.add(new Account(
                Id = fflib_IDGenerator.generate(Account.SObjectType),
                Name = 'Test Account ' + i,
                Industry = 'Technology',
                AnnualRevenue = 100000 + (i * 10000)
            ));
        }
        return accounts;
    }
    
    public static List<Account> createAccountsWithContacts(Integer accountCount, Integer contactsPerAccount) {
        List<Account> accounts = createAccounts(accountCount);
        
        for (Account acc : accounts) {
            List<Contact> contacts = new List<Contact>();
            for (Integer i = 0; i < contactsPerAccount; i++) {
                contacts.add(new Contact(
                    Id = fflib_IDGenerator.generate(Contact.SObjectType),
                    LastName = 'Test Contact ' + i,
                    AccountId = acc.Id
                ));
            }
            // Use fflib_ApexMocksUtils for relationships
            acc = (Account) fflib_ApexMocksUtils.makeRelationship(
                List<Account>.class,
                new List<Account>{ acc },
                Contact.AccountId,
                new List<List<Contact>>{ contacts }
            ).get(0);
        }
        return accounts;
    }
    
    public static fflib_ISObjectUnitOfWork createMockUnitOfWork() {
        fflib_ApexMocks mocks = new fflib_ApexMocks();
        return new fflib_SObjectMocks.SObjectUnitOfWork(mocks);
    }
}
```

**ApexMocks Advanced Patterns**
```apex
// Stubbing method returns
mocks.startStubbing();
mocks.when(mockSelector.selectById(accountIds)).thenReturn(accounts);
mocks.when(mockService.processAccounts(accounts)).thenReturn(true);
mocks.stopStubbing();

// Stubbing exceptions
mocks.startStubbing();
mocks.when(mockSelector.selectById(accountIds)).thenThrow(new QueryException('Mock query error'));
mocks.stopStubbing();

// Verification with counters
((IAccountsSelector) mocks.verify(mockSelector, 1)).selectById(accountIds);
((IAccountsSelector) mocks.verify(mockSelector, mocks.never())).selectAll();
((IAccountsSelector) mocks.verify(mockSelector, mocks.atLeast(1))).selectById(fflib_Match.anyObject());

// Verification with matchers
List<Account> expectedAccounts = (List<Account>) fflib_Match.sObjectsWith(
    new List<Map<Schema.SObjectField, Object>>{
        new Map<SObjectField, Object>{
            Account.Id => mockAccountId,
            Account.Name => 'Expected Name'
        }
    }
);
((fflib_ISObjectUnitOfWork) mocks.verify(mockUow)).registerDirty(expectedAccounts);
```

**Coverage Optimization Strategies**
- **Meaningful Coverage Over Percentage**: Test decision points, calculations, and validation rules
- **Edge Cases**: Test boundary conditions, null values, and empty collections
- **Error Scenarios**: Test exception handling and negative cases
- **Integration Points**: Test service-to-service communication

**Performance Testing Integration**
```apex
@IsTest
static void testGovernorLimits() {
    List<Account> accounts = TestDataFactory.createAccounts(200);
    
    Test.startTest();
    Integer soqlBefore = Limits.getQueries();
    Integer dmlBefore = Limits.getDMLStatements();
    
    AccountService.processAccounts(accounts);
    
    Integer soqlAfter = Limits.getQueries();
    Integer dmlAfter = Limits.getDMLStatements();
    Test.stopTest();
    
    // Verify efficient resource usage
    System.assert(soqlAfter - soqlBefore <= 2, 'Too many SOQL queries');
    System.assert(dmlAfter - dmlBefore <= 1, 'Too many DML operations');
}
```

**Key Testing Benefits**
- **5x Faster**: ApexMocks tests run 3-5x faster than traditional DML tests
- **No Setup**: No complex data setup or cleanup required
- **Parallel Safe**: Tests don't interfere with each other
- **Isolation**: Each test focuses on specific business logic
- **Quality over Quantity**: Focus on meaningful test scenarios
- **Business Logic Coverage**: Test decision points and calculations
- **Error Path Coverage**: Test exception handling and validation

### **Application Factory Pattern**
- **Purpose**: Centralized instantiation of major layer classes
- **Classes**: `Application.Service`, `Application.Domain`, `Application.Selector`
- **Best Practice**: Use Application Factory for consistent object creation
- **Usage**: `Application.Service.newInstance(ServiceClass.class)`

## 🔄 Migration Guidelines

### **From Legacy Code**
- **Gradual Migration**: Migrate triggers one at a time to Domain pattern
- **Existing Queries**: Refactor SOQL into Selector classes
- **Business Logic**: Extract from controllers/triggers into Service/Domain classes
- **Transaction Management**: Replace scattered DML with Unit of Work

### **Team Adoption**
- **Training**: Ensure team understands separation of concerns
- **Code Reviews**: Enforce pattern adherence through reviews
- **Documentation**: Reference comprehensive pattern documentation
- **Examples**: Use sample applications for learning

## 📚 Essential Resources

### **Documentation Links**
- Apex Enterprise Patterns - Separation of Concerns
- Apex Enterprise Patterns - Service Layer  
- Apex Enterprise Patterns - Domain Layer
- Apex Enterprise Patterns - Selector Layer
- Unit Testing with Apex Enterprise Patterns and ApexMocks

### **Learning Resources**
- "Salesforce Platform Enterprise Architecture" by Andrew Fawcett
- Advanced Apex Enterprise Patterns webinar
- Apex Hours: Apex Enterprise Patterns
- Dreamforce presentations on Enterprise Patterns

## ⚠️ Common Anti-Patterns to Avoid

### **Trigger Anti-Patterns**
- ❌ Business logic directly in triggers
- ❌ SOQL/DML inside loops
- ❌ Hardcoded values in triggers
- ❌ Trigger logic without bulkification

### **Architecture Anti-Patterns**
- ❌ Controllers with business logic
- ❌ Scattered SOQL queries throughout codebase
- ❌ Manual DML operations without Unit of Work
- ❌ Mixing data access with business logic

### **Testing Anti-Patterns**
- ❌ Tests that depend on existing data
- ❌ Tests without proper mocking
- ❌ Integration tests disguised as unit tests
- ❌ Poor test data management

## 🚀 Implementation Checklist

### **New Project Setup**
- [ ] Deploy ApexMocks dependency
- [ ] Deploy fflib-apex-common library
- [ ] Create Application Factory configuration
- [ ] Establish naming conventions
- [ ] Set up test data factories

### **For Each SObject**
- [ ] Create Domain class extending `fflib_SObjectDomain`
- [ ] Create Service class for business operations
- [ ] Create Selector class extending `fflib_SObjectSelector`
- [ ] Implement trigger using Domain delegation
- [ ] Write comprehensive unit tests with ApexMocks

### **Code Quality Gates**
- [ ] No business logic in triggers
- [ ] All SOQL in Selector classes
- [ ] DML operations through Unit of Work
- [ ] Proper separation of concerns
- [ ] Comprehensive unit test coverage

---

*This memory bank should be referenced for all Salesforce development projects to ensure consistent application of enterprise patterns and best practices.* 

## Lightning Web Components (LWC) Patterns & Best Practices

### Core LWC Principles
- **Web Standards**: LWC is built on web standards (ES6+, Web Components, Shadow DOM)
- **Reactive Architecture**: One-way data flow prevents bidirectional binding issues
- **Component Lifecycle**: connectedCallback, disconnectedCallback, renderedCallback, errorCallback
- **Security**: Locker Service provides secure component isolation

### Component Structure Best Practices

#### 1. **File Organization**
```
myComponent/
├── myComponent.js          // Component logic
├── myComponent.html        // Template
├── myComponent.css         // Styles (optional)
├── myComponent.js-meta.xml // Metadata
└── __tests__/             // Jest tests
    └── myComponent.test.js
```

#### 2. **Import Patterns**
```javascript
// Standard LWC imports
import { LightningElement, api, track, wire } from 'lwc';

// Apex method imports
import getContactList from '@salesforce/apex/ContactController.getContactList';

// Schema imports (static - preferred for referential integrity)
import CONTACT_OBJECT from '@salesforce/schema/Contact';
import NAME_FIELD from '@salesforce/schema/Contact.Name';

// LDS imports
import { getRecord, createRecord, updateRecord, deleteRecord } from 'lightning/uiRecordApi';
```

### Data Access Patterns

#### 1. **Apex Integration (@wire vs Imperative)**
**Preferred: @wire approach (reactive)**
```javascript
@wire(getContactList)
contacts;

// With parameters
@wire(getContactsByAccount, { accountId: '$recordId' })
wiredContacts({ error, data }) {
    if (data) {
        this.contacts = data;
    } else if (error) {
        this.error = error;
    }
}
```

**When to use Imperative:**
- User-triggered actions (button clicks)
- No reactive parameters needed
```javascript
async handleButtonClick() {
    try {
        const result = await createContact({ contact: this.contactData });
        // Handle success
    } catch (error) {
        // Handle error
    }
}
```

#### 2. **Lightning Data Service (LDS) Patterns**
**Preferred for CRUD operations over custom Apex:**
```javascript
// Create record
import { createRecord } from 'lightning/uiRecordApi';

async createContact() {
    const fields = {
        'FirstName': this.firstName,
        'LastName': this.lastName,
        'Email': this.email
    };
    
    try {
        await createRecord({ apiName: 'Contact', fields });
        // LDS automatically updates cache
    } catch (error) {
        this.handleError(error);
    }
}
```

**Form Component Hierarchy (in order of preference):**
1. `lightning-record-form` - Fastest, automatic layout
2. `lightning-record-edit-form` / `lightning-record-view-form` - More control
3. `@wire(getRecord)` - Maximum control, custom UI

#### 3. **Schema Reference Patterns**
**Static Schema (Preferred - Referential Integrity):**
```javascript
import CONTACT_OBJECT from '@salesforce/schema/Contact';
import NAME_FIELD from '@salesforce/schema/Contact.Name';

// Benefits: Compile-time validation, deletion protection, package inclusion
```

**Dynamic Schema (Object-Agnostic Components):**
```javascript
// Loose coupling but no referential integrity
const fields = ['Contact.Name', 'Contact.Email'];
```

### Component Communication Patterns

#### 1. **Parent-to-Child Communication**
```javascript
// Child component
export default class ChildComponent extends LightningElement {
    @api recordId;           // Public property
    @api message;

    @api refresh() {         // Public method
        // Refresh logic
    }
}

// Parent template
<c-child-component record-id={recordId} message={message}></c-child-component>
```

#### 2. **Child-to-Parent Communication**
```javascript
// Child dispatches custom event
handleButtonClick() {
    const event = new CustomEvent('contact-selected', {
        detail: { contactId: this.contactId, contactName: this.contactName },
        bubbles: false,      // Preferred: false
        composed: false      // Preferred: false
    });
    this.dispatchEvent(event);
}

// Parent handles event
<c-child-component oncontact-selected={handleContactSelected}></c-child-component>
```

#### 3. **Sibling Communication (Pub-Sub Pattern)**
**Use only for sibling components in flexipages - NOT for parent/child:**
```javascript
// Publisher
import { publish, MessageContext } from 'lightning/messageService';
import CONTACT_SELECTED_CHANNEL from '@salesforce/messageChannel/ContactSelected__c';

@wire(MessageContext)
messageContext;

publishContactSelection() {
    const payload = { contactId: this.selectedContactId };
    publish(this.messageContext, CONTACT_SELECTED_CHANNEL, payload);
}

// Subscriber
import { subscribe, MessageContext } from 'lightning/messageService';

connectedCallback() {
    this.subscription = subscribe(
        this.messageContext,
        CONTACT_SELECTED_CHANNEL,
        (message) => this.handleContactSelected(message)
    );
}
```

### Error Handling Patterns

#### 1. **In-Place Error Display (Preferred)**
```html
<template>
    <template if:true={contacts.data}>
        <!-- Component content -->
    </template>
    
    <template if:true={contacts.error}>
        <c-error-panel errors={contacts.error}></c-error-panel>
    </template>
</template>
```

#### 2. **Consistent Error Processing**
```javascript
handleError(error) {
    // Normalize error format
    let message = 'Unknown error';
    if (Array.isArray(error.body)) {
        message = error.body.map(e => e.message).join(', ');
    } else if (typeof error.body.message === 'string') {
        message = error.body.message;
    }
    
    this.error = { message };
}
```

### Performance & Security Best Practices

#### 1. **Performance Optimizations**
- **Use @wire over imperative calls** when possible
- **Implement conditional rendering** with `if:true` and `if:false`
- **Minimize DOM queries** - cache template references
- **Use async/await** over promise chains
- **Implement proper lifecycle management**

#### 2. **Security Practices**
- **Field-Level Security**: Use `WITH USER_MODE` in Apex
- **Object Permissions**: Check CRUD permissions
- **CSP Compliance**: No inline scripts, use static resources
- **Data Validation**: Server-side validation in Apex

### Testing Patterns

#### 1. **Jest Testing Structure**
```javascript
import { createElement } from 'lwc';
import MyComponent from 'c/myComponent';

describe('c-my-component', () => {
    afterEach(() => {
        // Clean up DOM
        while (document.body.firstChild) {
            document.body.removeChild(document.body.firstChild);
        }
    });

    it('renders component correctly', () => {
        const element = createElement('c-my-component', {
            is: MyComponent
        });
        document.body.appendChild(element);
        
        // Assertions
        expect(element.shadowRoot.querySelector('.my-class')).toBeTruthy();
    });
});
```

#### 2. **Mock Patterns**
```javascript
// Mock Apex methods
jest.mock('@salesforce/apex/ContactController.getContactList', () => ({
    default: jest.fn()
}), { virtual: true });

// Mock wire services
import { getRecord } from 'lightning/uiRecordApi';
jest.mock('lightning/uiRecordApi', () => ({
    getRecord: jest.fn()
}), { virtual: true });
```

### Aura-LWC Interoperability

#### 1. **Aura Containing LWC**
```javascript
// LWC dispatches DOM events to Aura parent
this.dispatchEvent(new CustomEvent('lwcaction', {
    detail: { data: this.componentData },
    bubbles: true
}));
```

#### 2. **Coexistence Patterns**
- Use Lightning Message Service for cross-framework communication
- Maintain consistent error handling patterns
- Share utility classes via static resources

### Migration Considerations

#### 1. **From Aura to LWC**
- **Events**: Replace application events with Lightning Message Service
- **Attributes**: Convert to @api properties
- **Methods**: Convert to @api methods
- **Data Binding**: Implement one-way flow patterns

#### 2. **Legacy Integration**
- **Visualforce**: Use postMessage for communication
- **Classic**: Provide Lightning-ready alternatives
- **Mixed Environments**: Plan gradual migration strategy

### Development Lifecycle

#### 1. **Local Development**
```bash
# Setup LWC local development
npm install -g @salesforce/lwc-dev-server
lwc-dev-server
```

#### 2. **Deployment Patterns**
- Use source format for scratch orgs
- Convert to metadata format for traditional orgs
- Implement CI/CD with SFDX commands

### Common Anti-Patterns to Avoid

1. **DON'T use pubsub for parent-child communication**
2. **DON'T bypass component encapsulation**
3. **DON'T perform DML in component constructors**
4. **DON'T use imperative Apex when @wire is appropriate**
5. **DON'T ignore error handling**
6. **DON'T create memory leaks in event listeners**
7. **DON'T violate CSP with inline scripts**

### Implementation Checklist

- [ ] Follow naming conventions (camelCase, descriptive)
- [ ] Implement proper error handling
- [ ] Use appropriate data access pattern (@wire vs imperative)
- [ ] Follow security best practices
- [ ] Write comprehensive Jest tests
- [ ] Document component APIs
- [ ] Optimize for performance
- [ ] Ensure accessibility compliance
- [ ] Validate cross-browser compatibility
- [ ] Plan for mobile responsiveness

## External API Integration Patterns

### Core API Integration Principles
- **Generated Client Code**: Use openapi-generator for auto-generated, type-safe API clients
- **Separation of Concerns**: Keep API logic separate from business logic
- **Error Handling**: Implement robust error handling for external service failures
- **Caching Strategies**: Cache responses where appropriate to optimize performance
- **Security**: Handle authentication tokens and API keys securely

### OpenAPI/Swagger Code Generation

#### 1. **Setting Up openapi-generator**
```bash
# Install via Homebrew (macOS)
brew install openapi-generator

# Install via npm (cross-platform)
npm install @openapitools/openapi-generator-cli -g

# Generate Apex client from OpenAPI spec
openapi-generator generate \
  -i https://api.example.com/swagger.yaml \
  -g apex \
  -o /tmp/api_client/ \
  --additional-properties=clientPackage=MyAPIClient
```

#### 2. **Generated Code Structure**
```
generated-apex/
├── force-app/main/default/classes/
│   ├── OASClient.cls              # Base HTTP client
│   ├── OASPetApi.cls             # API endpoint wrapper
│   ├── OASPet.cls                # Data model classes
│   ├── OASApiResponse.cls        # Response wrapper
│   └── OASConfiguration.cls      # Client configuration
```

#### 3. **Integration with Enterprise Patterns**
```apex
// Service Layer - External API Integration
public with sharing class PetService {
    private static final OASClient client = new OASClient();
    private static final OASPetApi petApi = new OASPetApi(client);
    
    /**
     * Retrieves pet information from external API
     * Integrates with Domain layer for validation
     */
    public static Pet retrievePetById(Id petId) {
        try {
            Map<String, Object> params = new Map<String, Object>{
                'petId' => Long.valueOf(String.valueOf(petId))
            };
            
            OASPet externalPet = petApi.getPetById(params);
            
            // Convert external model to internal domain model
            Pet internalPet = convertToDomainModel(externalPet);
            
            // Apply domain validation rules
            PetDomain domain = new PetDomain(new List<Pet>{ internalPet });
            domain.validateExternalData();
            
            return internalPet;
            
        } catch (Exception e) {
            throw new ExternalServiceException('Failed to retrieve pet: ' + e.getMessage());
        }
    }
    
    private static Pet convertToDomainModel(OASPet externalPet) {
        Pet pet = new Pet();
        pet.Name = externalPet.name;
        pet.Status__c = externalPet.status;
        pet.External_Id__c = String.valueOf(externalPet.id);
        return pet;
    }
}
```

### API Client Configuration Patterns

#### 1. **Secure Configuration Management**
```apex
public with sharing class APIConfiguration {
    // Use Custom Settings or Custom Metadata for configuration
    private static API_Configuration__c config = API_Configuration__c.getOrgDefaults();
    
    public static void configureClient(OASClient client) {
        // Set base URL from configuration
        client.setBasePath(config.Base_URL__c);
        
        // Configure authentication
        if (config.Auth_Type__c == 'Bearer') {
            client.setDefaultHeaders(new Map<String, String>{
                'Authorization' => 'Bearer ' + getBearerToken()
            });
        } else if (config.Auth_Type__c == 'ApiKey') {
            client.setDefaultHeaders(new Map<String, String>{
                'X-API-Key' => getApiKey()
            });
        }
        
        // Set timeout configuration
        client.setTimeout(Integer.valueOf(config.Timeout_Seconds__c) * 1000);
    }
    
    private static String getBearerToken() {
        // Implement secure token retrieval
        // Consider using Named Credentials or Protected Custom Settings
        return config.Bearer_Token__c;
    }
    
    private static String getApiKey() {
        // Implement secure API key retrieval
        return config.API_Key__c;
    }
}
```

#### 2. **Error Handling and Retry Logic**
```apex
public with sharing class APIClientWrapper {
    private static final Integer MAX_RETRIES = 3;
    private static final Integer RETRY_DELAY_MS = 1000;
    
    public static Object makeAPICall(APICallStrategy strategy) {
        Integer attempts = 0;
        Exception lastException;
        
        while (attempts < MAX_RETRIES) {
            try {
                return strategy.execute();
            } catch (CalloutException e) {
                lastException = e;
                attempts++;
                
                // Implement exponential backoff
                if (attempts < MAX_RETRIES) {
                    // Note: Apex doesn't support Thread.sleep, this is pseudocode
                    // In practice, you might queue a retry job
                    System.debug('Retrying API call, attempt: ' + attempts);
                }
            }
        }
        
        throw new ExternalServiceException(
            'API call failed after ' + MAX_RETRIES + ' attempts: ' + lastException.getMessage()
        );
    }
}

// Strategy pattern for different API calls
public interface APICallStrategy {
    Object execute();
}

public class GetPetStrategy implements APICallStrategy {
    private Id petId;
    
    public GetPetStrategy(Id petId) {
        this.petId = petId;
    }
    
    public Object execute() {
        OASClient client = new OASClient();
        APIConfiguration.configureClient(client);
        OASPetApi api = new OASPetApi(client);
        
        Map<String, Object> params = new Map<String, Object>{
            'petId' => Long.valueOf(String.valueOf(petId))
        };
        
        return api.getPetById(params);
    }
}
```

### Response Caching Patterns

#### 1. **Platform Cache Integration**
```apex
public with sharing class CachedAPIService {
    private static final String CACHE_PARTITION = 'local.APICache';
    private static final Integer DEFAULT_TTL = 300; // 5 minutes
    
    public static OASPet getCachedPet(Id petId) {
        String cacheKey = 'pet_' + petId;
        
        // Check cache first
        OASPet cachedPet = (OASPet) Cache.get(CACHE_PARTITION + '.' + cacheKey);
        
        if (cachedPet != null) {
            return cachedPet;
        }
        
        // Cache miss - fetch from API
        OASPet pet = fetchPetFromAPI(petId);
        
        // Store in cache
        Cache.put(CACHE_PARTITION + '.' + cacheKey, pet, DEFAULT_TTL);
        
        return pet;
    }
    
    private static OASPet fetchPetFromAPI(Id petId) {
        return (OASPet) APIClientWrapper.makeAPICall(new GetPetStrategy(petId));
    }
}
```

#### 2. **Custom Object Caching for Persistence**
```apex
public with sharing class PersistentAPICache {
    public static OASPet getCachedPet(Id petId) {
        // Check persistent cache
        List<API_Cache__c> cacheRecords = [
            SELECT Data__c, Cached_At__c 
            FROM API_Cache__c 
            WHERE Cache_Key__c = :('pet_' + petId)
            AND Cached_At__c > :System.now().addMinutes(-5)
            LIMIT 1
        ];
        
        if (!cacheRecords.isEmpty()) {
            return (OASPet) JSON.deserialize(cacheRecords[0].Data__c, OASPet.class);
        }
        
        // Fetch from API and cache
        OASPet pet = fetchPetFromAPI(petId);
        
        // Store in persistent cache
        API_Cache__c cacheRecord = new API_Cache__c(
            Cache_Key__c = 'pet_' + petId,
            Data__c = JSON.serialize(pet),
            Cached_At__c = System.now()
        );
        
        insert cacheRecord;
        return pet;
    }
}
```

### Bulk API Operations

#### 1. **Batch Processing Pattern**
```apex
public with sharing class BulkAPIProcessor {
    private static final Integer BATCH_SIZE = 10;
    
    public static List<OASPet> processPetsBatch(List<Id> petIds) {
        List<OASPet> results = new List<OASPet>();
        
        // Process in batches to respect API limits
        List<List<Id>> batches = createBatches(petIds, BATCH_SIZE);
        
        for (List<Id> batch : batches) {
            results.addAll(processSingleBatch(batch));
        }
        
        return results;
    }
    
    private static List<OASPet> processSingleBatch(List<Id> petIds) {
        List<OASPet> batchResults = new List<OASPet>();
        
        for (Id petId : petIds) {
            try {
                OASPet pet = CachedAPIService.getCachedPet(petId);
                batchResults.add(pet);
            } catch (Exception e) {
                // Log error but continue processing
                System.debug('Failed to process pet ' + petId + ': ' + e.getMessage());
            }
        }
        
        return batchResults;
    }
    
    private static List<List<Id>> createBatches(List<Id> items, Integer batchSize) {
        List<List<Id>> batches = new List<List<Id>>();
        List<Id> currentBatch = new List<Id>();
        
        for (Id item : items) {
            currentBatch.add(item);
            
            if (currentBatch.size() >= batchSize) {
                batches.add(currentBatch);
                currentBatch = new List<Id>();
            }
        }
        
        if (!currentBatch.isEmpty()) {
            batches.add(currentBatch);
        }
        
        return batches;
    }
}
```

### Testing External API Integrations

#### 1. **Mock Frameworks for Generated Clients**
```apex
@isTest
public class MockOASPetApi extends OASPetApi {
    private Map<Id, OASPet> mockData = new Map<Id, OASPet>();
    
    public MockOASPetApi() {
        super(null); // No real client needed for mocking
        setupMockData();
    }
    
    public override OASPet getPetById(Map<String, Object> params) {
        Long petId = (Long) params.get('petId');
        Id salesforcePetId = Id.valueOf(String.valueOf(petId).leftPad(15, '0'));
        
        if (mockData.containsKey(salesforcePetId)) {
            return mockData.get(salesforcePetId);
        }
        
        throw new CalloutException('Pet not found: ' + petId);
    }
    
    private void setupMockData() {
        OASPet mockPet = new OASPet();
        mockPet.id = 1;
        mockPet.name = 'Test Pet';
        mockPet.status = 'available';
        
        mockData.put('001000000000001', mockPet);
    }
}

@isTest
public class PetServiceTest {
    @isTest
    static void testRetrievePetById() {
        // Setup mock
        PetService.petApi = new MockOASPetApi();
        
        // Test
        Test.startTest();
        Pet result = PetService.retrievePetById('001000000000001');
        Test.stopTest();
        
        // Verify
        System.assertEquals('Test Pet', result.Name);
        System.assertEquals('available', result.Status__c);
    }
}
```

### Integration with Lightning Web Components

#### 1. **Apex Controller for LWC**
```apex
public with sharing class PetController {
    @AuraEnabled(cacheable=true)
    public static List<Pet> getAvailablePets() {
        try {
            // Use Service layer for external API calls
            return PetService.getAvailablePets();
        } catch (Exception e) {
            throw new AuraHandledException('Unable to retrieve pets: ' + e.getMessage());
        }
    }
    
    @AuraEnabled
    public static Pet updatePetStatus(Id petId, String newStatus) {
        try {
            return PetService.updatePetStatus(petId, newStatus);
        } catch (Exception e) {
            throw new AuraHandledException('Unable to update pet: ' + e.getMessage());
        }
    }
}
```

#### 2. **LWC Component with API Integration**
```javascript
import { LightningElement, wire } from 'lwc';
import getAvailablePets from '@salesforce/apex/PetController.getAvailablePets';
import updatePetStatus from '@salesforce/apex/PetController.updatePetStatus';

export default class PetManager extends LightningElement {
    @wire(getAvailablePets)
    pets;
    
    async handleStatusUpdate(event) {
        try {
            const petId = event.target.dataset.petId;
            const newStatus = event.target.value;
            
            await updatePetStatus({ petId, newStatus });
            
            // Refresh data
            return refreshApex(this.pets);
        } catch (error) {
            this.handleError(error);
        }
    }
}
```

### Performance and Governor Limit Considerations

#### 1. **Asynchronous Processing**
```apex
public class AsyncAPIProcessor implements Queueable, Database.AllowsCallouts {
    private List<Id> petIds;
    
    public AsyncAPIProcessor(List<Id> petIds) {
        this.petIds = petIds;
    }
    
    public void execute(QueueableContext context) {
        try {
            List<OASPet> pets = BulkAPIProcessor.processPetsBatch(petIds);
            
            // Process results
            processPetResults(pets);
            
            // Chain additional processing if needed
            if (hasMoreWork()) {
                System.enqueueJob(new AsyncAPIProcessor(getNextBatch()));
            }
        } catch (Exception e) {
            // Log error and potentially retry
            System.debug('Async API processing failed: ' + e.getMessage());
        }
    }
    
    private void processPetResults(List<OASPet> pets) {
        // Convert to Salesforce records and process through Domain layer
        List<Pet> salesforcePets = convertToSalesforceRecords(pets);
        
        // Use Unit of Work for bulk DML
        fflib_SObjectUnitOfWork uow = Application.UnitOfWork.newInstance();
        
        for (Pet pet : salesforcePets) {
            uow.registerUpsert(pet);
        }
        
        uow.commitWork();
    }
}
```

### Common Anti-Patterns to Avoid

1. **DON'T make API calls in loops without bulkification**
2. **DON'T ignore API rate limits and timeouts**
3. **DON'T store sensitive API credentials in plain text**
4. **DON'T skip error handling for external service failures**
5. **DON'T mix external API logic with business logic**
6. **DON'T forget to implement proper caching strategies**
7. **DON'T ignore API versioning and backward compatibility**

### Implementation Checklist

- [ ] Generate client code using openapi-generator
- [ ] Implement secure configuration management
- [ ] Add comprehensive error handling and retry logic
- [ ] Implement appropriate caching strategy
- [ ] Create proper test coverage with mocks
- [ ] Integrate with Enterprise Pattern Service layer
- [ ] Consider asynchronous processing for bulk operations
- [ ] Monitor API usage and performance metrics
- [ ] Document API dependencies and SLAs
- [ ] Plan for API versioning and changes

## Traditional Triggers & Workflow Migration 