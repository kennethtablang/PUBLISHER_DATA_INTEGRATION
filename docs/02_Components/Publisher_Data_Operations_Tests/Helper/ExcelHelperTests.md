# Tests — Helper Excel Helper Tests (analysis)

## One-line purpose

Test suite for ExcelHelper business logic validation focusing on brand new fund determination using mock dependencies and complex data-driven scenarios with Fund Fact document age status classification.

---

## Files analyzed

* `Publisher_Data_Operations_Tests/Helper/ExcelHelperTests.cs`

---

## What this code contains (high level)

1. **Mock Infrastructure Setup** — Comprehensive Moq framework usage with strict mock behavior for PDIStream and Logger dependencies
2. **Brand New Fund Logic Testing** — Business rule validation for determining fund launch status based on filing references and document age classifications
3. **Complex Test Data Scenarios** — Data-driven testing with realistic ETF launch scenarios and various fund document combinations
4. **Custom Test Data Providers** — Sophisticated test data generation using IEnumerable<object[]> pattern with DataTable filtering
5. **FFDocAge Integration** — Testing integration with Fund Fact document age enumeration for regulatory compliance

This test suite validates critical business logic for fund classification that drives regulatory reporting and document generation workflows.

---

## Classes, properties and methods (developer reference)

### `ExcelHelperTests` - Main Test Class

**Purpose:** Tests ExcelHelper business logic methods using mocked dependencies to isolate unit testing from external dependencies.

#### Test Infrastructure Setup

##### Mock Framework Configuration

```csharp
private MockRepository mockRepository;
private Mock<PDIStream> mockPDIStream;
private Mock<Logger> mockLogger;

public ExcelHelperTests()
{
    this.mockRepository = new MockRepository(MockBehavior.Strict);
    this.mockPDIStream = this.mockRepository.Create<PDIStream>();
    this.mockLogger = this.mockRepository.Create<Logger>();
}
```

**Mock Strategy:**

* **Strict Behavior:** `MockBehavior.Strict` ensures all mock interactions must be explicitly configured
* **Repository Pattern:** `MockRepository` provides centralized mock management
* **Dependency Injection:** Mocks key dependencies (PDIStream, Logger) for isolated testing

##### System Under Test Factory

```csharp
private ExcelHelper CreateExcelHelper()
{
    return new ExcelHelper(
        this.mockPDIStream.Object,
        this.mockLogger.Object);
}
```

**Factory Pattern:** Creates ExcelHelper instances with mocked dependencies for consistent test setup.

---

## Core Business Logic Testing

### Brand New Fund Determination Tests

#### `[Theory] void isBrandNewFundTests(DataTable dt, string filingID, bool expected)`

**Purpose:** Validates complex business logic for determining whether a fund qualifies as "brand new" based on multiple data criteria.

**Test Method Pattern:**

```csharp
[Theory]
[ClassData(typeof(ExcelHelperTestData))]
public void isBrandNewFundTests(DataTable dt, string filingID, bool expected)
{
    Assert.Equal(expected, ExcelHelper.IsBrandNewFund(dt, filingID));
}
```

**Static Method Testing:** Tests static method `ExcelHelper.IsBrandNewFund()` which suggests utility/helper method for business logic validation.

---

### Complex Test Data Provider

#### `ExcelHelperTestData` - Custom Test Data Class

**Purpose:** Implements sophisticated test data generation using realistic ETF launch scenarios with multiple fund document combinations.

**Implementation Pattern:**

```csharp
public class ExcelHelperTestData : IEnumerable<object[]>
{
    DataTable dt = SetupTable();
    
    public IEnumerator<object[]> GetEnumerator()
    {
        // Complex yield return scenarios with DataTable filtering
    }
}
```

**Test Scenario Analysis:**

##### 1. Brand New Series Scenario (Returns False)

```csharp
yield return new object[] {
    dt.Select("Document_Number = '31635FSEM' AND FundCode = '31635'").CopyToDataTable(),
    "ETFLAUNCH2021JUNE1",
    false 
};
```

**Data Context:**

* **Document:** 31635FSEM (FSEM suffix)
* **Fund Code:** 31635
* **Filing Reference:** ETFLAUNCH2021JUNE1
* **FFDocAge Status:** BrandNewSeries
* **Expected Result:** `false` (not brand new fund, just brand new series)

**Business Logic Inference:** A "BrandNewSeries" is different from a "BrandNewFund" - suggests funds can have new series without being entirely new funds.

##### 2. New Fund with Null Values (Returns True)

