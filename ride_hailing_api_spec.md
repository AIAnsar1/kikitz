## API Specification for Ride-Hailing Platform

### Common Patterns

* Base path: `/api/v1`
* Auth: OAuth2 JWT Bearer Token in `Authorization: Bearer <token>` header
* Responses: JSON with `{ "success": bool, "data": ..., "error": ... }`

---

### 1. Auth & User

| Method | Endpoint         | Request Body                                                                                | Response Body                                                    | Notes                             |
| ------ | ---------------- | ------------------------------------------------------------------------------------------- | ---------------------------------------------------------------- | --------------------------------- |
| POST   | `/auth/register` | `{ "phone": "+9989XXXXXXX", "sms_code": "123456" }`                                         | `{ "token": "...", "user": { id, phone, role } }`                | SMS OTP verification              |
| POST   | `/auth/login`    | `{ "phone": "+9989XXXXXXX", "password": "..." }` or `{ "phone": "...", "sms_code": "..." }` | `{ "token": "...", "user": { id, phone, role } }`                | Password or OTP login             |
| POST   | `/auth/logout`   | *(none)*                                                                                    | `{ "success": true }`                                            | Invalidates token                 |
| GET    | `/users/me`      | *(Auth)*                                                                                    | `{ "id", "phone", "role", "profile": {...}, "settings": {...} }` | Current user info                 |
| PUT    | `/users/me`      | `{ "full_name": "...", "avatar_url": "...", "language": "en", ... }`                        | Updated `user` object                                            | Update profile and settings       |
| POST   | `/users/docs`    | `multipart/form-data`: `passport_file`, `license_file`, `vehicle_passport_file`             | `{ "documents": [{type, url, status}, ...] }`                    | Upload documents for verification |

---

### 2. Favorites & Addresses

| Method | Endpoint                    | Request Body                                                    | Response Body                                         | Notes                     |
| ------ | --------------------------- | --------------------------------------------------------------- | ----------------------------------------------------- | ------------------------- |
| GET    | `/addresses/favorites`      | *(Auth)*                                                        | `[{ id, label, address, location: {lat, lon} }, ...]` | List saved addresses      |
| POST   | `/addresses/favorites`      | `{ "label": "Home", "address": "XYZ", "lat": ..., "lon": ... }` | `{ id, label, address, location: {lat, lon} }`        | Add new favorite (max 10) |
| DELETE | `/addresses/favorites/{id}` | *(Auth)*                                                        | `{ "success": true }`                                 | Remove favorite           |

---

### 3. Tariffs & Options

| Method | Endpoint           | Request Body | Response Body                                         | Notes                   |
| ------ | ------------------ | ------------ | ----------------------------------------------------- | ----------------------- |
| GET    | `/tariffs`         | *(Auth)*     | `[{ id, code, name, base_price, per_km_price, ... }]` | List all active tariffs |
| GET    | `/tariffs/options` | *(Auth)*     | `[{ id, code, name, price, is_free, ... }]`           | List ride options       |

---

### 4. Orders

| Method | Endpoint                      | Request Body                                                                                                                                                                         | Response Body                                                                                      | Notes                                             |
| ------ | ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------------------- | ------------------------------------------------- |
| POST   | `/orders`                     | `{ "pickup": { "address": "...", "lat": ..., "lon": ... }, "dropoff": {...}, "tariff_id": "...", "options": ["opt1","opt2"], "special_tariff_id": "...", "payment_method": "card" }` | `{ "order_id": "...", "status": "pending", "estimated_price": 123.45 }`                            | Create new order, calculate estimate              |
| GET    | `/orders/available`           | *(Auth driver)*                                                                                                                                                                      | `[{order_id, pickup, dropoff, distance_km, estimated_price}, ...]`                                 | Get nearby orders for driver (based on redis loc) |
| POST   | `/orders/{order_id}/accept`   | *(Auth driver)*                                                                                                                                                                      | `{ "success": true, "status": "accepted" }`                                                        | Driver accepts                                    |
| POST   | `/orders/{order_id}/arrive`   | *(Auth driver)*                                                                                                                                                                      | `{ "status": "arrived" }`                                                                          | Driver marks arrival                              |
| POST   | `/orders/{order_id}/start`    | *(Auth driver)*                                                                                                                                                                      | `{ "status": "started", "started_at": "..." }`                                                     | Driver starts trip                                |
| POST   | `/orders/{order_id}/wait`     | `{ "minutes": 3 }`                                                                                                                                                                   | `{ "status": "waiting", "wait_fee": 15.00 }`                                                       | Charge waiting fee                                |
| POST   | `/orders/{order_id}/complete` | *(Auth driver)*                                                                                                                                                                      | `{ "status": "completed", "final_price": 130.00 }`                                                 | Complete trip, trigger payment                    |
| POST   | `/orders/{order_id}/cancel`   | `{ "reason": "..." }`                                                                                                                                                                | `{ "status": "cancelled", "cancel_count": 1 }`                                                     | Cancel by driver or passenger                     |
| GET    | `/orders/{order_id}`          | *(Auth)*                                                                                                                                                                             | Detailed order info: `id, passenger, driver, status, timestamps, price, options, tracking_channel` | Order details                                     |
| GET    | `/orders/history`             | *(Auth)*                                                                                                                                                                             | `[{order...}, ...]`                                                                                | Past orders                                       |

