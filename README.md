#PL/SQL APEX Library for Deutsche Bahn's Timetable API
This PL/SQL package is a native wrapper for Oracle Database, especially when combined with Oracle APEX,
for the [Deutsche Bahn Fahrplan / Timetable REST API](http://data.deutschebahn.com/apis/fahrplan).

It web service requests are based on APEX_WEB_SERVICE package which comes with APEX.

The package include all web service functions which are included in the Fahrplan API. The functions can be summarized to:

- Functions that will return the plain JSON content of the web service response
- Functions that creates are APEX collection, based on the web service response
- Functions that will return the web service response pipelined, for using in table functions


##Demo
A demo application is available under
https://apex.danielh.de/ords/f?p=BAHN_FAHRPLAN

And of course you find a APEX export of it in "demo" folder. To use it just import it and then go through the Installation steps.
Under Shared Components --> Edit Application Definition --> Substitutions set "F_API_AUTH_KEY" to your API Key which you get from Deutsche Bahn.


##Changelog
####1.0 - Initial Release


##Installation
###Database ACL
All web service requests have as base domain "open-api.bahn.de". For that you need a ACL, so you are allowed to connect to this host.

```language-sql
DECLARE

  l_filename VARCHAR2(30) := 'open-api.bahn.de.xml';
  l_schema   VARCHAR2(20) := '<SCHEMA>';

BEGIN

  BEGIN
    dbms_network_acl_admin.drop_acl(acl => l_filename);

  EXCEPTION
    WHEN OTHERS THEN
      NULL;
  END;

  dbms_network_acl_admin.create_acl(acl         => l_filename,
                                    description => 'All requests to open-api.bahn.de',
                                    principal   => l_schema,
                                    is_grant    => TRUE,
                                    privilege   => 'connect');

  dbms_network_acl_admin.add_privilege(acl       => l_filename,
                                       principal => l_schema,
                                       is_grant  => TRUE,
                                       privilege => 'resolve');

  dbms_network_acl_admin.assign_acl(acl        => l_filename,
                                    host       => 'open-api.bahn.de',
                                    lower_port => 443);
  dbms_network_acl_admin.assign_acl(acl        => l_filename,
                                    host       => 'open-api.bahn.de',
                                    lower_port => 80);

END;
/
```

###Oracle SSL Wallet
To communicate with the Deutsche Bahn Fahrplan API (open-api.bahn.de) over HTTPS, a SSL Wallet is needed for database which contains the certificates from open-api.bahn.de.

As of time writing this the Fahrplan API is still available unencrypted over HTTP, but this will change an monday 2016-02-29. Then I will include a ready to go wallet here on Github.

For manually creating the wallet, either use Oracle Wallet Manager or create the wallet with openssl utils like:
- Grab the certificates from open-api.bahn.de via your browser
- Create the wallet on command line
```shell
openssl pkcs12 -export -in open-api.bahn.de.cert -out ewallet.p12 -nokey
```
- Place the wallet file on your database server
- Change the wallet path and password in the package specification under "Fahrplan REST API defaults"

###Compile the PL/SQL package
Connect to your database and compile the package spec and body (bahn_fahrplan_api.pks & bahn_fahrplan_api.pkb)


##Usage
###Location Service of Fahrplan API
####Plain JSON response
```language-sql
DECLARE
  l_response_json CLOB;
BEGIN
  l_response_json := bahn_fahrplan_api.get_location_name_json(i_api_auth_key  => 'YOUR_API_KEY',
                                                              i_language      => 'en', -- default en
                                                              i_search_string => 'Regensburg');
END;
```
####APEX collections
- Get stations
```language-sql
-- create collection
BEGIN
  bahn_fahrplan_api.get_location_stations_apex(i_api_auth_key   => 'YOUR_API_KEY',
                                               i_language       => 'en', -- default en
                                               i_search_string  => 'Regensburg',
                                               i_apex_coll_name => 'STATIONS'); -- default 'STATIONS'
END;
-- Select from collection
SELECT ac.c001 AS station_name,
       ac.c002 AS station_id,
       ac.c003 AS longitude,
       ac.c004 AS latitude
  FROM apex_collections ac
 WHERE ac.collection_name = 'STATIONS'
```
- Get other relevant locations
```language-sql
-- create collection
BEGIN
  bahn_fahrplan_api.get_location_otherloc_apex(i_api_auth_key   => 'YOUR_API_KEY',
                                               i_language       => 'en', -- default en
                                               i_search_string  => 'Regensburg',
                                               i_apex_coll_name => 'LOCATIONS'); -- default 'LOCATIONS'
END;
-- Select from collection
SELECT ac.c001 AS loc_name,
       ac.c002 AS loc_type,
       ac.c003 AS longitude,
       ac.c004 AS latitude
  FROM apex_collections ac
 WHERE ac.collection_name = 'LOCATIONS'
 ```
####Pipelined Function for use in table function
- Get stations
```language-sql
SELECT stations.station_name,
       stations.station_id,
       stations.longitude,
       stations.latitude
  FROM TABLE(bahn_fahrplan_api.get_location_stations_pipe(i_api_auth_key  => 'YOUR_API_KEY',
                                                          i_language      => 'en', -- default en
                                                          i_search_string => 'Regensburg')) stations
```
- Get other relevant locations
```language-sql
SELECT locations.loc_name,
       locations.loc_type,
       locations.longitude,
       locations.latitude
  FROM TABLE(bahn_fahrplan_api.get_location_otherloc_pipe(i_api_auth_key  => 'YOUR_API_KEY',
                                                          i_language      => 'en', -- default en
                                                          i_search_string => 'Regensburg')) locations
```

###Stationboard services (Departure & Arrival Board)
####Plain JSON response
```language-sql
DECLARE
  l_response_json CLOB;
BEGIN
  -- departure board
  l_response_json := bahn_fahrplan_api.get_departure_board_json(i_api_auth_key => 'YOUR_API_KEY',
                                                                i_language     => 'en', -- default en
                                                                i_station_id   => 1111111, -- from location service from before
                                                                i_date_time    => NULL); -- valid date with or without time
  -- arrival board
  l_response_json := bahn_fahrplan_api.get_arrival_board_json(i_api_auth_key => 'YOUR_API_KEY',
                                                              i_language     => 'en', -- default en
                                                              i_station_id   => 1111111, -- from location service from before
                                                              i_date_time    => NULL); -- valid date with or without time
END;
```
####APEX collections
-- Departure board
```language-sql
-- create collection
BEGIN
  bahn_fahrplan_api.get_departure_board_apex(i_api_auth_key   => 'YOUR_API_KEY',
                                             i_language       => 'en', -- default en
                                             i_station_id     => 1111111, -- from location service from before
                                             i_date_time      => NULL, -- valid date with or without time
                                             i_apex_coll_name => 'DEPARTURE_BOARD'); -- default 'DEPARTURE_BOARD'
END;
-- Select from collection
SELECT ac.c001 AS train_name,
       ac.c002 AS train_type,
       ac.c003 AS stop_id,
       ac.c004 AS stop_name,
       ac.c005 AS direction,
       ac.c006 AS stop_date,
       ac.c007 AS stop_time,
       ac.c008 AS track,
       ac.c009 AS journeydetailref
  FROM apex_collections ac
 WHERE ac.collection_name = 'DEPARTURE_BOARD'
```
-- Arrival board
```language-sql
-- create collection
BEGIN
  bahn_fahrplan_api.get_arrival_board_apex(i_api_auth_key   => 'YOUR_API_KEY',
                                           i_language       => 'en', -- default en
                                           i_station_id     => 1111111, -- from location service from before
                                           i_date_time      => NULL, -- valid date with or without time
                                           i_apex_coll_name => 'ARRIVAL_BOARD'); -- default 'ARRIVAL_BOARD'
END;
-- Select from collection
SELECT ac.c001 AS train_name,
       ac.c002 AS train_type,
       ac.c003 AS stop_id,
       ac.c004 AS stop_name,
       ac.c005 AS direction,
       ac.c006 AS stop_date,
       ac.c007 AS stop_time,
       ac.c008 AS track,
       ac.c009 AS journeydetailref
  FROM apex_collections ac
 WHERE ac.collection_name = 'ARRIVAL_BOARD'
```
####Pipelined Function for use in table function
-- Departure board
```language-sql
SELECT departure.train_name,
       departure.train_type,
       departure.stop_id,
       departure.stop_name,
       departure.direction,
       departure.stop_date,
       departure.stop_time,
       departure.track,
       departure.journeydetailref
  FROM TABLE(bahn_fahrplan_api.get_departure_board_pipe(i_api_auth_key => 'YOUR_API_KEY',
                                                        i_language     => 'en', -- default en
                                                        i_station_id   => 1111111, -- from location service from before
                                                        i_date_time    => NULL)) departure
```
-- Arrival board
```language-sql
SELECT arrival.train_name,
       arrival.train_type,
       arrival.stop_id,
       arrival.stop_name,
       arrival.direction,
       arrival.stop_date,
       arrival.stop_time,
       arrival.track,
       arrival.journeydetailref
  FROM TABLE(bahn_fahrplan_api.get_arrival_board_pipe(i_api_auth_key => 'YOUR_API_KEY',
                                                      i_language     => 'en', -- default en
                                                      i_station_id   => 1111111, -- from location service from before
                                                      i_date_time    => NULL)) arrival
```

###Journey detail service                                                      
####Plain JSON response
```language-sql
DECLARE
  l_response_json CLOB;
BEGIN
  l_response_json := bahn_fahrplan_api.get_journey_detail_json(i_journeydetailref_url => 'http://open-api.bahn.de/....'); -- journeydetailref from departure / arrival response
END;
```
####APEX collections
```language-sql
-- create collection
BEGIN
  bahn_fahrplan_api.get_journey_detail_apex(i_journeydetailref_url => 'http://open-api.bahn.de/....', -- journeydetailref from departure / arrival response
                                            i_apex_coll_name       => 'JOURNEY_DETAIL'); -- default 'JOURNEY_DETAIL'
END;
-- Select from collection
SELECT ac.c001 AS station_name,
       ac.c002 AS station_id,
       ac.c003 AS longitude,
       ac.c004 AS latitude,
       ac.c005 AS arr_time,
       ac.c006 AS arr_date,
       ac.c007 AS dep_time,
       ac.c008 AS dep_date,
       ac.c009 AS track
  FROM apex_collections ac
 WHERE ac.collection_name = 'JOURNEY_DETAIL'
```
####Pipelined Function for use in table function
```language-sql
SELECT journey.station_name,
       journey.station_id,
       journey.longitude,
       journey.latitude,
       journey.arr_time,
       journey.arr_date,
       journey.dep_time,
       journey.dep_date,
       journey.track
  FROM TABLE(bahn_fahrplan_api.get_journey_detail_pipe(i_journeydetailref_url => 'http://open-api.bahn.de/....')) journey
```

##License
This software is under **MIT License**.

---