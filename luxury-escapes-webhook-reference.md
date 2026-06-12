# Luxury Escapes Payloads Reference

> Environment: `trial-dev.tourchain.net` (staging).

## Summary

| Action | Method   | Endpoint                                                | Body                         | Response |
| ------ | -------- | ------------------------------------------------------- | ---------------------------- | -------- |
| Create | `POST`   | `/b2badminapi/api/IcsTransfer/webhook/booking-complete` | full payload                 | `201`    |
| Update | `POST`   | `/b2badminapi/api/IcsTransfer/webhook/booking-complete` | full payload (same `number`) | `201`    |
| Cancel | `DELETE` | `/b2badminapi/api/IcsTransfer/webhook/cancel/{number}`  | none                         | `201`    |

**Create and Update share one endpoint and one schema.**
They are indistinguishable at the payload level. Differentiation is by `number`: first arrival = create, subsequent arrivals with same `number` = update.
This is the auto-detect contract the consumer must implement.

`number` is the idempotency / correlation key (example: `ED4C07`).

## Field reference — `booking-complete`

| Field                                      | Type   | Notes                         |
| ------------------------------------------ | ------ | ----------------------------- |
| `number`                                   | string | Booking ID. Correlation key.  |
| `stopFollowUpButton`                       | bool   | Partner UI flag. Passthrough. |
| `customer_email`                           | string | `@luxuryescapes.com` domain.  |
| `customer_given_name` / `customer_surname` | string | PII.                          |
| `customer_phone`                           | string | PII.                          |
| `transfer_information`                     | object | See below.                    |
| `accommodation_items`                      | array  | See below.                    |

### `transfer_information`

| Field                                         | Type         | Notes                                                                                                                                                                        |
| --------------------------------------------- | ------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `type`                                        | string       | e.g. `arrival`.                                                                                                                                                              |
| `checkInDate` / `checkOutDate`                | ISO datetime | No timezone suffix.                                                                                                                                                          |
| `fullname`                                    | string       | PII.                                                                                                                                                                         |
| `countryCodePhone`                            | string       | e.g. `61`.                                                                                                                                                                   |
| `phone`                                       | string       | PII.                                                                                                                                                                         |
| `arrival.date`                                | ISO datetime |                                                                                                                                                                              |
| `arrival.time`                                | string       | `"N/A"` on create, real value (`"03:15"`) on update.                                                                                                                         |
| `arrival.flightNumber`                        | string       | `"N/A"` until known (`"QFF666"`).                                                                                                                                            |
| `arrival.adults` / `arrival.children`         | int          |                                                                                                                                                                              |
| `arrival.channelFareType`                     | string       | e.g. `FARETYPE_0006`.                                                                                                                                                        |
| `luggage` / `oversizeLuggage` / `babyCarSeat` | int          | `0` on create, updated later.                                                                                                                                                |
| `notes`                                       | string       | **Free-text passthrough. Unsanitized** — staging sample contained a human comment (`"Hey Yoga - can you see the comment here?"`). Sanitize/escape before display or storage. |
| `pickUpDescription` / `dropOffDescription`    | string       | Location labels.                                                                                                                                                             |

### `accommodation_items[]`

| Field                                                         | Type             | Notes                                       |
| ------------------------------------------------------------- | ---------------- | ------------------------------------------- |
| `id`                                                          | uuid             | Item ID. Changes between create and update. |
| `reservation.check_in` / `check_out`                          | ISO datetime     |                                             |
| `reservation.hotel_info.id`                                   | string           | Often `"N/A"`.                              |
| `reservation.hotel_info.name`                                 | string           | PII-adjacent / redacted.                    |
| `reservation.hotel_info.geo_data.administrative_area_level_1` | string           | e.g. `Denpasar`.                            |
| `reservation.hotel_info.geo_data.country`                     | string           |                                             |
| `reservation.hotel_info.geo_data.place_id`                    | string           | Google place ID, often `"N/A"`.             |
| `created_at`                                                  | ISO datetime (Z) | Item creation time.                         |

