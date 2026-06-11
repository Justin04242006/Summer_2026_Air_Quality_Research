This file contains all the R code I used to query and analyze data from
the clearlab\_purpleair SQL database for the 4-week Summer 2026 USRE
project titled “Understanding Air Quality Through Data.”

## Week 2: early queries, determining relevant dates per sensor, and adding activity status flag column

(no queries were accomplished during Week 1 due to technical
difficulties)

After much troubleshooting, the research began by connecting to the
database via an ODBC application programming interface (API).

    library(DBI)

    ## Warning: package 'DBI' was built under R version 4.4.3

    library(odbc)

    ## Warning: package 'odbc' was built under R version 4.4.3

    library(PurpleAirAPI)

    Sys.getenv("ODBCINI")
      Sys.getenv("ODBCSYSINI")
      
      Sys.setenv(ODBCSYSINI = "/Library/ODBC/odbc.ini")
      Sys.setenv(ODBCINI = "/Library/ODBC/odbc.ini")
      
      
      con <- dbConnect(
      odbc(),
      Driver = "/opt/homebrew/lib/libmsodbcsql.18.dylib",
      Server = "resdbsprdhs02.adms.luc.edu",
      Database = "clearlab_purpleair",
      UID = Sys.getenv("DB_USERNAME"),
      PWD = Sys.getenv("DB_PASSWORD"),
      Encrypt = "yes",
      TrustServerCertificate = "yes"
    )

Once connected to the database, I tested some codes provided by my
professor to pull data from the PurpleAir API for a random sensor at a
random date, to ensure the code behaves as expected (which it does).

    api_key<-Sys.getenv("API_KEY") #The CLEAR lab's API key, along with my database username and password, have been encoded into environment variables for security purposes.
    sensor_index<-"195173"


    start_date <- "2025-05-01"
    end_date   <- "2025-05-02"
    average <- "0" 
    fields <- c(
      "humidity",
      "temperature",
      "pressure",
      "pm2.5_cf_1_a",
      "pm2.5_cf_1_b",
      "rssi"
    )

    historical_data <- getSensorHistory(
      sensorIndex = sensor_index,
      apiReadKey  = api_key,
      startDate   = start_date,
      endDate     = end_date,
      fields      = fields,
      average     = average
    )

    head(historical_data, 20)