```csharp
yield return new object[] {
    dt.Select("Document_Number = '31636FS' AND FundCode = '31636'").CopyToDataTable(),
    "ETFLAUNCH2021JUNE2",
    true
};
```

**Data Context:**

* **Document:** 31636FS (FS suffix)
* **Fund Code:** 31636
* **Filing Reference:** ETFLAUNCH2021JUNE2
* **FFDocAge Status:** null (no previous status)
* **Expected Result:** `true` (qualifies as brand new fund)

**Business Logic Inference:** Funds with no previous document filing status qualify as brand new.

##### 3. Brand New Fund with Document Filing Reference (Returns True)

```csharp
yield return new object[] {
    dt.Select("Document_Number = '31637FSC' AND FundCode = '31637'").CopyToDataTable(),
    "ETFLAUNCH2021JUNE3",
    true
};
```

**Data Context:**

* **Document:** 31637FSC (FSC suffix)
* **Both DocFilingReferenceID and FilingReferenceID:** ETFLAUNCH2021JUNE3
* **Both DocFFDocAgeStatusID and FFDocAgeStatusID:** BrandNewFund
* **Expected Result:** `true` (confirmed brand new fund)

**Business Logic Inference:** Consistent document and filing references with BrandNewFund status confirm new fund status.

##### 4. Brand New Fund with Null Document Reference (Returns True)

```csharp
yield return new object[] {
    dt.Select("Document_Number = '31638FSB' AND FundCode = '31638'").CopyToDataTable(),
    "ETFLAUNCH2021JUNE4",
    true
};
```

**Data Context:**

* **Document:** 31638FSB (FSB suffix)
* **DocFilingReferenceID:** null
* **FilingReferenceID:** ETFLAUNCH2021JUNE4
* **FFDocAgeStatusID:** BrandNewFund
* **Expected Result:** `true` (brand new fund despite null document reference)

##### 5. Conflicting Status Scenario (Returns False)

```csharp
yield return new object[] {
    dt.Select("Document_Number = '31639SB' AND FundCode = '31639'").CopyToDataTable(),
    "ETFLAUNCH2021JUNE5",
    false
};
```

**Data Context:**

* **Document:** 31639SB (SB suffix)
* **DocFilingReferenceID:** "SomethingElsee" (different from filing ID)
* **DocFFDocAgeStatusID:** BrandNewFund
* **FilingReferenceID:** "SomethingElsee"
* **FFDocAgeStatusID:** NewSeries (not BrandNewFund)
* **Expected Result:** `false` (conflicting status prevents brand new fund classification)

**Business Logic Inference:** Conflicting document age statuses (BrandNewFund vs NewSeries) result in conservative classification.

---

### Test Data Structure Analysis

#### `SetupTable()` - Master Test Data Setup

**DataTable Schema:**

```csharp
dt.Columns.Add("Document_Number", Type.GetType("System.String"));
dt.Columns.Add("FundCode", Type.GetType("System.String"));
dt.Columns.Add("DocFilingReferenceID", Type.GetType("System.String"));
dt.Columns.Add("DocFFDocAgeStatusID", typeof(FFDocAge));
dt.Columns.Add("FilingReferenceID", Type.GetType("System.String"));
dt.Columns.Add("FFDocAgeStatusID", typeof(FFDocAge));
```

**Column Analysis:**

* **Document_Number:** Fund document identifier with suffix codes (FSEM, FS, FSC, FSB, SB)
* **FundCode:** Core fund identifier (matches document number prefix)
* **DocFilingReferenceID:** Document-specific filing reference (can be null)
* **DocFFDocAgeStatusID:** Document-specific Fund Fact age status (FFDocAge enum)
* **FilingReferenceID:** General filing reference identifier
* **FFDocAgeStatusID:** General Fund Fact age status (FFDocAge enum)

**Dual Reference System:** The presence of both Doc* and general columns suggests the system tracks both document-specific and general fund filing information.

**Test Data Scenarios:**

1. **31635FSEM:** BrandNewSeries with filing reference
2. **31636FS:** All null values (new fund scenario)
3. **31637FSC:** Consistent BrandNewFund across both systems
4. **31638FSB:** BrandNewFund with partial null values
5. **31639SB:** Conflicting statuses (BrandNewFund vs NewSeries)

---

## Business logic and integration patterns

### Fund Classification Business Rules

The tests validate complex fund classification logic:

**Brand New Fund Criteria (Inferred):**

