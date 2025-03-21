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
The Elexon Balancing Mechanism Reporting Service (BMRS) stands as a central pillar in the operational transparency of the British electricity market. It functions as a comprehensive repository for a wide array of data encompassing electricity generation, demand patterns, and the actions taken to balance the grid. The primary avenue for obtaining specific information involves interacting with the BMRS Application Programming Interface (API).
It is important to note that the reported renewable energy output in this section might be an underestimation due to the exclusion of embedded generation and certain wind farms that lack operational meters 
Data from the GB electricity Balancing Mechanism (BM) for a specific wind farm BM Units. 
 
The Balancing Mechanism is the real-time tool the System Operator (National Grid ESO) uses to match electricity supply and demand.
Elexon:https://developer.data.elexon.co.uk/api-details#api=prod-insol-insights-api&operation=get-balancing-settlement-stack-all-bidoffer-settlementdate-settlementperiod - documentation

GET: https://data.elexon.co.uk/bmrs/api/v1/balancing/settlement/stack/all/{bidOffer}/{settlementDate}/{settlementPeriod}[?format]

soFlag or cadlFlag status

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

Primary Tables:
- curtailment_records: Stores detailed wind farm curtailment data at the settlement period level
- historical_bitcoin_calculations: Contains Bitcoin mining calculations based on curtailed energy

Summary Tables:
- daily_summaries: Aggregates curtailment data by day
- monthly_summaries: Aggregates curtailment data by month
- yearly_summaries: Aggregates curtailment data by year
- bitcoin_monthly_summaries: Summarizes Bitcoin calculations by month

Key relationship:
- Summary tables are derived from the primary tables
- Controllers often query both the summary tables AND re-calculate from primary tables
- Each update to the primary tables triggers cascading updates to all summary tables

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


## FAQ:
Why £/MHh per Bitcoin for each hour is different from hour to hour despite the difficulty remaining constant?

Why Bitcoin Value (£/MWh) Varies Throughout the Day
The Bitcoin value per MWh appears to fluctuate from hour to hour because of how the calculations are being performed in our system. There are several key factors causing this:

Aggregation Effects:

The hourly data represents multiple settlement periods (each hour has 2 settlement periods)
When we aggregate these periods into hourly blocks, we're combining data with different characteristics
Variable Curtailment Volume Distribution:

Each farm has different curtailment volumes at different times
When we calculate the Bitcoin value per MWh, we're dividing the total Bitcoin mined by the total energy curtailed
If one period has a very small volume and another has a large volume, the weighted average creates variations
Calculation Methodology:

Our system calculates Bitcoin mined for each individual curtailment record
These individual calculations are then aggregated up to hourly values
This bottom-up approach preserves the variability in the source data
Looking at the code, specifically in the server/utils/bitcoin.ts file, the Bitcoin calculation uses:

// From the bitcoin.ts utility
function calculateExpectedBitcoin(totalHashes: number, difficulty: number): number {
  // The formula uses both difficulty AND total hashes
  return (totalHashes / difficulty) * 6.25; // 6.25 BTC per block
}
While the difficulty is constant for the day, the total hashes computation varies based on the energy amount in each record. When we aggregate these precise calculations to an hourly level, the slight differences between settlement periods become visible.

Would you like me to modify the calculation approach to ensure a more consistent Bitcoin value per MWh across time periods? This would involve normalizing the calculations to better represent the theoretical Bitcoin mining potential per MWh.




