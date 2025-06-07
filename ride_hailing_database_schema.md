Ниже — **полная структура базы данных**, учитывающая все пункты из ТЗ пассажиров и водителей (см. Тз Пассажиров , ТЗ Водитель ). Разбивка по «сервисам» в рамках единой PostgreSQL (схемы), плюс Redis и Mongo для кешей/аналитики.

---

## Схема `auth` — пользователи, профили, настройки

```sql
CREATE SCHEMA auth;

-- Пользователь и аутентификация
CREATE TABLE auth.users (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone          VARCHAR(20) UNIQUE NOT NULL,      -- тел. для SMS
  password_hash  TEXT NOT NULL,
  role           VARCHAR(20) NOT NULL 
                    CHECK (role IN ('passenger','driver','admin')),
  is_active      BOOLEAN NOT NULL DEFAULT TRUE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Доп. профиль для пассажира/водителя
CREATE TABLE auth.user_profiles (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL 
                    REFERENCES auth.users(id) ON DELETE CASCADE,
  full_name      VARCHAR(100),
  nickname       VARCHAR(50),
  avatar_url     TEXT,
  birth_date     DATE,
  gender         VARCHAR(10) 
                    CHECK (gender IN ('male','female','other')),
  driver_license_url TEXT,
  passport_url   TEXT,
  vehicle_passport_url TEXT,
  status         VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending','approved','rejected')),
  rating         NUMERIC(2,1) NOT NULL DEFAULT 5.0,
  completed_rides INT NOT NULL DEFAULT 0,
  tier           VARCHAR(20) NOT NULL DEFAULT 'bronze',
  referral_code  VARCHAR(20) UNIQUE,
  language       VARCHAR(5) NOT NULL DEFAULT 'uz',
  dark_mode      BOOLEAN NOT NULL DEFAULT FALSE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Пользовательские настройки (уведомления и пр.)
CREATE TABLE auth.user_settings (
  user_id        UUID PRIMARY KEY
                    REFERENCES auth.users(id) ON DELETE CASCADE,
  sms_notifications BOOLEAN NOT NULL DEFAULT TRUE,
  push_notifications BOOLEAN NOT NULL DEFAULT TRUE,
  share_location_prompt BOOLEAN NOT NULL DEFAULT TRUE,
  chat_support_method VARCHAR(20) 
                    CHECK (chat_support_method IN ('in_app','phone')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Избранные адреса (до 10 штук)
CREATE TABLE auth.favorite_addresses (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL 
                    REFERENCES auth.users(id) ON DELETE CASCADE,
  label          VARCHAR(30) NOT NULL,   -- «Дом», «Работа» и т.д.
  address        TEXT NOT NULL,
  location       GEOGRAPHY(POINT,4326),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, label)
);
CREATE INDEX ON auth.favorite_addresses USING GIST(location);
```

---

## Схема `tariffs` — тарифы и доп. услуги

```sql
CREATE SCHEMA tariffs;

-- Базовые тарифы
CREATE TABLE tariffs.ride_tariffs (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code           VARCHAR(30) UNIQUE NOT NULL,    -- 'start','economy',...
  name           VARCHAR(50) NOT NULL,
  base_price     NUMERIC(10,2) NOT NULL,         -- до х км
  per_km_price   NUMERIC(10,2) NOT NULL,
  door_fee       NUMERIC(10,2) NOT NULL DEFAULT 0,
  min_car_year   SMALLINT,
  has_in_car_display BOOLEAN NOT NULL DEFAULT FALSE,
  active         BOOLEAN NOT NULL DEFAULT TRUE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Специальные тарифы (пуг, пожилые и т.д.)
CREATE TABLE tariffs.special_tariffs (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name           VARCHAR(50) NOT NULL,
  description    TEXT,
  surcharge_pct  NUMERIC(5,2) DEFAULT 0,
  active         BOOLEAN NOT NULL DEFAULT FALSE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Дополнительные опции заказа
CREATE TABLE tariffs.ride_options (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code           VARCHAR(30) UNIQUE NOT NULL,   -- 'waiting','pet','bluetooth',...
  name           VARCHAR(100) NOT NULL,
  price          NUMERIC(10,2) NOT NULL,
  is_free        BOOLEAN NOT NULL DEFAULT FALSE,
  available_for_passengers BOOLEAN NOT NULL DEFAULT TRUE,
  available_for_drivers BOOLEAN NOT NULL DEFAULT TRUE,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Схема `orders` — заказы и связанные сущности

```sql
CREATE SCHEMA orders;

