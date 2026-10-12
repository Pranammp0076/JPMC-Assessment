# Position Book Application

A Java 21 / Spring Boot REST API for processing BUY, SELL and CANCEL trade events and retrieving positions by trading account and security. The implementation uses **only in-memory Java collections**; it does not use a database or persist data to the file system.

## Requirements

- JDK 21
- Maven 3.9+ (or a compatible Maven installation)

## Run

```bash
mvn clean test
mvn spring-boot:run
```

The API starts on `http://localhost:8080`. Data is held only in process memory and is lost when the application stops or restarts.

## API

### Process an event

`POST /api/events` — returns `201 Created` with the accepted event.

BUY example:

```json
{
  "id": 1,
  "type": "BUY",
  "tradingAccount": "ACC1",
  "securityId": "SEC1",
  "quantity": 100
}
```

SELL uses the same shape with `"type": "SELL"` and a positive quantity.

CANCEL example:

```json
{
  "id": 1,
  "type": "CANCEL",
  "quantity": 0
}
```

For a cancellation, `id` identifies the original trade. Account, security and quantity in a cancellation request are ignored; the cancellation is associated with the original trade's account/security and has quantity zero in the event history.

### Retrieve one position

`GET /api/positions/{tradingAccount}/{securityId}`

Example: `GET /api/positions/ACC1/SEC1`

Response includes the current net quantity and the event history for that account/security pair.

### Retrieve all positions

`GET /api/positions`

Returns all account/security pairs that have had an accepted event, including a zero position if all their trades have been cancelled.

## Business rules and assumptions

- A position is grouped by the exact pair `(tradingAccount, securityId)`.
- BUY adds quantity; SELL subtracts quantity.
- BUY/SELL quantities must be positive integers. This implementation allows a negative net position because the exercise does not prohibit short positions.
- Trade IDs are unique across original BUY/SELL events. A CANCEL intentionally reuses the original trade ID, as in the assessment example.
- A cancellation must refer to an existing original trade and may be accepted only once. It reverses the original BUY or SELL, regardless of whether the original was a BUY or SELL.
- The original event and cancellation both remain in the position's drilldown history.
- CANCEL requests need only provide `id` and `type`; quantity, security ID and trading account in the cancellation request are ignored because the exercise says those cancellation fields are meaningless.
- An unknown position returns `404`; malformed or invalid requests return `400`; duplicate IDs or repeated cancellations return `409`.
- A `long` quantity is used for integer quantities. An update that would overflow a 64-bit integer is rejected.
- Event processing is synchronized so a request is applied atomically relative to other requests in this single application instance. This is an in-memory single-process exercise, not a distributed system.
- Data is not durable and is lost on restart, as required by the exercise.

## Design

- `api`: HTTP controller and consistent error responses.
- `service`: validation and position business rules.
- `store`: Java maps, sets and lists that hold all state in memory.
- `model`: request and response types.

The current quantity is calculated from accepted event history while ignoring cancelled trades. This makes cancellation a clear reversal without deleting the original event from the audit/drilldown list.




