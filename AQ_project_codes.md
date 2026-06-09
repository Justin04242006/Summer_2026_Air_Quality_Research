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

    ## [1] ""

      Sys.getenv("ODBCSYSINI")

    ## [1] "/etc"

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

    ## Sensor 1 of 1 (ID: 195173), Interval: 2025-05-01 to 2025-05-02

    head(historical_data, 20)

    ##             time_stamp rssi humidity temperature pressure pm2.5_cf_1_a
    ## 1  2025-05-01 00:01:29  -70       43          63   993.77          3.4
    ## 2  2025-05-01 00:03:26  -72       43          63   993.80          4.4
    ## 3  2025-05-01 00:05:26  -72       43          63   993.89          4.5
    ## 4  2025-05-01 00:07:26  -72       43          64   993.94          4.2
    ## 5  2025-05-01 00:09:26  -72       43          63   993.95          3.5
    ## 6  2025-05-01 00:11:26  -74       42          64   993.97          3.4
    ## 7  2025-05-01 00:13:28  -72       42          64   993.99          3.3
    ## 8  2025-05-01 00:15:27  -74       42          65   994.00          4.1
    ## 9  2025-05-01 00:17:27  -74       41          65   994.01          4.0
    ## 10 2025-05-01 00:19:26  -74       41          64   994.01          4.0
    ## 11 2025-05-01 00:21:26  -73       41          65   994.06          4.5
    ## 12 2025-05-01 00:23:27  -72       41          65   994.07          3.8
    ## 13 2025-05-01 00:25:28  -72       41          65   994.08          4.4
    ## 14 2025-05-01 00:27:27  -72       40          65   994.03          4.4
    ## 15 2025-05-01 00:29:27  -74       40          64   993.93          4.0
    ## 16 2025-05-01 00:31:27  -74       40          63   993.88          2.9
    ## 17 2025-05-01 00:33:27  -74       40          64   993.84          3.4
    ## 18 2025-05-01 00:35:28  -74       40          63   993.82          3.2
    ## 19 2025-05-01 00:37:29  -73       41          63   993.84          3.7
    ## 20 2025-05-01 00:39:27  -74       41          63   993.82          3.4
    ##    pm2.5_cf_1_b sensor_index
    ## 1           3.7       195173
    ## 2           3.2       195173
    ## 3           3.4       195173
    ## 4           3.2       195173
    ## 5           2.9       195173
    ## 6           2.1       195173
    ## 7           2.7       195173
    ## 8           3.8       195173
    ## 9           2.7       195173
    ## 10          2.7       195173
    ## 11          3.2       195173
    ## 12          3.0       195173
    ## 13          3.6       195173
    ## 14          3.4       195173
    ## 15          3.3       195173
    ## 16          2.8       195173
    ## 17          2.4       195173
    ## 18          2.6       195173
    ## 19          2.6       195173
    ## 20          2.8       195173

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

    ##   Most_recent_date
    ## 1       2026-06-08

    most_recent_timestamp

    ##   Most_recent_time_stamp
    ## 1    2026-06-08 05:00:15

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

    ## Sensor 1 of 1 (ID: 195173), Interval: 2025-12-04 to 2026-01-03 Sensor 1 of 1
    ## (ID: 195173), Interval: 2026-01-03 to 2026-02-02 Sensor 1 of 1 (ID: 195173),
    ## Interval: 2026-02-02 to 2026-03-04 Sensor 1 of 1 (ID: 195173), Interval:
    ## 2026-03-04 to 2026-04-03 Sensor 1 of 1 (ID: 195173), Interval: 2026-04-03 to
    ## 2026-05-03 Sensor 1 of 1 (ID: 195173), Interval: 2026-05-03 to 2026-05-27

    #Trasnforming and reordering columns to match database structure
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

    ## [1] 0

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

    ## [1] 0

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

    ##     sensor_index            date_created           last_seen
    ## 1           3499     2017-09-20 16:13:03 2023-07-01 13:09:09
    ## 2         265767     2025-02-18 15:48:02 2025-10-31 10:48:43
    ## 3           5588     2017-12-26 18:49:23 2023-06-13 17:27:30
    ## 4           5774     2018-01-02 18:18:30 2023-10-29 10:32:56
    ## 5           7920     2018-02-23 15:28:44 2024-04-16 10:55:05
    ## 6           8476     2018-03-09 14:40:13 2024-01-12 08:05:09
    ## 7          36281     2019-08-01 09:48:46 2025-12-10 17:29:58
    ## 8          41581     2019-11-15 16:07:46 2023-07-11 12:23:13
    ## 9          43955     2019-12-13 14:31:56 2025-10-14 20:03:01
    ## 10         45359     2020-01-02 13:50:31 2025-10-14 19:59:05
    ## 11         51933     2020-04-09 15:27:04 2025-05-31 22:13:13
    ## 12         52011     2020-04-09 17:50:06 2024-10-04 10:28:27
    ## 13         56135     2020-05-20 16:32:08 2023-07-07 06:51:34
    ## 14         57579     2020-06-11 13:26:59 2024-04-22 06:15:36
    ## 15         65791     2020-09-04 11:35:07 2024-10-05 13:12:55
    ## 16         94003     2020-12-14 12:33:26 2024-06-21 23:25:21
    ## 17         95075     2020-12-16 16:08:04 2025-12-10 17:29:13
    ## 18         96079     2020-12-22 16:33:03 2025-06-04 19:32:34
    ## 19         96269     2020-12-23 12:44:33 2024-04-28 20:09:50
    ## 20         96933     2020-12-29 16:29:57 2025-12-10 17:29:28
    ## 21        109598     2021-06-18 17:58:16 2024-06-22 08:34:16
    ## 22        113494     2021-07-09 15:26:20 2023-06-05 15:24:41
    ## 23        124513     2021-08-31 16:41:18 2025-11-17 21:28:59
    ## 24        124649     2021-08-31 16:51:16 2025-12-10 17:30:38
    ## 25        124679     2021-08-31 16:54:55 2023-07-25 09:57:35
    ## 26        124685     2021-08-31 16:55:19 2025-07-31 11:26:58
    ## 27        124715     2021-08-31 16:58:35 2025-12-03 14:22:04
    ## 28        124737     2021-08-31 17:04:31 2025-12-10 17:29:34
    ## 29        124747     2021-08-31 17:05:07 2023-08-22 07:02:44
    ## 30        128349     2021-09-17 15:52:59 2024-12-15 12:47:33
    ## 31        133664     2021-10-19 18:12:05 2025-07-24 15:49:10
    ## 32        137618     2021-11-12 16:37:52 2025-12-10 17:30:59
    ## 33        137622     2021-11-12 16:38:07 2025-12-10 17:30:48
    ## 34        140102     2021-12-08 11:56:48 2024-10-15 05:22:21
    ## 35        144504     2022-02-14 12:36:05 2025-08-16 23:59:33
    ## 36        147803     2022-05-26 11:27:04 2024-09-14 20:27:53
    ## 37        147811     2022-05-26 11:29:08 2024-04-15 14:42:44
    ## 38        147893     2022-05-27 10:58:48 2025-10-15 16:31:28
    ## 39        148029     2022-05-27 13:55:39 2025-12-10 17:29:58
    ## 40        148989     2022-06-10 14:52:34 2024-02-20 08:55:57
    ## 41        149572     2022-06-15 16:31:13 2023-09-09 13:34:19
    ## 42        150042     2022-06-17 16:32:24 2023-10-14 10:23:18
    ## 43        150066     2022-06-20 12:58:18 2025-05-30 18:17:32
    ## 44        151042     2022-06-24 17:39:40 2025-11-22 16:50:46
    ## 45        151050     2022-06-24 17:40:40 2025-12-10 17:29:48
    ## 46        151074     2022-06-24 17:47:19 2024-12-26 00:15:08
    ## 47        151188     2022-06-27 11:46:30 2024-09-10 18:44:43
    ## 48        151606     2022-06-28 19:09:32 2024-05-13 09:42:39
    ## 49        169187     2022-10-31 16:16:20 2025-04-22 15:35:51
    ## 50        171015     2022-11-18 13:25:40 2025-07-31 11:27:08
    ## 51        175227     2023-02-27 14:01:09 2023-09-11 21:15:05
    ## 52        175921     2023-03-07 11:40:47 2024-10-11 20:36:50
    ## 53        176899     2023-03-15 12:26:40 2023-06-01 11:47:27
    ## 54        178649     2023-04-06 15:23:52 2024-12-07 13:23:43
    ## 55        179769     2023-04-24 15:54:16 2025-12-10 17:29:10
    ## 56        180725     2023-05-09 11:54:24 2025-12-10 17:29:51
    ## 57        185239     2023-06-22 15:07:55 2025-12-10 17:30:39
    ## 58        186099     2023-06-28 16:10:46 2025-07-11 14:41:16
    ## 59        187615     2023-07-11 11:05:54 2025-11-22 23:26:38
    ## 60        193661     2023-08-30 12:33:55 2025-11-08 22:12:03
    ## 61        193669     2023-08-30 12:34:48 2025-12-10 17:30:52
    ## 62        193689     2023-08-30 12:35:05 2025-12-10 17:30:22
    ## 63        195181     2023-09-12 14:24:22 2024-10-11 08:39:55
    ## 64        196015     2023-09-18 15:23:41 2024-04-30 22:30:59
    ## 65        196037     2023-09-18 15:30:48 2024-03-10 10:33:46
    ## 66        203303     2023-11-28 12:51:14 2025-12-10 17:30:59
    ## 67        220177     2024-04-02 10:14:08 2025-11-10 08:19:19
    ## 68        220191     2024-04-02 10:14:10 2025-11-10 15:17:39
    ## 69        220209     2024-04-02 10:15:03 2025-11-10 15:15:59
    ## 70        220223     2024-04-02 10:15:06 2025-11-13 11:50:26
    ## 71        220261     2024-04-02 10:54:49 2025-11-12 07:31:57
    ## 72        220521     2024-04-04 10:31:41 2025-11-14 11:56:09
    ## 73        220531     2024-04-04 10:31:58 2025-11-18 12:55:45
    ## 74        220537     2024-04-04 10:32:19 2025-11-22 14:36:36
    ## 75        220553     2024-04-04 10:32:22 2025-11-17 18:45:14
    ## 76        220579     2024-04-04 11:06:59 2025-11-14 07:06:24
    ## 77        220577     2024-04-04 11:06:59 2025-11-12 12:16:03
    ## 78        220583     2024-04-04 11:07:01 2025-11-12 10:07:59
    ## 79        220591     2024-04-04 11:07:08 2025-11-12 12:16:33
    ## 80        220613     2024-04-04 11:07:19 2025-11-13 15:26:10
    ## 81        221165     2024-04-09 10:19:05 2025-11-13 15:30:30
    ## 82        233117     2024-07-22 13:13:42 2025-10-22 20:30:55
    ## 83        244039     2024-09-20 13:15:04 2025-12-10 17:31:00
    ## 84        244057     2024-09-20 13:15:20 2025-12-10 17:30:25
    ## 85        244063     2024-09-20 13:15:25 2025-12-10 17:30:37
    ## 86        244083     2024-09-20 13:15:43 2025-11-30 17:02:28
    ## 87        251599     2024-11-15 11:46:00 2025-11-24 16:28:08
    ## 88        151082 2022-06-24 22:48:15.000          2026-06-09
    ## 89        166645 2022-10-12 18:39:19.000          2026-06-09
    ## 90        175455 2023-02-28 20:46:07.000          2026-06-09
    ## 91        175457 2023-02-28 20:46:11.000          2026-06-09
    ## 92        175483 2023-02-28 20:56:41.000          2026-06-09
    ## 93        148545 2022-06-07 21:40:25.000          2026-06-09
    ## 94        149530 2022-06-15 21:02:28.000          2026-06-09
    ## 95        153638 2022-07-11 17:36:35.000          2026-06-09
    ## 96        120369 2021-08-13 21:37:21.000          2026-06-09
    ## 97        113959 2021-07-12 21:55:18.000          2026-06-09
    ## 98         87741 2020-11-02 18:05:59.000          2026-06-09
    ## 99         96035 2020-12-22 22:25:49.000          2026-06-09
    ## 100        45079 2019-12-30 16:54:39.000          2026-06-09
    ## 101       289288 2025-07-24 18:18:00.000          2026-06-09
    ## 102        36901 2019-08-08 16:51:12.000          2026-06-09
    ## 103         6546 2018-01-26 02:50:16.000          2026-06-09
    ## 104       241195 2024-09-04 23:59:59.000          2026-06-09
    ## 105       226529 2024-05-28 22:24:09.000          2026-06-09
    ## 106       211477 2024-01-24 19:38:11.000          2026-06-09
    ## 107       191181 2023-08-04 16:27:28.000          2026-06-09
    ## 108       192597 2023-08-16 21:09:38.000          2026-06-09
    ## 109       192837 2023-08-21 18:45:10.000          2026-06-09
    ## 110       193671 2023-08-30 17:34:48.000          2026-06-09
    ## 111       193676 2023-08-30 17:34:48.000          2026-06-09
    ## 112       193787 2023-08-30 17:57:12.000          2026-06-05
    ## 113       193801 2023-08-30 17:57:29.000          2026-06-09
    ## 114       195173 2023-09-12 18:54:53.000          2026-06-09
    ## 115       186621 2023-07-03 22:27:14.000          2026-06-09
    ## 116       186941 2023-07-06 17:39:08.000          2026-06-09
    ## 117       187547 2023-07-11 16:03:13.000          2026-06-09
    ## 118       188475 2023-07-17 15:36:19.000          2026-06-09
    ## 119       188749 2023-07-19 17:55:38.000          2026-06-09
    ## 120       189233 2023-07-21 21:18:15.000          2026-06-09
    ## 121       189329 2023-07-21 23:29:29.000          2026-06-09
    ## 122       189821 2023-07-26 21:02:33.000          2026-06-09
    ## 123       182245 2023-05-26 22:43:21.000          2026-06-09
    ## 124       182281 2023-05-30 19:34:04.000          2026-06-09
    ## 125       184435 2023-06-15 23:25:56.000          2026-06-09
    ## 126       184939 2023-06-20 21:56:27.000          2026-06-09
    ## 127       185237 2023-06-22 19:30:27.000          2026-06-09
    ## 128       175419 2023-02-28 20:39:26.000          2026-06-09
    ## 129       175417 2023-02-28 20:39:23.000          2026-06-09
    ## 130       168725 2022-10-27 17:01:44.000          2026-06-09
    ## 131       171075 2022-11-18 19:41:58.000          2026-06-09
    ## 132       148035 2022-05-27 18:56:16.000          2026-06-09
    ## 133       149090 2022-06-13 16:54:58.000          2026-06-09
    ## 134       140390 2021-12-09 23:26:55.000          2026-06-09
    ## 135       292499 2025-08-25 17:07:49.000          2026-06-09
    ## 136        39173 2019-09-17 19:43:13.000          2026-06-09
    ## 137       123003 2021-08-25 21:06:05.000          2026-06-09
    ## 138       124677 2021-08-31 21:54:51.000          2026-06-09
    ## 139       124759 2021-08-31 22:11:22.000          2026-06-09
    ## 140       101643 2021-03-18 15:54:15.000          2026-06-09
    ## 141        88413 2020-11-04 16:50:42.000          2026-06-09
    ## 142         4395 2017-11-17 02:56:56.000          2026-06-09
    ## 143         4404 2017-11-17 02:57:51.000          2026-06-09
    ## 144       282164 2025-06-05 17:46:55.000          2026-06-09
    ## 145       271809 2025-04-02 16:45:58.000          2026-06-09
    ## 146       268129 2025-03-06 19:31:53.000          2026-06-09
    ## 147       244045 2024-09-20 18:15:10.000          2026-06-09
    ## 148       244051 2024-09-20 18:15:15.000          2026-06-09
    ## 149       244073 2024-09-20 18:15:35.000          2026-06-09
    ##                                            name location_type      model
    ## 1                             Ukrainian Village             1      PA-II
    ## 2                               Lyons and Dodge             1  PA-II-ZEN
    ## 3                                 1138 PLYMOUTH             1      PA-II
    ## 4                                       LVEJO_5             1      PA-II
    ## 5                                       LVEJO_5             1      PA-II
    ## 6                                 SASA_PA2_SL_W             1      PA-II
    ## 7                                 Bridgeport SW             1      PA-II
    ## 8                                 Otis, Chicago             1   PA-II-SD
    ## 9                  6147 N Maplewood Living Room             0      PA-II
    ## 10                             Beke-Sato Office             0       PA-I
    ## 11                               Gladstone Park             1   PA-II-SD
    ## 12                                CPS Burr Elem             1   PA-II-SD
    ## 13                           West Humboldt Park             1      PA-II
    ## 14                       Edgewater Condo Indoor             0       PA-I
    ## 15                                         Loop             1      PA-II
    ## 16                               38th and Damen             1      PA-II
    ## 17                             37TH &amp; DAMEN             1      PA-II
    ## 18                             45th and Ashland             1      PA-II
    ## 19   S Franciso Ave, W Pershing rd Intersection             1      PA-II
    ## 20                             wrightwood-drake             1      PA-II
    ## 21                            Hubbard &amp; May             1      PA-II
    ## 22                                        LVEJO             1      PA-II
    ## 23                             Space &amp; Time             0       PA-I
    ## 24                                   LUC_CARE 2             1      PA-II
    ## 25                                    LUC_CARE3             1      PA-II
    ## 26                                   LUC_CARE 9             1      PA-II
    ## 27                                  NEIU-IN-1.1             0       PA-I
    ## 28                                    LUC_CARE6             1      PA-II
    ## 29                                    LUC_CARE7             1      PA-II
    ## 30                 South Deering, Chicago 60617             1   PA-II-SD
    ## 31                                   Prairieone             0      PA-II
    ## 32                 CARE_SES Teaching Lab Indoor             0       PA-I
    ## 33                               LUC BVM Indoor             0       PA-I
    ## 34                           Archer and 39th Pl             1      PA-II
    ## 35                                      LUC BVM             1   PA-II-SD
    ## 36                           14th &amp; 51st Ct             1 PA-II-FLEX
    ## 37                        54th Ct &amp; 31st St             1 PA-II-FLEX
    ## 38                       59th Ave &amp; 38th St             1 PA-II-FLEX
    ## 39                      St. Louis &amp; Addison             1 PA-II-FLEX
    ## 40               Hyde Park - 53rd and Ingleside             1   PA-II-SD
    ## 41                               IIT AH Outdoor             1   PA-II-SD
    ## 42                                      LUC BVM             1      PA-II
    ## 43                               LUC Doyle Hall             1      PA-II
    ## 44                                     LVEJO-R2             1      PA-II
    ## 45                                     LVEJO_R8             1      PA-II
    ## 46                                     LVEJO_R4             1      PA-II
    ## 47                  Loyola WTC Shuttle Bus Stop             1      PA-II
    ## 48                                     LVEJO_R3             1      PA-II
    ## 49                          Rise Gardens Office             0 PA-I-TOUCH
    ## 50                             LUC_CARE_indoor2             0 PA-I-TOUCH
    ## 51                                     NEIU-3.3             1 PA-II-FLEX
    ## 52                          Pilsen - Loomis St.             1 PA-II-FLEX
    ## 53                                R5-ARD-SL1313             0   PA-II-SD
    ## 54                                     LVEJO-R1             1      PA-II
    ## 55                                   blackstone             0 PA-II-FLEX
    ## 56                                       WWAYSK             1 PA-II-FLEX
    ## 57                              RF200AshlandOut             1  PA-II-ZEN
    ## 58                                  Taylor Park             1 PA-II-FLEX
    ## 59                                      Kenwood             1      PA-II
    ## 60                              German Shepherd             1 PA-II-FLEX
    ## 61                                          Bug             1 PA-II-FLEX
    ## 62                                         Worm             1 PA-II-FLEX
    ## 63                          18th and California             1   PA-II-SD
    ## 64                         LUC Jesuit Residence             1      PA-II
    ## 65                          Student Apartment 1             1      PA-II
    ## 66                         Kimball &amp; Cullom             1 PA-II-FLEX
    ## 67                                    MCC16 OUT             1 PA-II-FLEX
    ## 68                                    MCC10 OUT             1 PA-II-FLEX
    ## 69                                     MCC10 IN             0 PA-II-FLEX
    ## 70                                     MCC16 IN             0 PA-II-FLEX
    ## 71                                    MCC09 OUT             1 PA-II-FLEX
    ## 72                                    MCC22 OUT             1 PA-II-FLEX
    ## 73                                    MCC21 OUT             1 PA-II-FLEX
    ## 74                                    MCC07 OUT             1 PA-II-FLEX
    ## 75                                     MCC21 IN             0 PA-II-FLEX
    ## 76                                     MCC22 IN             0 PA-II-FLEX
    ## 77                                    MCC06 OUT             1 PA-II-FLEX
    ## 78                                     MCC01 IN             0 PA-II-FLEX
    ## 79                                     MCC06 IN             1 PA-II-FLEX
    ## 80                                    MCC20 OUT             1 PA-II-FLEX
    ## 81                                     MCC20 IN             0 PA-II-FLEX
    ## 82                                      Ashland             0  PA-II-ZEN
    ## 83        S Damen Ave and 54th - N4EJ's network             1 PA-II-FLEX
    ## 84                                     Iron Lab             1 PA-II-FLEX
    ## 85                Back of the Yards High School             1 PA-II-FLEX
    ## 86    W 35th St and S Archer Ave - N4EJ Network             1 PA-II-FLEX
    ## 87                                   WWJ-222405             1 PA-II-FLEX
    ## 88                  Loyola LSC Shuttle Bus Stop             0      PA-II
    ## 89                            LUC Lewis Library             1       PA-I
    ## 90                                  LUC_CARE_13             0      PA-II
    ## 91                                  LUC_CARE_14             0      PA-II
    ## 92                           LUC_CARE_6018North             0      PA-II
    ## 93                                       ISP NU             0 PA-II-FLEX
    ## 94                           Hyde Park Egandale             0   PA-II-SD
    ## 95                                  Purple-HP-1             0   PA-II-SD
    ## 96               La Grange Country Club Section             0      PA-II
    ## 97                               MiraculousMess             1      PA-II
    ## 98                    Ukrainian Village Chicago             0      PA-II
    ## 99                             36th and Paulina             0      PA-II
    ## 100                                  Mulford in             1       PA-I
    ## 101                          Wildwood (Chicago)             0 PA-II-FLEX
    ## 102                                         ODE             1   PA-II-SD
    ## 103                       West of St Ann School             1       PA-I
    ## 104                  737 W Washington Boulevard             1 PA-I-TOUCH
    ## 105                                    LVEJO_R9             0   PA-II-SD
    ## 106                            1965 W. Pershing             1  PA-II-ZEN
    ## 107                                   Roof deck             0      PA-II
    ## 108                              3526 N Halsted             0  PA-II-ZEN
    ## 109                              mscjme-purple1             0  PA-II-ZEN
    ## 110                                      Girafa             0 PA-II-FLEX
    ## 111                                     Rooster             0 PA-II-FLEX
    ## 112                                   Chihuahua             0 PA-II-FLEX
    ## 113                                       Otter             0 PA-II-FLEX
    ## 114                          North Town Village             0 PA-II-FLEX
    ## 115                             Tech DB46 Laser             1 PA-II-FLEX
    ## 116                             McCleason Manor             0 PA-II-FLEX
    ## 117                       Evanston West End AQM             0      PA-II
    ## 118                           OAK PARK Lyman Av             0 PA-II-FLEX
    ## 119                                 Berwyn East             1 PA-I-TOUCH
    ## 120                          tri-taylor-chicago             0 PA-II-FLEX
    ## 121                      Sweet Water Foundation             0      PA-II
    ## 122                                  South Blvd             0  PA-II-ZEN
    ## 123                                    Lawndale             0 PA-II-FLEX
    ## 124                                        Home             0 PA-II-FLEX
    ## 125                                  Ravenswood             0      PA-II
    ## 126                               Skokie Lowell             0      PA-II
    ## 127                       Ukranian-Village-f296             0 PA-II-FLEX
    ## 128                       CARE_SES Teaching Lab             0      PA-II
    ## 129                                 LUC_CARE_15             0      PA-II
    ## 130                                       Clark             1 PA-I-TOUCH
    ## 131                            LUC_CARE_indoor1             1 PA-I-TOUCH
    ## 132                                     H $ 109             0 PA-II-FLEX
    ## 133                              Mitchell House             0   PA-II-SD
    ## 134         1742 w Winona st, Chicago, IL 60640             1       PA-I
    ## 135                    Clarence Ave Oak Park IL             1 PA-II-FLEX
    ## 136                                   West Lawn             0      PA-II
    ## 137                                  LUC Malibu             0      PA-II
    ## 138                                   LUC_CARE8             0      PA-II
    ## 139                                 NEIU-IN-2.1             1       PA-I
    ## 140                                    NEIU-1.2             0      PA-II
    ## 141                                   Lake View             1       PA-I
    ## 142                            West Rigers Park             0   PA-II-SD
    ## 143                               Mulford Manor             0   PA-II-SD
    ## 144                                  North HVAC             1 PA-II-FLEX
    ## 145                              NPU_Chicago_IL             0 PA-II-FLEX
    ## 146                         2422 S Harding Ave.             1 PA-I-TOUCH
    ## 147           35th and Wolcott - N4EJ's Network             0 PA-II-FLEX
    ## 148                              Hoyne and 32nd             0 PA-II-FLEX
    ## 149 S 37th St and S Wallace St - N4EJ's Network             0 PA-II-FLEX
    ##                                                          hardware latitude
    ## 1                               2.0+1M+BME280+PMSX003-B+PMSX003-A 41.89270
    ## 2     3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 42.05020
    ## 3                                         2.0+PMSX003-B+PMSX003-A 41.86889
    ## 4                                  2.0+BME280+PMSX003-B+PMSX003-A 41.84035
    ## 5                               2.0+1M+BME280+PMSX003-B+PMSX003-A 41.84948
    ## 6                                         2.0+PMSX003-B+PMSX003-A 41.86121
    ## 7                                  2.0+BME280+PMSX003-B+PMSX003-A 41.82936
    ## 8           2.0+OPENLOG+NO-DISK+DS3231+BME280+PMSX003-B+PMSX003-A 41.86100
    ## 9                           2.0+BME280+BME68X+PMSX003-B+PMSX003-A 41.99368
    ## 10                                           2.0+BME280+PMSX003-A 41.99365
    ## 11                       2.0+OPENLOG+15582 MB+PMSX003-B+PMSX003-A 41.98188
    ## 12          2.0+OPENLOG+NO-DISK+DS3231+BME280+PMSX003-B+PMSX003-A 41.91195
    ## 13                                 2.0+BME280+PMSX003-B+PMSX003-A 41.90253
    ## 14                                           2.0+BME280+PMSX003-A 41.99175
    ## 15                                 2.0+BME280+PMSX003-B+PMSX003-A 41.88100
    ## 16                          2.0+BME280+BME68X+PMSX003-B+PMSX003-A 41.82482
    ## 17                          2.0+BME280+BME68X+PMSX003-B+PMSX003-A 41.82671
    ## 18                                 2.0+BME280+PMSX003-B+PMSX003-A 41.81229
    ## 19                                 2.0+BME280+PMSX003-B+PMSX003-A 41.82272
    ## 20                                 2.0+BME280+PMSX003-B+PMSX003-A 41.92740
    ## 21                                 2.0+BME280+PMSX003-B+PMSX003-A 41.89003
    ## 22                                        2.0+PMSX003-B+PMSX003-A 41.84669
    ## 23                                           2.0+BME280+PMSX003-A 41.95386
    ## 24                                 2.0+BME280+PMSX003-B+PMSX003-A 41.68474
    ## 25                                 2.0+BME280+PMSX003-B+PMSX003-A 41.74125
    ## 26                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99153
    ## 27                                           2.0+BME280+PMSX003-A 41.98033
    ## 28                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99443
    ## 29                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99469
    ## 30         2.0+OPENLOG+15476 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.71180
    ## 31                                 2.0+BME280+PMSX003-B+PMSX003-A 41.85950
    ## 32                                           2.0+BME280+PMSX003-A 41.99790
    ## 33                                           2.0+BME280+PMSX003-A 41.99812
    ## 34                          2.0+BME280+BME68X+PMSX003-B+PMSX003-A 41.82193
    ## 35                2.0+OPENLOG+15476 MB+DS3231+PMSX003-B+PMSX003-A 41.99816
    ## 36   3.0+OPENLOG+NO-DISK+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 41.86173
    ## 37  3.0+OPENLOG+31037 MB+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 41.83537
    ## 38  3.0+OPENLOG+31037 MB+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 41.82380
    ## 39                   3.0+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 41.94750
    ## 40                 2.0+OPENLOG+NO-DISK+DS3231+PMSX003-B+PMSX003-A 41.79884
    ## 41                        2.0+OPENLOG+NO-DISK+PMSX003-B+PMSX003-A 41.83633
    ## 42                                        2.0+PMSX003-B+PMSX003-A 41.99813
    ## 43                                        2.0+PMSX003-B+PMSX003-A 42.00173
    ## 44                                 2.0+BME280+PMSX003-B+PMSX003-A 41.84156
    ## 45                                 2.0+BME280+PMSX003-B+PMSX003-A 41.84969
    ## 46                                 2.0+BME280+PMSX003-B+PMSX003-A 41.85321
    ## 47                                 2.0+BME280+PMSX003-B+PMSX003-A 41.89734
    ## 48                                 2.0+BME280+PMSX003-B+PMSX003-A 41.83481
    ## 49                                     3.0+BME68X+KX122+PMSX003-A 41.96392
    ## 50                                     3.0+BME68X+KX122+PMSX003-A 41.99149
    ## 51        3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A-R+PMSX003-B 41.98026
    ## 52          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.85944
    ## 53         2.0+OPENLOG+15833 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.87800
    ## 54                                 2.0+BME280+PMSX003-B+PMSX003-A 41.83401
    ## 55         3.0+OPENLOG+31037 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.79859
    ## 56          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 42.04800
    ## 57    3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 41.88442
    ## 58          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.90292
    ## 59                                 2.0+BME280+PMSX003-B+PMSX003-A 41.81122
    ## 60                       3.0+OPENLOG+31954 MB+PMSX003-A+PMSX003-B 41.70555
    ## 61                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.70301
    ## 62                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69673
    ## 63         2.0+OPENLOG+15485 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.85692
    ## 64                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99719
    ## 65                                        2.0+PMSX003-B+PMSX003-A 42.00102
    ## 66                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.95929
    ## 67                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68907
    ## 68          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69024
    ## 69          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69025
    ## 70          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68909
    ## 71                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68993
    ## 72          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.70539
    ## 73                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68487
    ## 74          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.64904
    ## 75          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68488
    ## 76          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.70540
    ## 77                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68941
    ## 78          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68993
    ## 79          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68941
    ## 80                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68968
    ## 81          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.68961
    ## 82    3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 41.93308
    ## 83                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.79569
    ## 84                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.82687
    ## 85                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.80721
    ## 86                          3.0+RV3028+BME68X+PMSX003-A+PMSX003-B 41.83044
    ## 87          3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.90699
    ## 88                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99897
    ## 89                                           2.0+BME280+PMSX003-A 41.89737
    ## 90                                 2.0+BME280+PMSX003-B+PMSX003-A 41.71845
    ## 91                                 2.0+BME280+PMSX003-B+PMSX003-A 41.72854
    ## 92                                 2.0+BME280+PMSX003-B+PMSX003-A 41.99145
    ## 93  3.0+OPENLOG+31037 MB+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 42.05839
    ## 94                2.0+OPENLOG+15476 MB+DS3231+PMSX003-B+PMSX003-A 41.79880
    ## 95          2.0+OPENLOG+NO-DISK+DS3231+BME280+PMSX003-B+PMSX003-A 41.79252
    ## 96                                 2.0+BME280+PMSX003-B+PMSX003-A 41.79734
    ## 97                                 2.0+BME280+PMSX003-B+PMSX003-A 42.03856
    ## 98                                        2.0+PMSX003-B+PMSX003-A 41.90011
    ## 99                                 2.0+BME280+PMSX003-B+PMSX003-A 41.82865
    ## 100                                          2.0+BME280+PMSX003-A 42.02250
    ## 101        3.0+OPENLOG+31268 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 42.00190
    ## 102        2.0+OPENLOG+15802 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.88437
    ## 103                                          2.0+BME280+PMSX003-A 41.85661
    ## 104                                    3.0+BME68X+KX122+PMSX003-A 41.88350
    ## 105        2.0+OPENLOG+31037 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.84956
    ## 106   3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 41.82234
    ## 107                                2.0+BME280+PMSX003-B+PMSX003-A 41.89528
    ## 108   3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 41.94632
    ## 109   3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 41.93369
    ## 110        3.0+OPENLOG+30945 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69997
    ## 111        3.0+OPENLOG+31016 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.72559
    ## 112        3.0+OPENLOG+31954 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69659
    ## 113        3.0+OPENLOG+31954 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.69767
    ## 114         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.90669
    ## 115         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 42.05857
    ## 116         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 42.04832
    ## 117                                2.0+BME280+PMSX003-B+PMSX003-A 42.03857
    ## 118         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.87748
    ## 119                                    3.0+BME68X+KX122+PMSX003-A 41.97880
    ## 120         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.86840
    ## 121                                2.0+BME280+PMSX003-B+PMSX003-A 41.78979
    ## 122   3.0+OPENLOG+NO-DISK+RV3028+BME68X+KX122+PMSX003-A+PMSX003-B 42.02816
    ## 123        3.0+OPENLOG+63864 MB+RV3028+BME68X+PMSX003-A+PMSX003-B 41.91638
    ## 124         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.93747
    ## 125                                2.0+BME280+PMSX003-B+PMSX003-A 41.96173
    ## 126                                2.0+BME280+PMSX003-B+PMSX003-A 42.02653
    ## 127         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.89235
    ## 128                                       2.0+PMSX003-B+PMSX003-A 41.99786
    ## 129                                2.0+BME280+PMSX003-B+PMSX003-A 41.97209
    ## 130                                    3.0+BME68X+KX122+PMSX003-A 41.97068
    ## 131                                    3.0+BME68X+KX122+PMSX003-A 41.99466
    ## 132  3.0+OPENLOG+NO-DISK+DS3231+BME280+BME68X+PMSX003-A+PMSX003-B 41.69688
    ## 133        2.0+OPENLOG+15476 MB+DS3231+BME280+PMSX003-B+PMSX003-A 41.69555
    ## 134                                          2.0+BME280+PMSX003-A 41.97526
    ## 135         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.87487
    ## 136                                2.0+BME280+PMSX003-B+PMSX003-A 41.76776
    ## 137                                2.0+BME280+PMSX003-B+PMSX003-A 41.99123
    ## 138                                       2.0+PMSX003-B+PMSX003-A 41.99066
    ## 139                                          2.0+BME280+PMSX003-A 41.98036
    ## 140                                2.0+BME280+PMSX003-B+PMSX003-A 41.98028
    ## 141                                          2.0+BME280+PMSX003-A 41.94883
    ## 142         2.0+OPENLOG+NO-DISK+DS3231+BME280+PMSX003-B+PMSX003-A 41.99566
    ## 143        2.0+OPENLOG+16012 MB+DS3231+BME280+PMSX003-B+PMSX003-A 42.02362
    ## 144         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.83377
    ## 145         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.97500
    ## 146                                    3.0+BME68X+KX122+PMSX003-A 41.84722
    ## 147         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.82961
    ## 148         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.83571
    ## 149         3.0+OPENLOG+NO-DISK+RV3028+BME68X+PMSX003-A+PMSX003-B 41.82723
    ##     longitude sensor_activity_status
    ## 1   -87.68569               Inactive
    ## 2   -87.69881               Inactive
    ## 3   -87.62943               Inactive
    ## 4   -87.73412               Inactive
    ## 5   -87.69537               Inactive
    ## 6   -87.62989               Inactive
    ## 7   -87.65014               Inactive
    ## 8   -87.66320               Inactive
    ## 9   -87.69320               Inactive
    ## 10  -87.69304               Inactive
    ## 11  -87.77780               Inactive
    ## 12  -87.66863               Inactive
    ## 13  -87.73614               Inactive
    ## 14  -87.65723               Inactive
    ## 15  -87.62566               Inactive
    ## 16  -87.67512               Inactive
    ## 17  -87.67516               Inactive
    ## 18  -87.66579               Inactive
    ## 19  -87.69708               Inactive
    ## 20  -87.71574               Inactive
    ## 21  -87.65576               Inactive
    ## 22  -87.70752               Inactive
    ## 23  -87.71078               Inactive
    ## 24  -87.53683               Inactive
    ## 25  -87.55403               Inactive
    ## 26  -87.65750               Inactive
    ## 27  -87.71664               Inactive
    ## 28  -87.66211               Inactive
    ## 29  -87.67048               Inactive
    ## 30  -87.55293               Inactive
    ## 31  -87.62050               Inactive
    ## 32  -87.65682               Inactive
    ## 33  -87.65649               Inactive
    ## 34  -87.69146               Inactive
    ## 35  -87.65659               Inactive
    ## 36  -87.75328               Inactive
    ## 37  -87.75943               Inactive
    ## 38  -87.77058               Inactive
    ## 39  -87.71516               Inactive
    ## 40  -87.60307               Inactive
    ## 41  -87.62723               Inactive
    ## 42  -87.65679               Inactive
    ## 43  -87.65813               Inactive
    ## 44  -87.73148               Inactive
    ## 45  -87.71228               Inactive
    ## 46  -87.69096               Inactive
    ## 47  -87.62692               Inactive
    ## 48  -87.71722               Inactive
    ## 49  -87.67387               Inactive
    ## 50  -87.65756               Inactive
    ## 51  -87.71705               Inactive
    ## 52  -87.66145               Inactive
    ## 53  -87.63050               Inactive
    ## 54  -87.71916               Inactive
    ## 55  -87.59103               Inactive
    ## 56  -87.74695               Inactive
    ## 57  -87.81662               Inactive
    ## 58  -87.78548               Inactive
    ## 59  -87.59382               Inactive
    ## 60  -87.53752               Inactive
    ## 61  -87.53435               Inactive
    ## 62  -87.54421               Inactive
    ## 63  -87.69554               Inactive
    ## 64  -87.65739               Inactive
    ## 65  -87.66438               Inactive
    ## 66  -87.71308               Inactive
    ## 67  -87.53751               Inactive
    ## 68  -87.53870               Inactive
    ## 69  -87.53867               Inactive
    ## 70  -87.53746               Inactive
    ## 71  -87.54111               Inactive
    ## 72  -87.53873               Inactive
    ## 73  -87.53506               Inactive
    ## 74  -87.53521               Inactive
    ## 75  -87.53497               Inactive
    ## 76  -87.53870               Inactive
    ## 77  -87.53270               Inactive
    ## 78  -87.54108               Inactive
    ## 79  -87.53263               Inactive
    ## 80  -87.53683               Inactive
    ## 81  -87.53670               Inactive
    ## 82  -87.66876               Inactive
    ## 83  -87.67447               Inactive
    ## 84  -87.65945               Inactive
    ## 85  -87.67779               Inactive
    ## 86  -87.67709               Inactive
    ## 87  -87.70682               Inactive
    ## 88  -87.65796                 Active
    ## 89  -87.62730                 Active
    ## 90  -87.55261                 Active
    ## 91  -87.67574                 Active
    ## 92  -87.65736                 Active
    ## 93  -87.67830                 Active
    ## 94  -87.60301                 Active
    ## 95  -87.58896                 Active
    ## 96  -87.87669                 Active
    ## 97  -87.69582                 Active
    ## 98  -87.68288                 Active
    ## 99  -87.66791                 Active
    ## 100 -87.76374                 Active
    ## 101 -87.77226                 Active
    ## 102 -87.61652                 Active
    ## 103 -87.68180                 Active
    ## 104 -87.64683                 Active
    ## 105 -87.69527                 Active
    ## 106 -87.67669                 Active
    ## 107 -87.67359                 Active
    ## 108 -87.64966                 Active
    ## 109 -87.71961                 Active
    ## 110 -87.54142                 Active
    ## 111 -87.53785                 Active
    ## 112 -87.54282                 Active
    ## 113 -87.53986                 Active
    ## 114 -87.64716                 Active
    ## 115 -87.67526                 Active
    ## 116 -87.69478                 Active
    ## 117 -87.69540                 Active
    ## 118 -87.77712                 Active
    ## 119 -87.65349                 Active
    ## 120 -87.68851                 Active
    ## 121 -87.62727                 Active
    ## 122 -87.68209                 Active
    ## 123 -87.71977                 Active
    ## 124 -87.66412                 Active
    ## 125 -87.66714                 Active
    ## 126 -87.73663                 Active
    ## 127 -87.68557                 Active
    ## 128 -87.65700                 Active
    ## 129 -87.86852                 Active
    ## 130 -87.66789                 Active
    ## 131 -87.66132                 Active
    ## 132 -87.53236                 Active
    ## 133 -87.67751                 Active
    ## 134 -87.67274                 Active
    ## 135 -87.79071                 Active
    ## 136 -87.73013                 Active
    ## 137 -87.65459                 Active
    ## 138 -87.66854                 Active
    ## 139 -87.71658                 Active
    ## 140 -87.71708                 Active
    ## 141 -87.66293                 Active
    ## 142 -87.69185                 Active
    ## 143 -87.76341                 Active
    ## 144 -87.83036                 Active
    ## 145 -87.71200                 Active
    ## 146 -87.72366                 Active
    ## 147 -87.67284                 Active
    ## 148 -87.67790                 Active
    ## 149 -87.64105                 Active

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

    ##    pull_id      pull_timestamp rows_inserted
    ## 1        1 2026-06-05 19:48:07      81186601
    ## 2        2 2026-06-05 15:07:59         43172
    ## 3        3 2026-06-06 09:28:04         33172
    ## 4        4 2026-06-06 23:41:23         25369
    ## 5        5 2026-06-08 00:02:07         43969
    ## 6        6 2026-06-09 14:57:42      81580295
    ## 7        7 2026-06-09 14:59:44      81704301
    ## 8        8 2026-06-09 15:06:01      81828307
    ## 9        9 2026-06-09 15:16:38      82324331
    ## 10      10 2026-06-09 15:43:45      82448337
    ##                                                                                                                                                                                                                                                                                                                                                                                                                                                                                     sensors_processed
    ## 1                                                                                                                                                                                                                                                                                                                                                                                                                                                                         Initial historical backfill
    ## 2  151082, 166645, 175455, 175457, 175483, 148545, 149530, 153638, 120369, 113959, 87741, 96035, 45079, 289288, 36901, 6546, 241195, 226529, 211477, 191181, 192597, 192837, 193671, 193676, 193787, 193801, 195173, 186621, 186941, 187547, 188475, 188749, 189233, 189329, 189821, 182245, 182281, 184435, 184939, 185237, 175419, 175417, 168725, 171075, 148035, 149090, 140390, 292499, 39173, 123003, 124677, 124759, 101643, 88413, 4395, 4404, 282164, 271809, 268129, 244045, 244051, 244073
    ## 3          151082, 166645, 175455, 175457, 175483, 148545, 149530, 153638, 120369, 113959, 87741, 96035, 45079, 289288, 36901, 6546, 241195, 226529, 211477, 191181, 192597, 192837, 193671, 193676, 193801, 195173, 186621, 186941, 187547, 188475, 188749, 189233, 189329, 189821, 182245, 182281, 184435, 184939, 185237, 175419, 175417, 168725, 171075, 148035, 149090, 140390, 292499, 39173, 123003, 124677, 124759, 101643, 88413, 4395, 4404, 282164, 271809, 268129, 244045, 244051, 244073
    ## 4          151082, 166645, 175455, 175457, 175483, 148545, 149530, 153638, 120369, 113959, 87741, 96035, 45079, 289288, 36901, 6546, 241195, 226529, 211477, 191181, 192597, 192837, 193671, 193676, 193801, 195173, 186621, 186941, 187547, 188475, 188749, 189233, 189329, 189821, 182245, 182281, 184435, 184939, 185237, 175419, 175417, 168725, 171075, 148035, 149090, 140390, 292499, 39173, 123003, 124677, 124759, 101643, 88413, 4395, 4404, 282164, 271809, 268129, 244045, 244051, 244073
    ## 5          151082, 166645, 175455, 175457, 175483, 148545, 149530, 153638, 120369, 113959, 87741, 96035, 45079, 289288, 36901, 6546, 241195, 226529, 211477, 191181, 192597, 192837, 193671, 193676, 193801, 195173, 186621, 186941, 187547, 188475, 188749, 189233, 189329, 189821, 182245, 182281, 184435, 184939, 185237, 175419, 175417, 168725, 171075, 148035, 149090, 140390, 292499, 39173, 123003, 124677, 124759, 101643, 88413, 4395, 4404, 282164, 271809, 268129, 244045, 244051, 244073
    ## 6                                                                                                                                                                                                                                                                                                                                                                                                                                                                         Initial historical backfill
    ## 7                                                                                                                                                                                                                                                                                                                                                                                                                                                                         Initial historical backfill
    ## 8                                                                                                                                                                                                                                                                                                                                                                                                                                                                         Initial historical backfill
    ## 9                                                                                                                                                                                                                                                                                                                                                                                                                                                                         Initial historical backfill
    ## 10                                                                                                                                                                                                                                                                                                                                                                                                                                                                        Initial historical backfill

After making a new ER diagram for the desired databse structure and
considering the high-level workflow of the script, I wrote the script
(with some syntactical help from ChatGPT and conceptual help from my
professor).

This is what the script accomplishes at a high level:

1.  Connect to the clearlab\_purpleair SQL server
2.  queries and vectorizes sensor IDs that are “Active” based on the
    flag column (active=last\_seen in the database more recent than
    6/1/26)
3.  pulls updated last\_seen timestamps from the PurpleAir API for all
    active sensors
4.  For each active sensor, its API last\_seen date is compared to the
    last\_seen date already in the database for that sensor. If the API
    last\_seen date is more recent, the last\_seen value in the
    corresponding row of the sensor\_infomation table is replaced with
    the updated API last\_seen date.
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