---

### 5. Location & Tracking

| Method | Endpoint                     | Request Body                 | Response Body                                      | Notes                                            |
| ------ | ---------------------------- | ---------------------------- | -------------------------------------------------- | ------------------------------------------------ |
| POST   | `/location/update`           | `{ "lat": ..., "lon": ... }` | `{ "success": true }`                              | Driver updates current location (Redis + log DB) |
| GET    | `/location/track/{order_id}` | *(Auth passenger)*           | WebSocket / SSE stream of `{ "lat", "lon", "ts" }` | Real-time driver location                        |

---

### 6. Chat & Feedback

| Method | Endpoint                   | Request Body                                           | Response Body                                       | Notes                  |
| ------ | -------------------------- | ------------------------------------------------------ | --------------------------------------------------- | ---------------------- |
| POST   | `/chat/{order_id}/send`    | `{ "message": "...", "attachment_url": "..." }`        | `{ "message_id": "...", "sent_at": "..." }`         | Send chat message      |
| GET    | `/chat/{order_id}/history` | *(Auth)*                                               | `[{sender, message, sent_at, attachment_url}, ...]` | Chat history for order |
| POST   | `/feedback`                | `{ "order_id": "...", "rating": 5, "comment": "..." }` | `{ "success": true }`                               | Leave rating & comment |
| GET    | `/feedback/{user_id}`      | *(Auth)*                                               | `[{from_user, rating, comment, created_at}, ...]`   | Feedback for user      |

---

### 7. Notifications & SOS

| Method | Endpoint        | Request Body                                                     | Response Body                                     | Notes                        |
| ------ | --------------- | ---------------------------------------------------------------- | ------------------------------------------------- | ---------------------------- |
| POST   | `/notify/send`  | `{ "user_id": "...", "type": "order_update", "payload": {...} }` | `{ "success": true }`                             | Send custom notification     |
| GET    | `/notify/list`  | *(Auth)*                                                         | `[{type, payload, status, created_at}, ...]`      | User's notifications         |
| POST   | `/sos`          | `{ "order_id": "...", "lat": ..., "lon": ... }`                  | `{ "sos_id": "...", "status": "pending" }`        | Trigger SOS event            |
| GET    | `/sos/{sos_id}` | *(Auth operator)*                                                | `{ "user_id", "location", "action_status", ... }` | SOS event status and history |

---

### 8. Payments & Wallets

| Method | Endpoint                     | Request Body                                    | Response Body                                  | Notes                 |
| ------ | ---------------------------- | ----------------------------------------------- | ---------------------------------------------- | --------------------- |
| GET    | `/wallets`                   | *(Auth)*                                        | `[{id, type, balance}, ...]`                   | List user wallets     |
| POST   | `/wallets/{wallet_id}/topup` | `{ "amount": 10000, "payment_method": "card" }` | `{ "transaction_id": "..." }`                  | Top-up wallet         |
| POST   | `/orders/{order_id}/payment` | *(triggered internally after completion)*       | `{ "payment_id": "...", "status": "pending" }` | Create payment record |
| GET    | `/payments/history`          | *(Auth)*                                        | `[{payment...}, ...]`                          | Transaction history   |

---

### 9. Promo & Referrals

| Method | Endpoint            | Request Body                      | Response Body                                               | Notes                        |
| ------ | ------------------- | --------------------------------- | ----------------------------------------------------------- | ---------------------------- |
| POST   | `/promo/apply`      | `{ "code": "PROMO123" }`          | `{ "discount": 10, "expires_at": "..." }`                   | Apply promo to current order |
| GET    | `/promo/codes`      | *(Auth)*                          | `[{code, discount_type, discount_value, valid_until}, ...]` | List available promos        |
| POST   | `/referrals/invite` | `{ "invitee_phone": "+9989..." }` | `{ "referral_id": "...", "status": "sent" }`                | Send referral link           |
| GET    | `/referrals`        | *(Auth)*                          | `[{invitee_id, invited_at, status}, ...]`                   | User's referral history      |

---

### 10. Admin & CRM

| Method | Endpoint                       | Request Body                                | Response Body                                  | Notes                        |
| ------ | ------------------------------ | ------------------------------------------- | ---------------------------------------------- | ---------------------------- |
| GET    | `/admin/users`                 | *(Auth admin)*                              | `[{id, phone, role, status, created_at}, ...]` | List all users               |
| GET    | `/admin/orders`                | `(optional filters: status, date)`          | `[{order...}, ...]`                            | View all orders              |
| GET    | `/admin/payments`              | `(optional filters: status, date)`          | `[{payment...}, ...]`                          | View all payments            |
| GET    | `/admin/complaints`            |                                             | `[{complaint...}, ...]`                        | List all complaints          |
| POST   | `/admin/users/{user_id}/block` | `{ "reason": "..." }`                       | `{ "success": true }`                          | Block user                   |
| GET    | `/admin/stats`                 |                                             | `{ "total_users": ..., "rides": ..., ... }`    | Platform statistics          |
| POST   | `/admin/sos/{sos_id}/dispatch` | `{ "operator_id": "...", "action": "..." }` | `{ "success": true }`                          | Operator action on SOS event |

---

*Документация автоматически генерируется из спецификации кодов и моделей.*
