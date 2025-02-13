This application analyzes the potential of using curtailed wind farm energy for Bitcoin mining. 

Here's what it does:

## Energy Analysis:
- Tracks real-time wind farm curtailment data
- Monitors energy curtailment across different wind farms
- Calculates total available energy for mining
- Bitcoin Mining Calculations:

## Supports multiple ASIC miner models (S19J Pro, S9, M20S)
- Calculates potential Bitcoin mining output based on available (curtailed) energy
- Tracks network difficulty and adjusts calculations accordingly

## Financial Analysis:
- Provides real-time profitability calculations
- Compares curtailment payments vs potential mining revenue
- Tracks historical performance and projections

## Data Visualization:
- Interactive charts showing hourly curtailment data
- Daily, monthly, and yearly summaries
- Real-time difficulty and price monitoring

## Data Sources:
Primary data comes from Elexon BMRS API via elexon.ts
Wind farm BMUs (Balancing Mechanism Units) mappings are stored in bmuMapping.json
minerstat API used for current exchange rate
Our own DynamoDB used for historical difficulty levels 

# Ingestion Process:
Data is ingested daily through processDailyCurtailment function in curtailment.ts
Each day is processed in 48 settlement periods (half-hourly intervals)

## Real-time Auto Updates (Every 15 minutes):
The system runs a data update service (dataUpdater.ts)
The service checks for new data from the Elexon API
Updates the current day's data if any changes are detected

## Historical Reconciliation (Every 30 minutes):
Looks back 7 days to catch any retroactive updates
Compares our database with the Elexon API across multiple periods (1, 12, 24, 36, 48)
If differences are found (>1 MWh or >£10), it automatically reprocesses that day's data

## daily reconciliation for the previous month's data:
It runs daily at 2 AM (before the regular 7-day reconciliation at 3 AM)
Processes the entire previous month's data to catch any retroactive updates
Uses the same thorough comparison logic as the 7-day reconciliation
Shows detailed logs of any discrepancies found and corrections made

## On-demand Reconciliation: 
Available through reprocessDay.ts script

## Data Verification:
Checks both volume and payment data
Compares individual settlement periods and daily totals
Uses more granular logging to track any discrepancies

## Batch Processing:
Processes up to 5 days concurrently for efficiency
Includes automatic retries if API calls fail

## Data Storage:
Records are stored in PostgreSQL database across multiple tables:
- curtailmentRecords: Individual curtailment events
- dailySummaries: Aggregated daily totals (MWh and £)
- monthlySummaries: Monthly aggregations (MWh and £)
- yearlySummaries: Yearly aggregations (MWh and £)
- historicalBitcoinCalculations: Historical Bitcoin calculations are stored in the 


AWS DynamoDB for stored Difficulty levels used to calculate MWh to Bitcoin conversion

## The sign convention:
Negative £ values in DB → Positive display (payments TO farms)
Positive £ values in DB → Negative display (payments FROM farms)
the MWh energy values can be negative or positive in DB => display absolute numbers since they represent physical quantities

# MWh to Bitcoin conversion:
## Data Collection & Storage:
Fetching curtailment data (in MWh) from wind farms (via the Elexon API), processing the records (filtering by negative volumes and flags), and storing them in a PostgreSQL database.
Update daily, monthly, and yearly summaries based on these records.

## Bitcoin Calculation:
The core conversion logic converts the curtailed energy from MWh to kWh, calculates the number of miners that could run, and then estimates the Bitcoin mined during each 30-minute settlement period (48 periods per day).
This calculation takes into account the current Bitcoin network difficulty, ASIC's specification efficiency (e.g., for models like S19J_PRO), the network hashrate, and block reward.

## API Endpoint:
The /mining-potential endpoint accepts parameters like date, miner model, lead party, and farm ID, and it returns the calculated potential Bitcoin mined, its current value, network difficulty, and a breakdown by settlement period.

NB: So far I restrict the calculation to only process data for today’s date. To be changed for historical data.

## Frontend Display:
React frontend uses React Query to fetch the summary data (daily, monthly, yearly, and hourly) and the Bitcoin mining potential.
The UI components (cards, charts, filter bars) display the curtailed energy, potential Bitcoin mining amounts, and corresponding fiat values, along with hourly breakdowns in charts.
