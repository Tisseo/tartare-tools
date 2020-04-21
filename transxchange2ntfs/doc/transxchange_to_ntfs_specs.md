# TransXChange reading specification
## Introduction
This document describes how a [TransXChange](http://naptan.dft.gov.uk/transxchange/schema/2.4/doc/TransXChangeSchemaGuide-2.4-v-52.pdf) feed is read in Navitia Transit model (NTM)
and transformed into a [NTFS feed](https://github.com/CanalTP/ntfs-specification/blob/master/ntfs_fr.md).

For the sake of simplicity, the NTM properties that are not specified in the source
data are not described below.

In order to guarantee that the NTFS objects identifiers are unique and stable, each
object id is prefixed with a unique prefix (specified for each datasource), following
the general pattern `<prefix>:<id>`.

## Input data description
Each file of a TransXChange dataset represents a transit line for a *specific
operating period*. Several files of the same archive might need to be read in order
to consolidate all the trips associated to a transit NTFS line.

An additional data source is necessary in order to retrieve the information relative to the
stops used in the TransXChange feed. The National Public Transport Access Nodes ([NaPTAN](http://naptan.app.dft.gov.uk/DataRequest/Naptan.ashx?format=csv)) database
is a UK nationwide system for uniquely identifying all the points of access to public transport in the UK.
As `EndDate` validity of a Service may not be provided or specifying a date in a very far future (more than 50 years), this convertor requires a specified end date to compute calendars.

This version of the connector handles TranXChange files with a single *Service*
specifying one or more *Lines* (if a file specifies more than one *Service*, the reading of the transxchange file abort).  
Multiple *Service*s will be possibly considered in a
later version. Also, on-demand-transport services as well as frequency-based trips
are not handled; therefore, any input feed that contains a *Service/FlexibleService*
or a *VehicleJourney/Frequency* will be ignored.

## Mapping between TransXChange elements and NTFS objects
To ease the deterministic reading of a TransXCange archive, the files are read in alphabetical order.

### networks.txt

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
network_id | *Operators/Operator/OperatorCode* | This field is prefixed. If more than one operators are specified, the operator referenced by *Services/Service/RegisteredOperatorRef* is used to create the network.
network_name | *Operators/Operator/TradingName* | If the element is not present, the *Operators/Operator/OperatorShortName* is used instead.
network_url | *Operators/Operator/WebSite* |
network_timezone | | Fixed value `Europe/London`.
network_phone | *Operators/Operator/ContactTelephoneNumber* |

### companies.txt

TransXChange includes a basic representation of an *Operator* without explicitly making the distinction between the transport network and the operator company, as it is the case in NTFS. If a *VehicleJourney* uses a reference of an *OperatorRef* different from the *Services/Service/RegisteredOperatorRef*, this *Operator* is used to create the company.

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
company_id | *Operators/Operator/OperatorCode* | This field is prefixed.
company_name | *Operators/Operator/OperatorShortName* |

### stops.txt
For each *AnnotatedStopPointRef* identified in the TransXChange feed, the coordinates
of the stop_point are retrieved from the **Stops.csv** file of the NaPTAN dataset.

The stop_areas are referenced in the **StopsInArea.csv** file and then detailed in
the **StopAreas.csv** file of the NaPTAN dataset.

#### For stop_points

NTFS field | TransXChange element | NaPTAN file | NaPTAN field | Mapping rule/Comment
--- | --- | --- | --- | ---
stop_id | *StopPoints/AnnotatedStopPointRef/StopPointRef* | Stops.csv | *ATCOCode* | This field is prefixed.
stop_name | | Stops.csv | *CommonName* | The stop name is also available in TransXChange *StopPoints/AnnotatedStopPointRef/CommonName*, but the NaPTAN value is considered to be the reference.
location_type | | Stops.csv | | Fixed value `0`.
stop_lat | | Stops.csv | *Latitude* |
stop_lon | | Stops.csv | *Longitude* |
parent_station | | StopsInArea.csv | *StopAreaCode* | This field is prefixed. The field *AtcoCode* is used as a matching key with the *ATCOCode* of the associated stop_point.
platform_code | | Stops.csv | *Indicator* |

In addition, if the *NaptanCode* field is defined for a stop_point in the NaPTAN dataset, this code is added as a complementary code for the associated stop_point with the value `NaptanCode` as the name of the identification system.

#### For stop_areas

NTFS field | NaPTAN file | NaPTAN field | Mapping rule/Comment
--- | --- | --- | ---
stop_id | StopsInArea.csv | *StopAreaCode* | This field is prefixed.
stop_name | StopAreas.csv | *Name* |
stop_lat | StopAreas.csv | *Northing* | The input coordinate system Easting/Northing (EPSG:27700) should be converted to WGS84 (EPSG:4326). In case the coordinates of the stop_area are not specified, they are computed as the barycenter of all the associated stop_points.
stop_lon | StopAreas.csv | *Easting* | The input coordinate system Easting/Northing (EPSG:27700) should be converted to WGS84 (EPSG:4326). In case the coordinates of the stop_area are not specified, they are computed as the barycenter of all the associated stop_points.
location_type | | | Fixed value `1`.

### lines.txt
A *Services/Service* in the input feed might specify one or more *Lines*.

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
line_id | *Services/Service/ServiceCode*, *Services/Service/Line{id}* | This field is prefixed and formed by the concatenation of the two fields separated by a `:`. Ex. "\<prefix>:1_58_BC:SL1".
line_code | *Services/Service/Lines/Line/LineName* |
line_name | *Services/Service/Description* | 
forward_line_name | *Services/Service/StandardService/Destination* |
backward_line_name | *Services/Service/StandardService/Origin* |
network_id | *Services/Service/RegisteredOperatorRef* | The referenced *Operators/Operator/OperatorCode* value is taken into account. This field is prefixed. Link to the file [networks.txt](#networkstxt).
commercial_mode_id | *Services/Service/Mode* | This field is not prefixed. Link to the file commercial_modes.txt of the NTFS. See the mapping rule below.

#### Defining modes
*Services/Service/Mode* in TransXChange | physical_mode_id in NTFS | physical_mode_name in NTFS | commercial_mode_id in NTFS | commercial_mode_name in NTFS
--- | --- | --- | --- | ---
air | Air | Air | Air | Air
bus | Bus | Bus | Bus | Bus
coach | Coach | Coach | Coach | Coach
ferry | Ferry | Ferry | Ferry | Ferry
metro/underground | Metro | Metro | Metro | Metro
rail | Train | Train | Train | Train
tram | Tramway | Tramway| Tramway| Tramway
trolleyBus | Shuttle | Shuttle | Shuttle | Shuttle

If *Services/Service/Mode* is not specified or the value is unknown (a different value than those listed above), the default mode `Bus` is used.

All `physical_mode` are enhanced with CO2 emission and fallback modes, following
the documentation in [common.md](common.md#co2-emissions-and-fallback-modes).

### routes.txt
A Route is created from a line and a direction of journey patterns (*StandardService/JourneyPattern/Direction*).

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
route_id | *Services/Service/ServiceCode*, *Services/Service/Line{id}* *StandardService/JourneyPattern/Direction* | This field is prefixed and formed by the concatenation of *Services/Service/ServiceCode*, *Services/Service/Line{id}* and *StandardService/JourneyPattern/Direction* separated by a `:`. Ex. "\<prefix>:1_58_BC:SL1:inbound".
route_name |  | See [`common.md`](common.md#general-rules) to generate the `name`
direction_type | *StandardService/JourneyPattern/Direction* | The value is set to `inbound` or `clockwise` when the specified value for the direction is `inboundAndOutbound` or `circular`, respectively.
line_id | *Services/Service/ServiceCode*, *Services/Service/Line{id}* | This field is prefixed. Link to the file [lines.txt](#linestxt).
destination_id |  | See [`common.md`](common.md#general-rules) to generate the `destination_id`

### calendar.txt and calendar_dates.txt
The validity period of a service is stated in *Services/Service/OperatingPeriod*. In case the validity period is open ended (the *EndDate* is not specified), the end date is defined by a mandatory parameter of this converter.

Service days are calculated from the *VehicleJourneys/VehicleJourney/OperatingProfile* (if not specified, the operation days are inherited from *Service/OperatingProfile*). The corresponding days of the week are activated according to the pattern given by *RegularDayType/DaysOfWeek*. In particular, it is allowed any meaningful combination of the following possible values: `Monday`, `Tuesday`, `Wednesday`, `Thursday`, `Friday`, `Saturday`, `Sunday`, `MondayToFriday`, `MondayToSaturday`, `MondayToSunday`, `NotSaturday`, `Weekend`. If no particular day of the week is explicitly specified, all days of the week (Monday to Sunday) are considered by default.

The element *BankHolidayOperation* may be also be present, specifying how the service operates on a bank holiday. The possible values are the following: `AllBankHolidays`, `AllHolidaysExceptChristmas`, `ChristmasDay`, `Christmas`, `BoxingDay`, `NewYearsDay`, `Jan2ndScotland`, `GoodFriday`, `EasterMonday`, `MayDay`, `SpringBank`, `AugustBankHolidayScotland`, `LateSummerBankHolidayNotScotland`, `StAndrewsDay`.

Bank holidays for England and Wales are taken into account as defined [here](https://www.gov.uk/bank-holidays/england-and-wales.json).

### trips.txt
A trip is created for each *VehicleJourneys/VehicleJourney*. The referenced
*JourneyPattern* is used to link the trip to the corresponding NTFS route via the
*Services/Service/StandardService/JourneyPattern/Direction*.
The referenced *JourneyPatternSections* are then used to retrieve the sequence of
stops and scheduled stop times of the trip.

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
route_id | *Services/Service/ServiceCode*, *Services/Service/Line{id}*, *StandardService/JourneyPattern/Direction* | This field is prefixed and formed by the concatenation of the three elements separated by a `:`. Link to the file [routes.txt](#routestxt).
service_id | *VehicleJourney/ServiceRef*, *VehicleJourney/LineRef*, *VehicleJourney/VehicleJourneyCode* | This field is prefixed and formed by the concatenation of the three elements and an index/integer (in fourth element), separated by a `:`, and a prefix `CD`. Ex: "<prefix>:CD:1_58_BC:SL1:VJ1:1". If the validity pattern (set of dates identified by a `service_id` in [`calendar.txt`](#calendartxt) and [`calendar_dates.txt`](#calendar_datestxt)) already exists, then the existing one is used instead of creating a new validity pattern (with a new `service_id`). See above for the details about the services dates attached to a trip.
trip_id | *VehicleJourney/ServiceRef*, *VehicleJourney/LineRef*, *VehicleJourney/VehicleJourneyCode* | This field is prefixed and formed by the concatenation (separated by a `:`) of the three elements below, and an index/integer (in fourth element) in order to guarantee uniqueness.
trip_headsign | *Services/Service/StandardService/JourneyPattern/DestinationDisplay* | 
company_id | *VehicleJourney/OperatorRef* | This field is prefixed. Link to the file [companies.txt](#companiestxt). The referenced *Operators/Operator/OperatorCode* is used. If no *OperatorRef* is specified for the trip, the associated *Services/Service/RegisteredOperatorRef* is used to retrieve the company for the trip.
physical_mode_id | *Services/Service/Mode* | This field is not prefixed. Link to the file physical_modes.txt of the NTFS. See above for the mapping of transport modes.

### stop_times.txt
The passing times at each stoppoint of a trip are specified as an ordered list of
links between the stoppoints (*JourneyPatternTimingLink*s) in the *JourneyPatternSection* attached to the associated *JourneyPattern*.

In some cases, a *VehicleJourney* might specify explicitly some timing links
that are different from the underlying *JourneyPattern*. In this case, a
*VehicleJourneyTimingLink* overrides any common property it shares with a *JourneyPatternTimingLink*.

#### Computing passing times at each stoppoint
The arrival/departure time for the first stoppoint of a trip is explicitly specified
in *VehicleJourney/DepartureTime*. For each subsequent stoppoint, the passing time is
calculated from the cumulative sum of the current *JourneyPatternTimingLink* values
for all preceding stops in the journey link sequence as follows:
- arrival_time at stoppoint<sub>n</sub> = [departure_time from stop<sub>n-1</sub>] + [*RunTime* for inbound link from stop<sub>n-1</sub>]
- departure_time at stop<sub>n</sub> = [arrival_time at stop<sub>n</sub>] + [*WaitTime* for destination end of inbound link from stop<sub>n-1</sub>] + [*WaitTime* for origin of outbound link to stop<sub>n+1</sub>]

If *WaitTime* is not specified, the default waiting time at a stoppoint is considered `0`.

Note that *RunTime* and *WaitTime* are given as [durations](https://en.wikipedia.org/wiki/ISO_8601#Durations).

Except fot the last stop_point of the trip, the departure (*JourneyPatternTimingLink/From*)
of each *JourneyPatternTimingLink* specifies the stoppoint id, its sequence as well as the pickup/dropoff method. The last stop_point id specified by the last arrival (*JourneyPatternTimingLink/To*).

NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
trip_id | *VehicleJourney/ServiceRef*, *VehicleJourney/LineRef*, *VehicleJourney/VehicleJourneyCode* | This field is prefixed. Link to the file [trips.txt](#tripstxt).
arrival_time | *JourneyPatternTimingLink/RunTime* | See computing rule above.
departure_time | *JourneyPatternTimingLink/RunTime*, *JourneyPatternTimingLink/From/WaitTime* | See computing rule above.
stop_id | *JourneyPatternTimingLink/From/StopPointRef* | This field is prefixed. Link to the file [stops.txt](#stopstxt).
stop_sequence | *JourneyPatternTimingLink/From{SequenceNumber}* | The value should be `1` for the first stoppoint of the trip.
pickup_type | *JourneyPatternTimingLink/From/Activity* | `1` when the input value is `setDown`, `0` otherwise. Note that if the attribute *Activity* is explicitly specified at the level of *VehicleJourneyTimingLink* and its value is `setDown`, then it overrides the property specified at the *JourneyPattern* level.
drop_off_type | *JourneyPatternTimingLink/From/Activity* | `1` when the input value is `pickUp`, `0` otherwise. Note that if the attribute *Activity* is explicitly specified at the level of *VehicleJourneyTimingLink* and its value is `pickUp`, then it overrides the property specified at the *JourneyPattern* level.

## Possible evolutions
The following points might be possibly included in a later version of the connector:

### Provide comments for trips
Two additional files for comments are to be created:

**comments.txt**
NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
comment_id | *VehicleJourney/VehicleJourneyCode*, *VehicleJourney/Note/NoteCode* | This field is prefixed and formed by the concatenation of the two fields separated by a `:`.
comment_name | *VehicleJourney/Note/NoteText* |

**comment_links.txt**
NTFS field | TransXChange element | Mapping rule/Comment
--- | --- | ---
object_id | *VehicleJourney/ServiceRef*, *VehicleJourney/LineRef*, *VehicleJourney/VehicleJourneyCode* | This field is prefixed. Link to the file [trips.txt](#tripstxt).
object_type |  | Fixed value `trip`.
comment_id | *VehicleJourney/VehicleJourneyCode*, *VehicleJourney/Note/NoteCode* | This field is prefixed. Link to the file [comments.txt](#commentstxt).

### Provide accessibility info for trips
The wheelchair_accessible property is created in the trip_properties.txt file after the element *VehicleJourney/Operational/VehicleType/WheelchairAccessible*. The value of wheelchair_accessible is `1` when the trip is accesible, `2` when the trip is not accessible and `0` when the field is not specified.

### Take into account exceptional service dates
The element *SpecialDaysOfOperation* may also be present within an *OperatingProfile* specifying a *DateRange* with the specific dates of (non) operation. The days on which the service does (*DaysOfOperation*) or does not (*DaysOfNonOperation*) run are specified separately. Note that special days of operation are additional to the regular operating period (inclusion); inversely, special days of non operation further restrict the regular operating period (exclusion). Note also that special days override any Bank holiday day types.

However, the current version handles only regular days and bank holidays that are explicitily specified as normal elements of an *OperatingProfile*. Special or periodic service dates (e.g. *SpecialDaysOfOperation*, *PeriodicDayType*, *ServicedOrganisationDayType*) are not yet handled, they might be added in a later version.

### Handle shared attributes of *JourneyPattern* and *VehicleJourney*
The current version does not take into account the common properties of a *VehicleJourney* shared with the associated *JourneyPattern*. In particular, waiting times at a timing link that are explicitly specified in the scope of a *VehicleJourneyTimingLink* are not yet handled and might be added in a later version.

Similarly, the case where the *Activity* attribute at the *VehicleJourney* level has the value `pass` is not yet handled; for the moment, the activity specified in the *JourneyPattern* level is taken into account.

### Handle implicit references of a *JourneyPattern* in a *VehicleJourney*
The current version only handles vehicle journeys that explicitly reference a single journey pattern and its timing links. In a later version, the following advanced cases might be considered:

- a vehicle journey that references all the links of another vehicle journey (which, in turn, references a journey pattern)
- a vehicle journey that references more than one journey patterns.
