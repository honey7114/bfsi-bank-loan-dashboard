# DAX Measures Reference

All 20 measures used in the BFSI Bank Loan Performance Dashboard, grouped by category. Built on a Star Schema with `LoanApplications` as the fact table, related to `LoanOfficerMaster`, `BranchMaster`, and `DateTable`.

## Core Aggregations

```dax
Total Loan Amount = SUM('LoanApplications'[Loan Amount])

Total Applications = COUNT('LoanApplications'[Loan ID])

Total Approved = SUM('LoanApplications'[Approved])

Total Defaulted = SUM('LoanApplications'[Defaulted])

Total EMI = SUM('LoanApplications'[EMI Amount])

Avg CIBIL Score = AVERAGE('LoanApplications'[CIBIL Score])
```

## Business KPIs

```dax
Approval Rate % = DIVIDE([Total Approved], [Total Applications], 0)

Default Rate % = DIVIDE([Total Defaulted], [Total Approved], 0)

Avg Loan Amount = DIVIDE([Total Loan Amount], [Total Applications], 0)
```

## Time Intelligence

Requires a dedicated Date table created with `CALENDAR()` and connected to the fact table via a one-to-many relationship on date.

```dax
DateTable = CALENDAR(
    MIN('LoanApplications'[Application Date]),
    MAX('LoanApplications'[Application Date])
)

Loan Amount MTD = TOTALMTD([Total Loan Amount], 'DateTable'[Date])

Loan Amount YTD = TOTALYTD([Total Loan Amount], 'DateTable'[Date])

Loan Amount Prev Month = CALCULATE([Total Loan Amount], PREVIOUSMONTH('DateTable'[Date]))

MOM Growth % = 
DIVIDE(
    [Total Loan Amount] - [Loan Amount Prev Month],
    [Loan Amount Prev Month],
    0
)

Loan Amount Prev Year = CALCULATE([Total Loan Amount], SAMEPERIODLASTYEAR('DateTable'[Date]))

YOY Growth % = 
DIVIDE(
    [Total Loan Amount] - [Loan Amount Prev Year],
    [Loan Amount Prev Year],
    0
)
```

## Filter Context Manipulation

These three measures answer different versions of "what % does this represent" depending on whether the comparison base should be fixed, partially fixed, or fully dynamic.

```dax
// Always compares to the full portfolio total, ignoring every filter on the report
Loan % of Total = 
DIVIDE(
    [Total Loan Amount],
    CALCULATE([Total Loan Amount], ALL('LoanApplications')),
    0
)

// Always compares within the selected Region, regardless of any other filter applied
Approval Rate Within Region = 
DIVIDE(
    [Total Approved],
    CALCULATE([Total Applications], ALLEXCEPT('LoanApplications', 'LoanApplications'[Region])),
    0
)

// Dynamically compares to whatever combination of filters/slicers the user currently has selected
Loan % of Selected = 
DIVIDE(
    [Total Loan Amount],
    CALCULATE([Total Loan Amount], ALLSELECTED('LoanApplications')),
    0
)
```

## Rolling Average

```dax
Rolling 3M Avg = 
CALCULATE(
    [Total Loan Amount],
    DATESINPERIOD(
        'DateTable'[Date],
        LASTDATE('DateTable'[Date]),
        -3,
        MONTH
    )
)
```

## Notes

- `ALL()` vs `ALLEXCEPT()` vs `ALLSELECTED()` is the most conceptually demanding part of this measure set. The short version: `ALL()` ignores every filter for a fixed grand-total comparison; `ALLEXCEPT()` hardcodes one column to keep while dropping every other filter; `ALLSELECTED()` respects whatever the user has actively selected on slicers, making it the right choice for dashboards where the comparison base should move with the user's selection.
- All Time Intelligence measures depend on a continuous, unbroken Date table — not the transaction date column directly — since the fact table only contains dates where a transaction occurred.