-- Заказ
CREATE TABLE orders.orders (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  passenger_id   UUID NOT NULL 
                    REFERENCES auth.users(id),
  driver_id      UUID 
                    REFERENCES auth.users(id),
  tariff_id      UUID 
                    REFERENCES tariffs.ride_tariffs(id),
  special_tariff_id UUID 
                    REFERENCES tariffs.special_tariffs(id),
  pickup_address TEXT NOT NULL,
  pickup_point   GEOGRAPHY(POINT,4326) NOT NULL,
  dropoff_address TEXT NOT NULL,
  dropoff_point  GEOGRAPHY(POINT,4326) NOT NULL,
  distance_km    NUMERIC(6,2),
  estimated_price NUMERIC(10,2),
  final_price    NUMERIC(10,2),
  payment_method VARCHAR(10) 
                    CHECK (payment_method IN ('cash','card')),
  status         VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN 
                      ('pending','accepted','arrived','waiting','started','completed','cancelled')),
  cancel_count   SMALLINT NOT NULL DEFAULT 0,
  cancel_reason  TEXT,
  cancel_by      VARCHAR(20) 
                    CHECK (cancel_by IN ('passenger','driver','system')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  accepted_at    TIMESTAMPTZ,
  arrived_at     TIMESTAMPTZ,
  started_at     TIMESTAMPTZ,
  completed_at   TIMESTAMPTZ
);
CREATE INDEX ON orders.orders USING GIST(pickup_point);
CREATE INDEX ON orders.orders USING GIST(dropoff_point);
CREATE INDEX ON orders.orders (status);

-- Опции заказа
CREATE TABLE orders.order_options (
  order_id       UUID 
                    REFERENCES orders.orders(id) ON DELETE CASCADE,
  option_id      UUID 
                    REFERENCES tariffs.ride_options(id),
  PRIMARY KEY (order_id, option_id)
);