1. **Null Status Check:** Funds with no previous filing status qualify
2. **Consistent Status Validation:** Both document and general status must align
3. **Filing Reference Matching:** Filing references should be consistent
4. **Series vs Fund Distinction:** BrandNewSeries ≠ BrandNewFund
5. **Conservative Classification:** Conflicting data results in negative classification

### ETF Launch Scenario Modeling

The test data models realistic ETF launch scenarios:

**Launch Campaign Structure:**

* **Sequential Launch IDs:** ETFLAUNCH2021JUNE1 through ETFLAUNCH2021JUNE5
* **Multiple Fund Types:** Various document suffixes (FSEM, FS, FSC, FSB, SB)
* **Coordinated Timing:** All funds launched in June 2021 campaign

**Document Type Classification:**

* **FS Variants:** FS, FSC, FSB, FSEM (different fund series or types)
* **SB Type:** Different document classification (possibly simplified prospectus)

### Regulatory Compliance Integration

Fund classification drives regulatory requirements:

**FFDocAge Enumeration Usage:**

* **BrandNewFund:** Completely new fund requiring full disclosure
* **BrandNewSeries:** New series of existing fund with different requirements
* **NewSeries:** Standard new series classification

**Filing Reference Tracking:**

* Document-level and fund-level filing reference management
* Consistency validation across multiple reference systems
* Support for campaign-based fund launches

---

## Technical implementation considerations

### Mock Framework Usage

**Strict Mock Behavior:**

* Ensures all dependencies are explicitly configured
* Prevents accidental dependency interactions during testing
* Validates that tested methods use only intended dependencies

**Dependency Isolation:**

* PDIStream and Logger dependencies mocked for unit testing
* Business logic tested independently of external systems
* Clean separation of concerns for maintainable tests

### Test Data Management Strategy

**DataTable-Based Testing:**

* Realistic data structures matching production schemas
* Complex filtering and selection capabilities
* Type-safe column definitions including custom enums

**Data-Driven Test Approach:**

* Single test method validates multiple business scenarios
* Custom test data providers enable complex scenario modeling
* Maintainable test data through centralized setup methods

### Performance Characteristics

* **In-Memory Testing:** DataTable operations performed in memory
* **Static Method Testing:** No instance creation overhead for utility methods
* **Mock Framework Overhead:** Moq framework adds some performance cost but enables isolated testing

---

## Test coverage and quality assurance

### Business Logic Coverage

**Fund Classification Scenarios:**

* New fund with no previous history
* Brand new fund with complete documentation
* Brand new series (different from brand new fund)
* Conflicting status scenarios
* Partial data scenarios with null values

**Data Quality Scenarios:**

* Complete data sets with all fields populated
* Partial data sets with strategic null values
* Conflicting data between document and general systems
* Filing reference consistency validation

### Edge Case Coverage

**Null Handling:**

* Various combinations of null values in different fields
* Graceful handling of missing filing references
* Default behavior for undefined document age statuses

**Data Consistency:**

* Matching vs. non-matching filing references
* Consistent vs. conflicting document age statuses
* Document number and fund code relationship validation

---

## Potential enhancements

### Test Coverage Expansion

1. **Exception Scenarios:** Test invalid DataTable structures or malformed data
2. **Performance Testing:** Validate performance with large DataTable datasets
3. **Boundary Conditions:** Test edge cases like empty DataTables or invalid enum values
4. **Integration Testing:** Test with real PDIStream and Logger implementations

### Mock Framework Enhancements

1. **Behavior Verification:** Verify expected interactions with mocked dependencies
2. **Setup Validation:** Ensure all mock setups are actually used during testing
3. **State-Based Testing:** Add stateful mock scenarios for more complex testing
4. **Async Testing:** Support for asynchronous ExcelHelper method testing

### Test Data Improvements

1. **External Test Data:** Load test scenarios from JSON or XML files
2. **Generated Test Data:** Programmatically generate test scenarios for broader coverage
3. **Real Data Integration:** Use sanitized production data for more realistic testing
4. **Scenario Documentation:** Better documentation of business rules being tested

---

## Action items for system maintenance

1. **Business Rule Documentation:** Document the complete fund classification business rules
2. **Test Data Synchronization:** Keep test data current with production fund types and scenarios
3. **Mock Framework Updates:** Monitor Moq framework updates and compatibility
4. **Performance Monitoring:** Track test execution time as business logic complexity grows
5. **Integration Validation:** Ensure mocked behavior matches real dependency behavior
6. **Regulatory Compliance:** Validate that tested business rules meet current regulatory requirements
