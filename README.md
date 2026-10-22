# Gisual Challenge

## Services

### Incident Server

The main RESTful API server. All user traffic hits this server, but its job is simple.

Incident users make requests to add incidents and retrieve updates about their current incidents.
The server adds those incidents to the cache, database, and queue to be processed.
Incidents set an initial priority based on incident type. 

Vehicle users make requests to update their current location and be Dispatched to their assigned incidents.

### Incident Queue

Handles ordering and providing of incidents to the async handler. 
Includes a DLQ for incidents that are not picked up within a reasonable time frame multiple times.
When it's calculated that a vehicle will not arrive in the allotted time, 
the incident is put back on the queue to be attempted again by a different vehicle.

### Async Incident Manager

Async Job Handler (i.e., Celery, AirFlow, TemporalIO, etc) which picks up incidents as jobs.
Will calculate and assign a vehicle to the incident based on distance. 
Regularly checking the vehicle's updated location.
If the time to arrive is greater than the allotted time, the job will put the incident back on the queue.

### Routing Service

Separate service for vehicle routing and distance calculation.

## Expected Experience

1. Incident User submits a new incident
2. Request is routed to Incident Server
3. Incident Server calculates priority of incident, saves incident and places on queue
4. Async Incident Manager pulls incident from queue.
5. Async Incident Manager uses Routing Service to calculate the closest vehicle and update the incident record accordingly.
6. Meanwhile, Vehicle User is regularly updating Location, and if not currently dispatched requesting a dispatch.
7. Newly assigned Vehicle now requests route to incident, which routes to the routing service.
8. While Vehicle is on the way to the incident Async Incident Manager is still monitoring the incident, if vehicle is calculated to not arrive on time, the incident will be placed back on the Incident Queue.

### Notes and Considerations

- Data layer uses a cache/DB pair to prevent slowdown from waiting on read/write with the database, while still having a hard backup.
- Dead letter queue exists for worst case scenario that an incident is not properly addressed in time so that an oncall engineer can be notified.

#### Security Consideration: 

- All requests require authentication. 
- Incidents are tied to the user who submitted them, and the vehicles tied to them, data cannot be retrieved unless you are logged in as one of those associated accounts.
- Data provided to vehicles is minimum required for processing incident. Typically only location, incident type, and notes about incident. Health emergencies can include pertinent health information, ie, allergies, major health conditions, etc.

#### Scalability Considerations:

Auto-scaling should be implemented for each major service. Scaling should be based on number of active incidents and vehicles. Notably incident manager needs to scale with vehicles as it can only handle as many incidents as there are active vehicles.
Meanwhile, the incident server and routing service usage can scale with the number of incoming incidents.

Worth considering having separate queues for separate incident categories and/or criticality. For example a fire department vs police department queue.

## APIs

### Incident Creation

**Method:** POST

**Route:** */emergency_service/incident*

**Status Codes:**

- 201: On successful creation of incident
- 400: In case incident time or coordinates are malformed, or incident type is not supported.

**Request Shape:**

```json
{
  "incident_time": "YYYY-MM-DD HH:MM:SS.SS", // In UTC
  "incident_type": "String", // Standardized enum of incident types
  "location": { // LocationShape
    "street_1": "String or NULL",
    "street_2": "String or NULL",
    "city": "String or NULL",
    "state": "String or NULL",
    "zip_code": "String or NULL",
    "coordinates": "String",
    "note": "String or NULL"
  }
}
```

**Response Shape:**
```json
{
  "uuid": "UUID4 related to the created incident",
  "expected_arrival_time": "YYYY-MM-DD HH:MM:SS.SS" // In UTC
}
```

### Vehicle Dispatch

Note: Currently expecting this API to be polled by Vehicle users until they are dispatched.

**Method:** GET

**Route:** */emergency_service/dispatch*

**Status Code:**

- 200: Generally should always return this.
- 401: If Vehicle is not logged in or active.

**Request Shape:** Nothing required/allowed.

**Response Shape:**

```json
{
  "incident_uuid": "UUID4 or NULL", // uuid related to assigned incident or null if nothing currently assigned
  "destination_location": {"LocationShape or NULL"}
}
```