After doing some practice queries on the air\_history dataset, I was
ready for my first actual SQL query: determining the most recent date in
the air\_history table (which was necessary to determine the date range
I needed to pull from the API in order to completely update it).

    most_recent_date<-dbGetQuery(con, "SELECT MAX(CAST(time_stamp AS date)) 
                                 AS Most_recent_date 
                                 FROM air_history")

    most_recent_timestamp<-dbGetQuery(con, "SELECT MAX(time_stamp) 
                                      AS Most_recent_time_stamp 
                                      FROM air_history")

    most_recent_date
    most_recent_timestamp

When I originally ran the above code on May 27th, the most recent date
in the SQL database was December 3, 2025, and the most recent timestamp
was 23:59 (11:59 PM). Therefore, I needed to pull data from 12/4/25 to
5/27/26 (inclusive) and append this data to the database.

    #Defining parameters and desired columns
    start_date <- "2025-12-04"
    end_date   <- "2026-05-27"
    average <- "0"  
    fields <- c(
      "humidity",
      "temperature",
      "pressure",
      "pm2.5_cf_1_a",
      "pm2.5_cf_1_b",
      "rssi")

    #Downloading data
    data_since_Dec_4 <- getSensorHistory(
      sensorIndex = sensor_index,
      apiReadKey  = api_key,
      startDate   = start_date,
      endDate     = end_date,
      fields      = fields,
      average     = average
    )

    #Transforming and reordering columns to match database structure
    data_since_Dec_4$time_stamp <- as.POSIXct(
      data_since_Dec_4$time_stamp,
      origin = "1970-01-01",
      tz = "UTC"
    )
    data_since_Dec_4$sensor_index <- sensor_index
    data_since_Dec_4$pm2p5_cf_1_a <- data_since_Dec_4$`pm2.5_cf_1_a`
    data_since_Dec_4$pm2p5_cf_1_b <- data_since_Dec_4$`pm2.5_cf_1_b`
    data_since_Dec_4$pm2p5_diff   <- data_since_Dec_4$pm2p5_cf_1_b - data_since_Dec_4$pm2p5_cf_1_a
    data_since_Dec_4$rssi <- as.character(data_since_Dec_4$rssi)

    data_since_Dec_4 <- data_since_Dec_4[, c(
      "sensor_index",
      "time_stamp",
      "humidity",
      "temperature",
      "pressure",
      "pm2p5_cf_1_a",
      "pm2p5_cf_1_b",
      "rssi",
      "pm2p5_diff"
    )]

    #Appending data to database

    dbWriteTable(
      con,
      name = "air_history",
      value = data_since_Dec_4,
      append = TRUE,
      row.names = FALSE
    )

    #Rerunning the most recent date query to ensure the update was successful (which it was, since the result was 05/27/26 instead of 12/04/25)

    new_most_recent_date<-dbGetQuery(con, "SELECT MAX(CAST(time_stamp as date)) AS Most_recent_date FROM air_history")

    # Appending a flag column to the data in the database (Inactive=last_seen before May 1, 2026; Active=last_seen on or after May 1, 2026)

    all_metadata<-dbGetQuery(con, "SELECT * FROM sensor_infomation")

    #I can't actually rerun any of the following code from scratch because they represent operations on the database that have already been done, so all the code related to updating the metadata table is set to eval=FALSE.

    for (i in all_metadata$sensor_index) {

    query <- "
    UPDATE sensor_infomation
    SET sensor_activity_status =
        CASE
          WHEN CAST(last_seen AS date) >= '2026-05-01'
            THEN 'Active'
          ELSE 'Inactive'
        END
      WHERE sensor_index = ?"

      dbExecute(con, query, params = list(i))
    }

The resulting data had a lot of duplicate sensor indices, so I removed
the duplicates from the database using the following code (again, I
can’t actually run the code again since the changes have already been
made to the database):

    duplicate_sensors <- dbGetQuery(con, 
    "SELECT
        sensor_index,
        COUNT(*) AS n
    FROM sensor_information
    GROUP BY sensor_index
    HAVING COUNT(*) > 1")

    dbExecute(con, "
    WITH duplicates AS (
        SELECT *,
               ROW_NUMBER() OVER (
                   PARTITION BY sensor_index
                   ORDER BY sensor_index) AS rn
        FROM sensor_information)
    DELETE FROM duplicates
    WHERE rn > 1")

Noticing that the cleaned metadata table contained NA values for the
model, name, hardware, and location\_type fields, I then used the
getPurpleAirsensors() function in the PurpleAirAPI package to retrieve
updated sensor metadata. When I specified all the fields of data in the
metadata table, the function failed in the following way:

    fields=c("date_created", "last_seen", "name", "location_type", "model", "hardware", "latitude", "longitude")
    getPurpleairSensors(apiReadKey = api_key, fields=fields)

    #Error in `as.POSIXlt.character()`:
    #! character string is not in a standard unambiguous format

This motivated me to do something I had never done before: examine and
try to debug the source code of the function.

    f<-edit(getPurpleairSensors)

This opened a panel within R containing the source code, which was the
following:

    function (apiReadKey = NULL, fields = c("latitude", "longitude", 
        "date_created", "last_seen")) 
    {
        if (is.null("apiReadKey")) {
            stop(paste("apiReadKey not defined!"))
        }
        required_params <- c("apiReadKey")
        for (param in required_params) {
            if (is.null(get(param))) {
                stop(paste(param, "not defined!"))
            }
        }
        validate_api_key(apiReadKey)
        api_endpoint <- paste0("https://api.purpleair.com/v1/sensors?fields=", 
            paste(fields, collapse = "%2C"))
        result <- httr::GET(api_endpoint, config = httr::add_headers(`X-API-Key` = apiReadKey))
        if (httr::http_error(result)) {
            error_content <- httr::content(result, as = "text", encoding = "UTF-8")
            error_details <- jsonlite::fromJSON(error_content)
            error_message <- paste(httr::status_code(result), error_details$error, 
                error_details$description)
            stop(error_message)
        }
        raw <- httr::content(result, as = "text", encoding = "UTF-8")
        response_list <- jsonlite::fromJSON(raw)
        purpleair <- as.data.frame(response_list$data)
        names(purpleair) <- response_list$fields
        date_cols <- c("date_created", "last_seen")
        for (col in date_cols) {
            if (col %in% names(purpleair)) {
                purpleair[[col]] <- as.Date(as.POSIXct(purpleair[[col]], 
                    origin = "1970-01-01"))
            }
        }
        return(purpleair)
    }

The first issue I noticed with this code is that only four of the fields
are specified in the fields vector, indicating that this code might have
been designed for an outdated version of the database. I thus added the
other relevant fields into the fields vector, but that modification
still yielded the same error message.

After explaining my predicament to ChatGPT, it told me that the other
issue resided in the for loop towards the end of the function.
Specifically, the date\_created and last\_seen fields needed to be
converted to numeric type before they could be transformed into POSIXct
type. It also pointed out that apiReadKey was presented as a string
towards the beginning of the function, not as a variable.

After making these modifications and saving them to an alias called f,
the source code looked like this:

    f<-edit(getPurpleairSensors)
    print(f)


    function (apiReadKey = NULL, fields = c("date_created", "last_seen", "name", "location_type", "model", "hardware", "latitude", "longitude", "sensor_activity_status")) 
    {
        if (is.null(apiReadKey)) {
            stop(paste("apiReadKey not defined!"))
        }
        required_params <- c(apiReadKey)
        for (param in required_params) {
            if (is.null(get(param))) {
                stop(paste(param, "not defined!"))
            }
        }
        validate_api_key(apiReadKey)
        api_endpoint <- paste0("https://api.purpleair.com/v1/sensors?fields=", 
            paste(fields, collapse = "%2C"))
        result <- httr::GET(api_endpoint, config = httr::add_headers(`X-API-Key` = apiReadKey))
        if (httr::http_error(result)) {
            error_content <- httr::content(result, as = "text", encoding = "UTF-8")
            error_details <- jsonlite::fromJSON(error_content)
            error_message <- paste(httr::status_code(result), error_details$error, 
                error_details$description)
            stop(error_message)
        }
        raw <- httr::content(result, as = "text", encoding = "UTF-8")
        response_list <- jsonlite::fromJSON(raw)
        purpleair <- as.data.frame(response_list$data)
        names(purpleair) <- response_list$fields
        date_cols <- c("date_created", "last_seen")
        for (col in date_cols) {
            if (col %in% names(purpleair)) {
                numeric_time<-as.numeric(purpleair[[col]])
                purpleair[[col]] <- as.POSIXct(purpleair[[col]], 
                    origin = "1970-01-01", tz="UTC")
            }
        }
        return(purpleair)
    }

I then called the function f to pull updated sensor metadata and append
it to the sensor\_infomation table in the SQL server. This successfully
updated all the fields of the table, leaving no missing values.

    fields=c("date_created", "last_seen", "name", "location_type", "model", "hardware", "latitude", "longitude", "sensor_activity_status")
    api_key<-"F38F115A-78CA-11EE-A8AF-42010A80000A"
    newdata<-f(apiReadKey = api_key, fields=fields)

    dbWriteTable(
      con,
      name = "sensor_infomation",
      value = newdata,
      append = TRUE,
      row.names = FALSE)

\#Week 3: shrinking of data table and conceptualizing/designing a
trigger to automatically pull the previous 24 hours of data for relevant
sensors

Dr. Whalen informed me that the server currently contains information
for all PurpleAir sensors on Earth, and we are only interested in
Chicago area sensors. Therefore, I needed to clean the air\_history and
sensor\_infomation tables to only include Chicago area sensors.

Dr. Whalen gave me a csv file of metadata for all the Chicago sensors
(149 total sensors).

    library(tidyverse)

    ## Warning: package 'ggplot2' was built under R version 4.4.3

    ## Warning: package 'tibble' was built under R version 4.4.3

    ## Warning: package 'tidyr' was built under R version 4.4.3

    ## Warning: package 'readr' was built under R version 4.4.3

    ## Warning: package 'purrr' was built under R version 4.4.3

    ## Warning: package 'dplyr' was built under R version 4.4.3

    ## ── Attaching core tidyverse packages ──────────────────────── tidyverse 2.0.0 ──
    ## ✔ dplyr     1.2.0     ✔ readr     2.1.6
    ## ✔ forcats   1.0.1     ✔ stringr   1.6.0
    ## ✔ ggplot2   4.0.2     ✔ tibble    3.3.1
    ## ✔ lubridate 1.9.4     ✔ tidyr     1.3.2
    ## ✔ purrr     1.2.1     
    ## ── Conflicts ────────────────────────────────────────── tidyverse_conflicts() ──
    ## ✖ dplyr::filter() masks stats::filter()
    ## ✖ dplyr::lag()    masks stats::lag()
    ## ℹ Use the conflicted package (<http://conflicted.r-lib.org/>) to force all conflicts to become errors

    sensor_info<-read_csv("~/Desktop/Github_Portfolio_Projects/STAT_308_Project/USRE_2026_Air_Quality_Research/sensor_info_May26.csv")

    ## Rows: 149 Columns: 9
    ## ── Column specification ────────────────────────────────────────────────────────
    ## Delimiter: ","
    ## chr  (4): name, location_type, model, hardware
    ## dbl  (3): sensor_index, latitude, longitude
    ## dttm (2): date_created, last_seen
    ## 
    ## ℹ Use `spec()` to retrieve the full column specification for this data.
    ## ℹ Specify the column types or set `show_col_types = FALSE` to quiet this message.

Based on this file, I ran a SQL query to filter rows of the air\_history
and sensor\_infomation tables to only include rows for which the sensor
index matched an index in the csv file.

    air_history_cleaning<-"DELETE FROM air_history WHERE sensor_index NOT IN ('3499','265767','4395','4404','5588','5774','268129','6546','7920','8476','271809','282164',
                                 '289288','292499','36281','36901','39173','41581','43955','45079','45359','51933',
                                 '52011','56135','57579','65791','87741','88413','94003','95075','96035','96079','96269',
                                 '96933','101643','109598','113494','113959','120369','123003','124513','124649','124677','124679',
                                 '124685','124715','124737','124747','124759','128349','133664','137618','137622','140102','140390',
                                 '144504','147803','147811','147893','148029','148035','148545','148989','149090','149530','149572','150042',
                                 '150066','151042','151050','151074','151082','151188','151606','153638','166645','168725','169187','171015',
                                 '171075','175227','175419','175417','175455','175457','175483','175921','176899','178649','179769','180725',
                                 '182245','182281','184435','184939','185239','185237','186099','186621','186941','187547','187615','188475','188749',
                                 '189233','189329','189821','191181','192597','192837','193661','193671','193669','193676','193689','193787','193801',
                                 '195173','195181','196015','196037','203303','211477','220177','220191','220209','220223','220261','220521','220531',
                                 '220537','220553','220579','220577','220583','220591','220613','221165','226529','233117','241195','244039','244045',
                                 '244051','244057','244063','244073','244083','251599')"

    dbExecute(con, air_history_cleaning)

The output of 0 means that the database operation did not remove any
rows from the air\_history table. In other words, the air\_history table
only contained records for Chicago sensors to begin with.

    sensor_infomation_cleaning<-"DELETE FROM sensor_infomation WHERE sensor_index NOT IN ('3499','265767','4395','4404','5588','5774','268129','6546','7920','8476','271809','282164',
                                 '289288','292499','36281','36901','39173','41581','43955','45079','45359','51933',
                                 '52011','56135','57579','65791','87741','88413','94003','95075','96035','96079','96269',
                                 '96933','101643','109598','113494','113959','120369','123003','124513','124649','124677','124679',
                                 '124685','124715','124737','124747','124759','128349','133664','137618','137622','140102','140390',
                                 '144504','147803','147811','147893','148029','148035','148545','148989','149090','149530','149572','150042',
                                 '150066','151042','151050','151074','151082','151188','151606','153638','166645','168725','169187','171015',
                                 '171075','175227','175419','175417','175455','175457','175483','175921','176899','178649','179769','180725',
                                 '182245','182281','184435','184939','185239','185237','186099','186621','186941','187547','187615','188475','188749',
                                 '189233','189329','189821','191181','192597','192837','193661','193671','193669','193676','193689','193787','193801',
                                 '195173','195181','196015','196037','203303','211477','220177','220191','220209','220223','220261','220521','220531',
                                 '220537','220553','220579','220577','220583','220591','220613','221165','226529','233117','241195','244039','244045',
                                 '244051','244057','244063','244073','244083','251599')"


    dbExecute(con, sensor_infomation_cleaning)

This gives an output of 0 because I have already done this operation on
the sensor\_infomation table. When I first ran this code, the output was
54,921, meaning 54,921 out of 55,132 rows were deleted from this table.
However, this left 211 rows in sensor\_infomation, which was still
greater than the 149 rows in the csv file.

I queried the table to inspect it.

    quick_check<-dbGetQuery(con, "SELECT * FROM sensor_infomation")

The reason for the extra rows is the presence of duplicates for some of
the sensors, from when I used the original version of the
getPurpleairSensors() function to update the sensor metadata and then
did the same using my modified version of the function. This resulted in
a row for a particular sensor index containing all fields followed by
another row with the same sensor index but missing the name, hardware,
model, and location\_type fields. Therefore, I ran other database
operation to clear out rows with missing fields.

    more_sensor_infomation_cleaning<-"DELETE FROM sensor_infomation WHERE name IS NULL"
    dbExecute(con, more_sensor_infomation_cleaning)

After data cleaning was finished, the sensor\_infomation table looked
like this:

    updated_sensor_infomation<-dbGetQuery(con, "SELECT * FROM sensor_infomation")

    updated_sensor_infomation

The final task of the USRE project was creating a script to
automatically update the last\_seen column of the sensor\_infomation
table and the air\_history table with new data every 24 hours. This
script also updates a newly created trigger\_pulls table with metadata
about the historical data pulls, including a unique pull\_id, the
time\_stamp of the pull, how many rows were appended to air\_history,
and the sensors for which historical data was pulled.

The key when writing this script was ensuring that we only pulled
historical data for sensors meeting both of these criteria: 1) the
sensor is active, as defined based on the flag column previously created
2) the sensor’s last\_seen date as seen in the API was more recent than
the last\_seen date currently in the sensor\_infomation table
(indicating that the sensor had collected new data over the previous 24
hours)

