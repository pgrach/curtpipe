# curtpipe
New Curtailment Data Pipeline
Data reconciliation service:

## Real-time Updates (Every 15 minutes):
The service checks for new data from the Elexon API
Updates the current day's data if any changes are detected

## Historical Reconciliation (Every 30 minutes):
Looks back 7 days to catch any retroactive updates
Compares our database with the Elexon API across multiple periods (1, 12, 24, 36, 48)
If differences are found (>1 MWh or >£10), it automatically reprocesses that day's data

## Data Verification:
Checks both volume and payment data
Compares individual settlement periods and daily totals
Uses more granular logging to track any discrepancies

## Batch Processing:
Processes up to 5 days concurrently for efficiency
Includes automatic retries if API calls fail