## Behavioral notes

- Create payload has empty/`N/A`/`0` defaults; update payload fills real flight, luggage, time values. Treat every field as nullable-equivalent.
- Datetime formats are inconsistent: `transfer_information` uses no suffix, `created_at` uses `Z`. Parse defensively.
- Cancel carries no body. The `{number}` in the path is the only key. Confirm the resource exists before acting.

## Example payloads

### Create (`POST /webhook/booking-complete`)

```json
{
  "number": "ED4C07",
  "stopFollowUpButton": true,
  "customer_email": "bf8dcb76_@_luxuryescapes.com",
  "customer_given_name": "[REDACTED]",
  "customer_phone": "[REDACTED]",
  "customer_surname": "[REDACTED]",
  "transfer_information": {
    "type": "arrival",
    "checkInDate": "2026-07-08T00:00:00",
    "checkOutDate": "2026-07-08T00:00:00",
    "fullname": "[REDACTED]",
    "countryCodePhone": "61",
    "phone": "[REDACTED]",
    "arrival": {
      "date": "2026-07-08T00:00:00",
      "time": "N/A",
      "flightNumber": "N/A",
      "adults": 2,
      "children": 0,
      "channelFareType": "FARETYPE_0006"
    },
    "luggage": 0,
    "oversizeLuggage": 0,
    "babyCarSeat": 0,
    "notes": "Arrival: ",
    "pickUpDescription": "Bali Denpasar",
    "dropOffDescription": "Mutiara Bali Boutique Resort Villas & Spa"
  },
  "accommodation_items": [
    {
      "id": "2e96e3ed-93a1-48cc-a739-82797bf77b12",
      "reservation": {
        "check_in": "2026-07-08T00:00:00",
        "check_out": "2026-07-08T00:00:00",
        "hotel_info": {
          "id": "N/A",
          "name": "[REDACTED]",
          "geo_data": {
            "administrative_area_level_1": "Denpasar",
            "country": "[REDACTED]",
            "place_id": "N/A"
          }
        }
      },
      "created_at": "2026-06-09T03:34:20.181Z"
    }
  ]
}
```

### Update (`POST /webhook/booking-complete`, same `number`)

```json
{
  "number": "ED4C07",
  "stopFollowUpButton": true,
  "customer_email": "bf8dcb76_@_luxuryescapes.com",
  "customer_given_name": "[REDACTED]",
  "customer_phone": "[REDACTED]",
  "customer_surname": "[REDACTED]",
  "transfer_information": {
    "type": "arrival",
    "checkInDate": "2026-07-08T03:15:00",
    "checkOutDate": "2026-07-08T03:15:00",
    "fullname": "[REDACTED]",
    "countryCodePhone": "61",
    "phone": "[REDACTED]",
    "arrival": {
      "date": "2026-07-08T03:15:00",
      "time": "03:15",
      "flightNumber": "QFF666",
      "adults": 2,
      "children": 0,
      "channelFareType": "FARETYPE_0006"
    },
    "luggage": 2,
    "oversizeLuggage": 1,
    "babyCarSeat": 0,
    "notes": "Arrival: [free-text passthrough]",
    "pickUpDescription": "Bali Denpasar",
    "dropOffDescription": "Mutiara Bali Boutique Resort Villas & Spa"
  },
  "accommodation_items": [
    {
      "id": "4563be33-60e8-46a7-a3a1-0d7eae5d3b96",
      "reservation": {
        "check_in": "2026-07-08T03:15:00",
        "check_out": "2026-07-08T03:15:00",
        "hotel_info": {
          "id": "N/A",
          "name": "[REDACTED]",
          "geo_data": {
            "administrative_area_level_1": "Denpasar",
            "country": "[REDACTED]",
            "place_id": "N/A"
          }
        }
      },
      "created_at": "2026-06-09T03:03:05.529Z"
    }
  ]
}
```

### Cancel (`DELETE /webhook/cancel/ED4C07`)

No request body. Responds `201`.
