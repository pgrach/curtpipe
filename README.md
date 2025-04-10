Data Pipeline Overview
The data pipeline follows this general flow:

Elexon API → curtailment_records → summaries (daily/monthly/yearly) → historical_bitcoin_calculations → bitcoin summary tables
1. Data Ingestion from Elexon API
The system fetches curtailment data from the Elexon API using the following process:

The fetchBidsOffers function in server/services/elexon.ts makes API requests to Elexon's API endpoints:

Bids endpoint: ${ELEXON_BASE_URL}/balancing/settlement/stack/all/bid/${date}/${period}
Offers endpoint: ${ELEXON_BASE_URL}/balancing/settlement/stack/all/offer/${date}/${period}
The system includes rate limiting to avoid exceeding API limits (4500 requests per minute)

It validates the responses and filters the data to only include wind farms by:

Checking records against a pre-defined set of valid BMU (Balancing Mechanism Unit) IDs from a mapping file
Filtering for records where volume < 0 and soFlag is true (indicating system operator curtailment)
Processing 48 settlement periods per day (the UK electricity market uses 48 half-hour settlement periods)
2. Storing Curtailment Records
The filtered data is stored in the curtailment_records table:

The processDailyCurtailment function in server/services/curtailment_enhanced.ts processes the data:

It first clears any existing records for the target date to prevent duplicates
Processes each settlement period (1-48)
For each valid curtailment record, it creates a database entry with:
settlementDate: The date (YYYY-MM-DD)
settlementPeriod: The period (1-48)
farmId: The BMU ID from Elexon
leadPartyName: The party responsible for the wind farm
volume: The curtailed energy in MWh (stored as a negative value)
payment: The payment for curtailed energy (price * volume)
The system processes batches of records to optimize database operations

3. Generating Summary Tables
After populating the curtailment_records table, the system generates various summary tables:

Daily Summaries:

Aggregates curtailment data by day
Calculates total curtailed energy and payments
Stores in daily_summaries table
Monthly Summaries:

Aggregates curtailment data by month
Updates the monthly_summaries table
The system will auto-update the monthly summary when a daily summary changes
Yearly Summaries:

Aggregates curtailment data by year
Updates the yearly_summaries table
The system will auto-update the yearly summary when a monthly summary changes
4. Bitcoin Mining Potential Calculations
For each curtailment record, the system calculates potential Bitcoin mining:

Historical Bitcoin Calculations:

The processHistoricalCalculations or processBitcoinCalculations functions process each curtailment record
For each curtailment record and each configured miner model (S19J_PRO, S9, M20S):
The system calculates how much Bitcoin could be mined using the curtailed energy
It uses the calculateBitcoin function in server/utils/bitcoin.ts
Bitcoin calculation formula:
Convert curtailed MWh to kWh
Determine how many miners could operate with that energy
Calculate the potential hash power those miners would produce
Calculate the Bitcoin that could be mined based on network difficulty
Network Difficulty Data:

The system fetches Bitcoin network difficulty from DynamoDB using getDifficultyData in server/services/dynamodbService.ts
If the difficulty isn't available for a specific date, it falls back to a default or the latest known value
Storing Bitcoin Calculations:

The system stores these calculations in the historical_bitcoin_calculations table
Each record includes:
settlementDate: The date
settlementPeriod: The period (1-48)
farmId: The BMU ID
minerModel: The miner model used for calculation
bitcoinMined: The calculated potential Bitcoin amount
difficulty: The Bitcoin network difficulty used for calculation
5. Bitcoin Summary Tables
Finally, the system generates Bitcoin summary tables:

Bitcoin Daily Summaries:

Aggregates Bitcoin calculations by day for each miner model
Stores in bitcoin_daily_summaries table
Bitcoin Monthly Summaries:

Aggregates Bitcoin calculations by month for each miner model
Stores in bitcoin_monthly_summaries table
Bitcoin Yearly Summaries:

Aggregates Bitcoin calculations by year for each miner model
Stores in bitcoin_yearly_summaries table
6. Additional Data Processing
The system also includes:

Wind Generation Data:

Stored in the wind_generation_data table
Updated using functions in server/services/windGenerationService.ts
Provides context for total wind generation vs. curtailed generation
Data Verification and Reconciliation:

The system includes tools to verify data integrity (verify_data_integrity.ts)
It can identify and fix discrepancies between tables (verify_service.ts)
Reprocessing Capabilities:

Various scripts allow reprocessing of specific dates or ranges
These scripts maintain data consistency across all dependent tables
Automatic Update Chain
The system implements a cascading update mechanism:

When new curtailment records are added, it triggers:
Daily summary updates
Bitcoin calculation updates
When daily summaries are updated, it triggers:
Monthly summary updates
Bitcoin daily summary updates
When monthly summaries are updated, it triggers:
Yearly summary updates
Bitcoin monthly summary updates
When Bitcoin daily summaries are updated, it triggers:
Bitcoin monthly summary updates
When Bitcoin monthly summaries are updated, it triggers:
Bitcoin yearly summary updates
This ensures that all summary tables remain consistent even when new data is added or existing data is modified.




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
Wind farm BMUs (Balancing Mechanism Units) mappings are stored in bmuMapping.json; 
The process of obtaining BMU-specific wind generation data involves several steps: 
1) retrieving the aggregated wind generation data from report B1630; second, obtaining both the comprehensive list of BMUs and the BMU-Fuel Type mapping; third, utilizing the mapping to identify those BMUs categorized as wind generation units; and finally, potentially distributing the aggregated wind generation data among these identified wind BMUs. This final step might require additional data or assumptions, such as the installed capacity of each wind BMU, as the API does not appear to directly provide BMU-level generation for wind through report B1630. 
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




