# Methods

## Data Source

This project uses a historical CSV extract from Metro Nashville’s hubNashville service-request system. The repository includes the analyzed dataset for reproducibility.

The source contains request identifiers, dates, complaint categories, descriptions, incident addresses, ZIP codes, trash haulers, collection routes, council districts, and State Plane coordinates.

The data-preparation workflow is organized into two layers:

- **Bronze:** Source ingestion, header promotion, and data typing
- **Silver:** Cleaned fact and dimension tables used by the report

## Missed-Pickup Classification

Missed-pickup requests are identified through a rule-based classifier implemented in Power Query. The classifier evaluates both the request category and complaint description.

It searches for terms such as `missed`, `skip`, `not picked`, `failed`, and `did not`. Case-insensitive comparisons and error handling prevent missing or malformed descriptions from interrupting a refresh.

The core M logic uses list transformations to evaluate multiple keywords:

```powerquery
keywords = {
    "missed",
    "miss",
    "skip",
    "skipped",
    "not picked",
    "failed",
    "fail",
    "forgot",
    "never",
    "did not"
},

isMissed =
    Text.Contains(
        request,
        "Missed",
        Comparer.OrdinalIgnoreCase
    )
    or List.AnyTrue(
        List.Transform(
            keywords,
            each Text.Contains(
                description,
                _,
                Comparer.OrdinalIgnoreCase
            )
        )
    )
```

The resulting `MissedFlag` provides a consistent missed-pickup filter across the report. Because the classification is rule-based, ambiguous or incomplete descriptions may produce false positives or false negatives.

## Address Preparation

Address-preparation rules standardize incident locations before address-level and geographic analysis.

The process includes:

- Converting addresses to uppercase
- Separating street and city information
- Removing punctuation and selected unit identifiers
- Standardizing street suffixes
- Deriving missing cities from ZIP codes where possible
- Replacing malformed values
- Combining the normalized address, city, and ZIP code into an `AddressKey`

A representative composite-key expression is:

```powerquery
Table.AddColumn(
    Source,
    "AddressKey",
    each Text.Combine(
        {
            [Incident Address],
            [City],
            Text.From([Zip Code])
        },
        ","
    )
)
```

The resulting key supports distinct-residence calculations and connects service requests and policy events to the location dimension.

## Contractor Preparation

Contractor-preparation rules standardize trash-hauler names and collection routes before comparison.

The process:

- Standardizes inconsistent hauler names
- Classifies missing haulers as `UNKNOWN`
- Converts blank routes to null
- Combines the hauler and route into `TrashHaulerKey`
- Removes duplicate hauler-and-route combinations

A representative key-building expression is:

```powerquery
Table.AddColumn(
    Source,
    "TrashHaulerKey",
    each [Trash Hauler] & ":" &
        Text.From([Trash Route] ?? "")
)
```

The contractor key prevents routes with the same number but different haulers from being treated as the same operational unit.

### Geographic preparation

A Python step embedded in Power Query converts Tennessee State Plane coordinates to standard longitude and latitude.

```python
transformer = Transformer.from_crs(
    "EPSG:2274",
    "EPSG:4326",
    always_xy=True
)

dataset["Longitude"], dataset["Latitude"] = transformer.transform(
    dataset["State Plan X"],
    dataset["State Plan Y"]
)
```

The converted coordinates provide the geographic fields used by the Azure Maps visual.

## Data Model

The semantic model follows a two-fact dimensional design with shared dimensions.

| Table                  | Purpose                                               |
| ---------------------- | ----------------------------------------------------- |
| `FACT_ServiceRequests` | Complaint and service-request analysis                |
| `FACT_PenaltyEvents`   | Fine-policy scenario outputs                          |
| `DIM_Date`             | Year, month, week, weekday, and prior-year analysis   |
| `DIM_Location`         | Address, ZIP code, council district, and map analysis |
| `DIM_TrashRoute`       | Trash-hauler and collection-route analysis            |
| `PAR_Policy`           | Interactive policy-scenario selection                 |
| `_MeasureTable`        | Central organization of DAX measures                  |

Both fact tables connect to the Date, Location, and Trash Route dimensions.

`PAR_Policy` is a disconnected parameter table. Its selected value determines which policy result is returned by DAX rather than filtering a fact table through a relationship.

This structure supports complaint and financial-scenario analysis without joining the two fact tables directly.

## DAX Measures

DAX measures calculate:

- Distinct complaint totals
- Distinct affected residences
- Distinct complaint days
- Prior-year complaint totals
- Route-level complaint growth
- Policy-scenario totals
- Contractor-specific fine totals

The central scenario measure returns the result associated with the selected policy:

```DAX
Total Fines =
SWITCH(
    SELECTEDVALUE(PAR_Policy[PolicyID], 1),
    1, SUM(FACT_PenaltyEvents[CurrentPolicy]),
    2, SUM(FACT_PenaltyEvents[Alt1Policy]),
    3, SUM(FACT_PenaltyEvents[Alt2Policy])
)
```

Additional measures apply contractor filters while preserving the existing report context. Time-intelligence measures use the shared date dimension to compare complaints with the corresponding prior-year period.

## Report Behavior

The report contains three pages:

- **Overview:** Complaint totals, affected residences, trends, addresses, routes, ZIP codes, and report filters
- **Season:** Monthly and weekday patterns, route growth, policy selection, and contractor fine comparisons
- **Map:** Geographic incidents, route summaries, ZIP-code concentrations, and drill-through results

The default report state focuses on requests classified as missed pickups.

Configured visual interactions prevent exploratory selections from changing unrelated metrics. The policy selector changes the fine analysis without altering complaint-volume visuals.

The Map page is the only drill-through destination. It accepts route and ZIP-code context from compatible visuals.

The `CLEAR FILTERS` button is connected to a Map Reset bookmark. A Back action returns users to the visual that initiated drill-through.

The report compares two historical years. Because the later year is incomplete, the totals should not be treated as an equal-period full-year comparison.
