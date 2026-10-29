# Metro Nashville Trash Hauler Analysis

[View the interactive Power BI dashboard](https://app.powerbi.com/view?r=eyJrIjoiMTc0Zjk4YzAtYTVkNC00YTc4LWJkODUtNjZlZDM0OWFkYWYyIiwidCI6IjEwMWRhNTg3LTE4NDMtNGY1Mi04YjhhLTE3YjA2OWM2NmQzMyIsImMiOjJ9)

Additional technical details are available in [METHODS.md](METHODS.md).

## Project Overview

This project analyzes historical hubNashville service requests to evaluate missed trash pickups, recurring service problems, collection performance, and financial liability under different fine policies.

The Power BI dashboard compares trash haulers and collection routes, identifies geographic and seasonal complaint patterns, and allows users to test three policy scenarios.

## Business Assumptions

- Not every service request represents a missed pickup.
- Repeated missed pickups at the same address indicate a recurring service problem.
- Under the **Current Policy**, the first missed pickup at an address does not result in a fine. Each subsequent missed pickup results in a $200 fine.
- Under **Alternative Method 1**, a $1,500 fine is assessed when three unique missed-pickup dates occur within a rolling 180-day period. Dates may support multiple fines.
- Under **Alternative Method 2**, the same three-date and 180-day requirements apply, but each date may support only one fine.

## Project Objectives

- Identify service requests likely to represent missed pickups
- Compare performance across trash haulers and collection routes
- Identify addresses and areas with recurring complaints
- Analyze complaint patterns by year, month, and weekday
- Compare estimated liability under three fine structures
- Provide route- and ZIP-level geographic analysis

## Technologies Used

- Microsoft Power BI
- Power Query and M
- DAX
- Python
- pandas
- pyproj

## Techniques and Methodology Used

- Rule-based complaint classification
- Address normalization and composite-key construction
- Dimensional data modeling
- Geographic coordinate conversion
- Time-intelligence calculations
- Scenario-based policy analysis
- Controlled visual interactions and drill-through
- Policy-output validation

## Data Source

The project uses historical service-request data from Metro Nashville’s hubNashville system. The analyzed CSV is included in the repository for reproducibility.

A current version of the broader dataset is available through [Nashville Open Data](https://data.nashville.gov/maps/Nashville%3A%3Ahubnashville-311-service-requests-current-year/about).

The results represent the historical project period and should not be interpreted as current collection performance.