This drastically reduces the number of API points consumed each time the
script runs.

The trigger\_pulls table was created using the following code:

    dbExecute(con, "
      CREATE TABLE trigger_pulls (
          pull_id INT IDENTITY(1,1) PRIMARY KEY,
          pull_timestamp DATETIME NOT NULL,
          rows_inserted INT NOT NULL,
          sensors_processed VARCHAR(MAX) NOT NULL
      )
    ")

Once created, I added a placeholder row to the trigger\_pulls table to
represent all the historical data currently in air\_history.

    history_summary <- dbGetQuery(con, "
    SELECT
        COUNT(*) AS rows_inserted,
        MIN(time_stamp) AS first_record,
        MAX(time_stamp) AS last_record
    FROM air_history
    ")

    trigger_log_df <- data.frame(
      pull_timestamp = Sys.time(),
      rows_inserted = history_summary$rows_inserted,
      sensors_processed = "Initial historical backfill",
      stringsAsFactors = FALSE
    )

    dbWriteTable(
      conn = con,
      name = "trigger_pulls",
      value = trigger_log_df,
      append = TRUE,
      row.names = FALSE
    )

    #Testing to make sure the row was added

    new_query<-dbGetQuery(con, "SELECT * FROM trigger_pulls")
    new_query

After making a new ER diagram for the desired database structure and
considering the high-level workflow of the script, I wrote the script
(with some syntactical help from ChatGPT and conceptual help from my
professor).

This is what the script accomplishes at a high level:

1.  Connect to the clearlab\_purpleair SQL server
2.  Query and vectorize sensor IDs that are “Active” based on the flag
    column (active=last\_seen in the database more recent than 6/1/26)
3.  Pull updated last\_seen timestamps from the PurpleAir API for all
    active sensors
4.  For each active sensor, compare its API last\_seen date to the
    last\_seen date already in the database for that sensor. If the API
    last\_seen date is more recent, replace the last\_seen value in the
    corresponding row of the sensor\_infomation table with the updated
    API last\_seen date.
5.  If at least 1 sensor’s last\_seen date was updated, pull historical
    sensor data from the API from the most recent timestamp until the
    current time, then append this historical data to the air\_history
    table.
6.  At the same time as the historical data is pulled, add a row to the
    trigger\_pulls table with a unique pull ID, the timestamp of the
    historical data pull, the number of rows inserted into air\_history,
    and a list of the sensors whose historical data was added.
7.  Disconnect from the SQL server.

<!-- -->

    library(tidyverse)
    library(readxl)
    library(httr)
    library(jsonlite)
    library(lubridate)
    library(PurpleAirAPI)
    library(DBI)
    library(odbc)

    # 1. Connect to server
    api_key <- "F38F115A-78CA-11EE-A8AF-42010A80000A"

    cat(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "- Script began executing.\n")

    con = dbConnect(
      odbc(),
      Driver = "/opt/homebrew/lib/libmsodbcsql.18.dylib",
      Server = "resdbsprdhs02.adms.luc.edu",
      Database = "clearlab_purpleair",
      UID = "jpeabody",
      PWD = "bf265n8eNmLq!",
      Encrypt = "yes",
      TrustServerCertificate = "yes"
    )

    cat(
      format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
      "- Connected to database\n"
    )

    ### 2. Fetch Database Active Sensors
    active_sensors <- dbGetQuery(con, "SELECT sensor_index, last_seen FROM sensor_infomation WHERE sensor_activity_status='Active'")
    sensoridsvec <- active_sensors$sensor_index

    cat(
      format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
      "- Retrieved active sensors\n"
    )

    ### 3.Fetch Live API Timestamps

    api_sensors_all <- getPurpleairSensors(apiReadKey = api_key, fields = c("last_seen"))

    updated_sensors <- c() 
    sensors_pulled_log<-c()

    ### 4. Update last_seen dates where necessary. 

    for (i in sensoridsvec) {
      query1 <- dbGetQuery(con, "SELECT last_seen FROM sensor_infomation WHERE sensor_index = ?", params = list(i))
      db_last_seen <- if(nrow(query1) > 0) query1$last_seen else NA
      
      api_row <- api_sensors_all %>% 
        filter(sensor_index == i)
      
      if (nrow(api_row) > 0) {
        api_last_seen <- api_row$last_seen[1]
        
        if (is.na(db_last_seen) || db_last_seen < api_last_seen) {
          
          query2 <- "UPDATE sensor_infomation SET last_seen = ? WHERE sensor_index = ?"
          
          dbExecute(con, query2, params = list(as.character(api_last_seen), i))
          
          updated_sensors <- c(updated_sensors, i) 
          sensors_pulled_log <- c(sensors_pulled_log, as.character(i)) 
        }
      }
    }

    cat(
      format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
      "- Updated last_seen where necessary\n"
    )

    ### 5. Fetch and Process Historical Data

    if (length(updated_sensors) > 0) { 
      
      current_time <- Sys.time()
      time_query <- dbGetQuery(con, "SELECT MAX(time_stamp) AS time_stamp FROM air_history")
      initial_time<-time_query$time_stamp
      
      start_date_str <- format(initial_time,  format = "%Y-%m-%d %H:%M:%S", tz = "UTC")
      end_date_str   <- format(current_time, format = "%Y-%m-%d %H:%M:%S", tz = "UTC")
      
      update_raw<-NULL
      
      tryCatch({update_raw = getSensorHistory(
        sensorIndex = updated_sensors, 
        apiReadKey=api_key,
        startDate=start_date_str,
        endDate=end_date_str,
        average='0', 
        fields=c('humidity', 'temperature', 'pressure', 'pm2.5_cf_1_a', 'pm2.5_cf_1_b', 'rssi')
      )
      
      cat(
        format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
        "- Pulled previous 24 hours of historical data\n"
      )}, error= function(e) {
        cat(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "- ERROR: PurpleAir API history pull failed:", conditionMessage(e), "\n")
      })
      
      if (!is.null(update_raw) && nrow(update_raw) > 0) {
      
      update_clean = update_raw %>% 
        mutate(
          pm2p5_cf_1_a = `pm2.5_cf_1_a`,
          pm2p5_cf_1_b = `pm2.5_cf_1_b`,
          pm2p5_diff   = pm2p5_cf_1_a - pm2p5_cf_1_b)%>%
        select(sensor_index, time_stamp, humidity, temperature, pressure,
               pm2p5_cf_1_a = `pm2.5_cf_1_a`, pm2p5_cf_1_b = `pm2.5_cf_1_b`, rssi, pm2p5_diff)
      
      cat(
        format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
        "- Processed and cleaned historical data\n"
      )
      
      
      dbWriteTable(conn = con,
                   name = 'air_history',
                   value = update_clean,
                   append = TRUE,
                   row.names = FALSE)
      
      cat(
        format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
        "- Appended historical data to air_history\n"
      )
      
      sensor_list_string <- paste(sensors_pulled_log, collapse = ", ")
      
    ### 6. Add metadata about the new historical pulls to the trigger_pulls table
      
      trigger_log_df <- data.frame(
        pull_timestamp    = format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
        rows_inserted     = as.integer(nrow(update_clean)),
        sensors_processed = sensor_list_string,
        stringsAsFactors  = FALSE
      )
      
      dbWriteTable(conn=con, name='trigger_pulls', value=trigger_log_df, append=TRUE, row.names=FALSE)
      
      cat(
        format(Sys.time(), "%Y-%m-%d %H:%M:%S"),
        "- Appended historical pull metadata to trigger_pulls table\n"
      )
      
    } else { cat(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "- No sensors required historical updates.\n") }
    }

    ### 7. Disconnect cleanly

    dbDisconnect(con)
    cat(format(Sys.time(), "%Y-%m-%d %H:%M:%S"), "- Script finished executing and disconnected cleanly.\n")
