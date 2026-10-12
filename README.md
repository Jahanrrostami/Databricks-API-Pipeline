# Databricks-API-Pipeline
A project based in Databricks where data is extracterd transformed in DB and loaded into containers in Azure in a madalion architecture.

A Databricks lakehouse that ranks Swedish municipalities for 50 new fast chargers. The decision uses three signals: many newly registered electric cars, fast growth in those registrations, and a low electricity price.
 
The catalog is `dbjahan`. Each layer is a schema with a managed location on its own container in the storage account `datatiersjahan`:
 
| Layer | Schema | Container | What it holds |
|---|---|---|---|
| Bronze | `bronze_data` | `bronze` | Raw API responses |
| Silver | `silver_data` | `silver` | Municipality-level tables |
| Gold | `gold_data` | `gold` | The decision model |
 
Notebooks do not contain storage keys or `abfss` paths. `CREATE TABLE` writes Delta to the container configured on the schema.
 
## Sources
 
| Table | API | Content |
|---|---|---|
| `bronze_data.nyreg_bilar_raw` | SCB TAB3277 | New cars by municipality, fuel and month, from January 2022 |
| `bronze_data.bilar_trafik_raw` | SCB TAB3276 | Cars in traffic from 2020 |
| `bronze_data.folkmangd_raw` | SCB TAB638 | Population from 2020, including marital status |
| `bronze_data.elpris_raw` | elprisetjustnu.se | Hourly price for SE1–SE4 during 2025 |
 
SCB does not publish the electric-car fleet per municipality. "Many electric cars" in this model means new electric cars in the last 12 months per 1,000 inhabitants. `bilar_i_trafik` counts every fuel type and is only a description of the municipality.
 
## Silver
 
| Table | Grain |
|---|---|
| `nyreg_bilar` | Municipality, fuel, year, month |
| `bilar_trafik` | Municipality, year |
| `folkmangd` | Municipality, year |
| `elpris` | Electricity area and year. One row per area and hour before the yearly average |
| `kommun_elomrade` | Municipality to electricity area |
 
The area is not only the first two digits of the municipality code. The border cuts through some counties. Gävle, Hofors and Sandviken are SE3. Bollnäs stays SE2. Kalmar county is SE4 except Hultsfred, Vimmerby and Västervik. Skellefteå, Malå and Norsjö are SE1. Varberg and Kungsbacka are SE3. Gnosjö and Värnamo are SE4. Heby is kept as `0331`. The old code `1917` is dropped, so the municipality table has 290 rows.
 
## Gold
 
One row per municipality.
 
- `dim_kommun` holds the code, name, electricity area, population and cars in traffic.
- `fakt_laddplats` holds the measures and the score.
- `topp_50_kommuner` is the 50 municipalities with the highest score and at least one new electric car in the last year.
 
The score is:
 
```text
0.4 * rank(new electric cars per 1,000 inhabitants)
+ 0.4 * rank(growth)
+ 0.2 * rank(low electricity price)
```
 
Growth compares the last 12 months with the 12 months before that. The price is the yearly average for the municipality's area. A lower price ranks higher.
 
## Reruns
 
Bronze loads with `INSERT` and `WHERE NOT EXISTS` on the natural key, so the same source row is not stored twice. Tables are created with `CREATE TABLE IF NOT EXISTS`.
 
Silver is rebuilt with `CREATE OR REPLACE TABLE`. Gold is replaced with `INSERT OVERWRITE` into `fakt_laddplats` and `topp_50_kommuner`. A new run recalculates the score.
 
Names such as `nyreg_nya` and `elpris_nya` are temporary views in the same cell. They are not tables in the catalog.
 
## Quality check
 
`silver_data.kontroll` runs after silver and before gold. It stops the pipeline when:
 
- the municipality count is not 290
- code `1917` is still present
- an electricity area is null
- a checked municipality has the wrong area
- an area does not have about 8,760 hours in the year
 
Gold runs only after this check is green.
 
## Orchestration
 
The Databricks job `laddplatst_pipeline` runs the chain:
 
1. The four bronze notebooks run in parallel.
2. Each silver notebook waits for its bronze table. `kommun_elomrade` waits for registrations and the electricity price.
3. `silver_kontroll` waits for every silver task.
4. Gold waits only for the check.
 
Azure Data Factory is not part of this project. The free Azure subscription had ended, so the schedule and the retries live on the Databricks job. The trigger is 06:00 Europe/Stockholm on the first day of each month. SCB updates monthly, and placing chargers is not a daily decision. The price in the model is a yearly average, so one extra day moves it very little. One run a month is enough and starts compute only then.
 
## API failures
 
`hamta` calls the same URL three times, with five seconds between tries. The third failure aborts the notebook. The message names the API, the URL and the status code. Each bronze task also retries twice, 60 seconds apart, so a failed extract is repeated before silver and gold write.

## Runbook
 
**Run.** Open `laddplatst_pipeline` and choose **Run now**, or wait for the monthly trigger.
 
**Investigate.** Open the red task under **Runs**. A `hamta` error names the API and the status. A `silver_kontroll` error names the broken rule, for example the municipality count or the hour count.
 
**Known cases.** An API error after three tries is intentional. Run the job again later. The check stops the job if Heby `1917` comes back or if the electricity price is not a clean hourly average. The next green run rebuilds silver and gold. Do not run the SQL half of a bronze cell without the line that creates the temporary view.
 
## Limitations
 
New registrations are a flow, not the number of electric cars already on the road. A small municipality can rank high on percentage growth from a handful of cars. The price series is 2025. A few municipalities are split by an area border, and the model assigns one area to the whole municipality. The list is a priority input. Existing chargers and grid capacity are not included.