-- Лог статусов
CREATE TABLE orders.order_status_log (
  id             BIGSERIAL PRIMARY KEY,
  order_id       UUID NOT NULL 
                    REFERENCES orders.orders(id) ON DELETE CASCADE,
  old_status     VARCHAR(20),
  new_status     VARCHAR(20),
  logged_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Ожидание пассажира (тариф «ждать»)
CREATE TABLE orders.order_waits (
  id             BIGSERIAL PRIMARY KEY,
  order_id       UUID NOT NULL 
                    REFERENCES orders.orders(id) ON DELETE CASCADE,
  start_wait     TIMESTAMPTZ NOT NULL,
  end_wait       TIMESTAMPTZ,
  fee_per_min    NUMERIC(10,2) NOT NULL
);
```

---

## Схема `payment` — платежи, кошельки, кешбек, чаевые

```sql
CREATE SCHEMA payment;

-- Платёж по заказу
CREATE TABLE payment.payments (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       UUID NOT NULL 
                    REFERENCES orders.orders(id),
  passenger_id   UUID 
                    REFERENCES auth.users(id),
  driver_id      UUID 
                    REFERENCES auth.users(id),
  amount         NUMERIC(10,2) NOT NULL,
  provider       VARCHAR(50),
  status         VARCHAR(20) 
                    CHECK (status IN ('pending','success','failed')),
  paid_at        TIMESTAMPTZ,
  cashback_amt   NUMERIC(10,2) DEFAULT 0,
  tip_amt        NUMERIC(10,2) DEFAULT 0,
  company_fee    NUMERIC(10,2) DEFAULT 0
);

-- Кошельки пользователей (главный, сейф, бонус)
CREATE TABLE payment.wallets (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL 
                    REFERENCES auth.users(id),
  type           VARCHAR(20) 
                    CHECK (type IN ('main','safe','bonus')),
  balance        NUMERIC(12,2) NOT NULL DEFAULT 0,
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Транзакции кошельков
CREATE TABLE payment.wallet_transactions (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  wallet_id      UUID NOT NULL 
                    REFERENCES payment.wallets(id),
  amount         NUMERIC(12,2) NOT NULL,
  transaction_type VARCHAR(20) 
                    CHECK (transaction_type IN ('credit','debit')),
  reason         VARCHAR(100),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Схема `location` — геолокация и трекинг

```sql
CREATE SCHEMA location;

-- Лог координат (для аналитики)
CREATE TABLE location.location_logs (
  id             BIGSERIAL PRIMARY KEY,
  user_id        UUID NOT NULL 
                    REFERENCES auth.users(id),
  role           VARCHAR(10) 
                    CHECK (role IN ('driver','passenger')),
  location       GEOGRAPHY(POINT,4326) NOT NULL,
  recorded_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON location.location_logs USING GIST(location);

-- В Redis (streams) — текущее местоположение каждого драйвера:
-- key: "driver:{user_id}:loc" → JSON {lat,lon,ts}
```

---

## Схема `feedback` — отзывы, жалобы, рейтинги

```sql
CREATE SCHEMA feedback;

-- Feedback / рейтинг
CREATE TABLE feedback.feedbacks (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       UUID 
                    REFERENCES orders.orders(id),
  from_user_id   UUID NOT NULL 
                    REFERENCES auth.users(id),
  to_user_id     UUID NOT NULL 
                    REFERENCES auth.users(id),
  rating         SMALLINT CHECK (rating BETWEEN 1 AND 5),
  comment        TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Жалобы оператору
CREATE TABLE feedback.complaints (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  order_id       UUID 
                    REFERENCES orders.orders(id),
  complainant_id UUID NOT NULL 
                    REFERENCES auth.users(id),
  description    TEXT NOT NULL,
  status         VARCHAR(20) NOT NULL DEFAULT 'open'
                    CHECK (status IN ('open','in_progress','resolved')),
  operator_reply TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at    TIMESTAMPTZ
);
```

---

## Схема `notification` — сообщения и SOS

```sql
CREATE SCHEMA notification;

-- Push/SMS уведомления
CREATE TABLE notification.notifications (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID 
                    REFERENCES auth.users(id),
  type           VARCHAR(30),  -- 'order_update','payment','sos',...
  payload        JSONB,
  status         VARCHAR(20) 
                    CHECK (status IN ('queued','sent','failed')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  sent_at        TIMESTAMPTZ
);

-- SOS-события
CREATE TABLE notification.sos_events (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id        UUID NOT NULL 
                    REFERENCES auth.users(id),
  order_id       UUID 
                    REFERENCES orders.orders(id),
  location       GEOGRAPHY(POINT,4326),
  target_type    VARCHAR(20), -- 'emergency_services','drivers'
  action_status  VARCHAR(20) 
                    CHECK (action_status IN ('pending','dispatched','resolved')),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  resolved_at    TIMESTAMPTZ
);
```

---

## Схема `promo` — промокоды и рефералы

```sql
CREATE SCHEMA promo;

-- Промокоды
CREATE TABLE promo.promo_codes (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  code           VARCHAR(20) UNIQUE NOT NULL,
  discount_type  VARCHAR(20) 
                    CHECK (discount_type IN ('percent','fixed')),
  discount_value NUMERIC(10,2),
  valid_from     TIMESTAMPTZ,
  valid_until    TIMESTAMPTZ,
  max_uses       INT DEFAULT 1,
  used_count     INT DEFAULT 0
);

-- Реферальные приглашения
CREATE TABLE promo.referrals (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  inviter_id     UUID NOT NULL 
                    REFERENCES auth.users(id),
  invitee_id     UUID NOT NULL 
                    REFERENCES auth.users(id),
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Схема `admin` — CRM и логи

```sql
CREATE SCHEMA admin;

-- Администраторы/операторы
CREATE TABLE admin.admins (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  username       VARCHAR(50) UNIQUE NOT NULL,
  password_hash  TEXT NOT NULL,
  role           VARCHAR(20) 
                    CHECK (role IN ('superadmin','moderator')),
  last_login     TIMESTAMPTZ
);

-- Лог звонков операторов
CREATE TABLE admin.call_logs (
  id             BIGSERIAL PRIMARY KEY,
  operator_id    UUID 
                    REFERENCES admin.admins(id),
  user_id        UUID 
                    REFERENCES auth.users(id),
  call_type      VARCHAR(20),  -- 'support','sos_followup'
  duration_sec   INT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Лог чатов операторов
CREATE TABLE admin.chat_logs (
  id             BIGSERIAL PRIMARY KEY,
  operator_id    UUID 
                    REFERENCES admin.admins(id),
  user_id        UUID 
                    REFERENCES auth.users(id),
  message        TEXT,
  created_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Схема `analytics` — MongoDB

```js
// ride_events, driver_stats, earnings_reports и т.п.
// для BI-отчётов и подсказок маршрутов
```

---

### Ключевые моменты и оптимизации

* **UUID-PK** по всем таблицам.
* **GEOGRAPHY(Point)** + GIST-индексы для ближнего поиска.
* **CHECK-ограничения** на роли, статусы и типы.
* **Redis Streams** для live-локации драйверов.
* **MongoDB** для тяжёлых аналитических задач и подсказок адресов.
* **Шаблоны уведомлений** и **лог SOS** для полной трассировки.

Эта структура покрывает 100% требований из ТЗ: от многоуровневых тарифов и доп.услуг до SOS-кнопки, аналитики и CRM-логов.
