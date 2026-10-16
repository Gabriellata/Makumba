# Makumba
WarsztatPro

BEGIN;

-- =========================================================
-- 1. UZYTKOWNICY
-- Role: admin - konfiguracja programu
--       user  - codzienna obsluga warsztatu
-- =========================================================

CREATE TABLE users (
    id            BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    username      VARCHAR(100) NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    role          VARCHAR(10) NOT NULL DEFAULT 'user'
        CHECK (role IN ('admin', 'user')),
    is_active     BOOLEAN NOT NULL DEFAULT TRUE,
    created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================
-- 2. KLIENCI
-- =========================================================

CREATE TABLE customers (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name            VARCHAR(200) NOT NULL,
    phone           VARCHAR(30),
    email           VARCHAR(255),
    postal_code     VARCHAR(12),
    city            VARCHAR(100),
    street_address  VARCHAR(200),
    tax_id          VARCHAR(20),
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_archived     BOOLEAN NOT NULL DEFAULT FALSE
);


-- =========================================================
-- 3. TYPY POJAZDOW
-- Konfigurowalne przez administratora
-- =========================================================

CREATE TABLE vehicle_types (
    id          BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name        VARCHAR(80) NOT NULL UNIQUE,
    is_active   BOOLEAN NOT NULL DEFAULT TRUE,
    sort_order  INTEGER NOT NULL DEFAULT 0
);


-- =========================================================
-- 4. POJAZDY
-- Jeden klient moze posiadac wiele pojazdow.
-- =========================================================

CREATE TABLE vehicles (
    id                  BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id         BIGINT NOT NULL
        REFERENCES customers(id) ON DELETE RESTRICT,
    vehicle_type_id     BIGINT NOT NULL
        REFERENCES vehicle_types(id) ON DELETE RESTRICT,

    registration_number VARCHAR(20),
    vin                 VARCHAR(17),
    make                VARCHAR(100),
    model               VARCHAR(100),
    version             VARCHAR(150),
    production_year     SMALLINT,
    engine_capacity_cc  INTEGER,
    fuel_type           VARCHAR(30),
    power_kw            NUMERIC(7,2),
    drivetrain          VARCHAR(30),
    body_type           VARCHAR(50),
    transmission        VARCHAR(30),

    notes               TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_archived         BOOLEAN NOT NULL DEFAULT FALSE,

    CHECK (
        production_year IS NULL
        OR production_year BETWEEN 1886 AND 2200
    ),
    CHECK (
        engine_capacity_cc IS NULL
        OR engine_capacity_cc >= 0
    ),
    CHECK (power_kw IS NULL OR power_kw >= 0)
);


-- =========================================================
-- 5. KATALOG USLUG
-- Jedna cena domyslna; cene mozna zmienic na zleceniu.
-- =========================================================

CREATE TABLE services (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    name              VARCHAR(200) NOT NULL,
    category          VARCHAR(100),
    description       TEXT,

    unit              VARCHAR(20) NOT NULL DEFAULT 'service'
        CHECK (unit IN ('service', 'hour', 'piece')),

    default_price     NUMERIC(12,2) NOT NULL DEFAULT 0
        CHECK (default_price >= 0),

    estimated_minutes INTEGER
        CHECK (
            estimated_minutes IS NULL
            OR estimated_minutes >= 0
        ),

    is_active         BOOLEAN NOT NULL DEFAULT TRUE,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================
-- 6. ZLECENIA
-- Klient i pojazd sa zapisani bezposrednio w zleceniu.
-- Dzieki temu zmiana aktualnego wlasciciela pojazdu
-- nie zmienia klienta na historycznym zleceniu.
-- =========================================================

CREATE TABLE work_orders (
    id                 BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    order_number       VARCHAR(40) NOT NULL UNIQUE,

    customer_id        BIGINT NOT NULL
        REFERENCES customers(id) ON DELETE RESTRICT,

    vehicle_id         BIGINT NOT NULL
        REFERENCES vehicles(id) ON DELETE RESTRICT,

    assigned_user_id   BIGINT
        REFERENCES users(id) ON DELETE SET NULL,

    created_by         BIGINT
        REFERENCES users(id) ON DELETE SET NULL,

    status             VARCHAR(30) NOT NULL DEFAULT 'received'
        CHECK (status IN (
            'received',
            'diagnosis',
            'awaiting_approval',
            'awaiting_parts',
            'in_progress',
            'quality_check',
            'ready_for_pickup',
            'closed',
            'cancelled'
        )),

    complaint          TEXT,
    diagnosis          TEXT,
    internal_notes     TEXT,

    odometer_km        INTEGER
        CHECK (odometer_km IS NULL OR odometer_km >= 0),

    received_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    planned_start_at   TIMESTAMPTZ,
    due_at             TIMESTAMPTZ,
    started_at         TIMESTAMPTZ,
    completed_at       TIMESTAMPTZ,
    closed_at          TIMESTAMPTZ,
    cancelled_at       TIMESTAMPTZ,
    cancellation_reason TEXT,

    created_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at         TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    is_archived        BOOLEAN NOT NULL DEFAULT FALSE
);


-- =========================================================
-- 7. POZYCJE ZLECENIA / KOSZTORYSU
-- Nazwa i cena sa kopiowane z katalogu podczas dodawania.
-- Zmiana cennika nie zmienia historycznych pozycji.
-- =========================================================

CREATE TABLE work_order_items (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    work_order_id     BIGINT NOT NULL
        REFERENCES work_orders(id) ON DELETE RESTRICT,

    service_id        BIGINT
        REFERENCES services(id) ON DELETE SET NULL,

    item_type         VARCHAR(20) NOT NULL DEFAULT 'service'
        CHECK (item_type IN ('service', 'part', 'other')),

    name              VARCHAR(200) NOT NULL,
    description       TEXT,

    quantity          NUMERIC(10,2) NOT NULL DEFAULT 1
        CHECK (quantity > 0),

    unit              VARCHAR(20) NOT NULL DEFAULT 'service',

    unit_price        NUMERIC(12,2) NOT NULL DEFAULT 0
        CHECK (unit_price >= 0),

    discount_percent  NUMERIC(5,2) NOT NULL DEFAULT 0
        CHECK (discount_percent BETWEEN 0 AND 100),

    vat_rate          NUMERIC(5,2) NOT NULL DEFAULT 23
        CHECK (vat_rate BETWEEN 0 AND 100),

    status            VARCHAR(20) NOT NULL DEFAULT 'pending'
        CHECK (status IN (
            'pending',
            'in_progress',
            'completed',
            'on_hold',
            'cancelled'
        )),

    assigned_user_id  BIGINT
        REFERENCES users(id) ON DELETE SET NULL,

    estimated_minutes INTEGER
        CHECK (
            estimated_minutes IS NULL
            OR estimated_minutes >= 0
        ),

    actual_minutes    INTEGER
        CHECK (
            actual_minutes IS NULL
            OR actual_minutes >= 0
        ),

    sort_order        INTEGER NOT NULL DEFAULT 0,
    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================
-- 8. TERMINARZ
-- Bez stanowisk warsztatowych.
-- Wizyta moze istniec bez utworzonego zlecenia.
-- =========================================================

CREATE TABLE appointments (
    id                BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    customer_id       BIGINT NOT NULL
        REFERENCES customers(id) ON DELETE RESTRICT,

    vehicle_id        BIGINT
        REFERENCES vehicles(id) ON DELETE SET NULL,

    work_order_id     BIGINT
        REFERENCES work_orders(id) ON DELETE SET NULL,

    assigned_user_id  BIGINT
        REFERENCES users(id) ON DELETE SET NULL,

    title             VARCHAR(200) NOT NULL,
    description       TEXT,

    starts_at         TIMESTAMPTZ NOT NULL,
    ends_at           TIMESTAMPTZ NOT NULL,

    status            VARCHAR(20) NOT NULL DEFAULT 'scheduled'
        CHECK (status IN (
            'scheduled',
            'confirmed',
            'in_progress',
            'completed',
            'cancelled',
            'no_show'
        )),

    created_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    CHECK (ends_at > starts_at)
);


-- =========================================================
-- 9. ZALACZNIKI ZLECEN
-- Pliki przechowujemy poza baza; w bazie zapisujemy
-- identyfikator lokalizacji pliku.
-- =========================================================

CREATE TABLE work_order_attachments (
    id              BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,

    work_order_id   BIGINT NOT NULL
        REFERENCES work_orders(id) ON DELETE RESTRICT,

    uploaded_by     BIGINT
        REFERENCES users(id) ON DELETE SET NULL,

    file_name       VARCHAR(255) NOT NULL,
    storage_key     TEXT NOT NULL,
    mime_type       VARCHAR(100),

    file_size_bytes BIGINT
        CHECK (
            file_size_bytes IS NULL
            OR file_size_bytes >= 0
        ),

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);


-- =========================================================
-- 10. INDEKSY
-- Przyspieszaja wyszukiwanie i listy w aplikacji.
-- =========================================================

CREATE INDEX idx_customers_name
    ON customers(name);

CREATE INDEX idx_customers_phone
    ON customers(phone);

CREATE INDEX idx_vehicles_registration
    ON vehicles(registration_number);

CREATE INDEX idx_vehicles_vin
    ON vehicles(vin);

CREATE INDEX idx_vehicles_customer
    ON vehicles(customer_id);

CREATE INDEX idx_orders_status
    ON work_orders(status, received_at DESC);

CREATE INDEX idx_orders_customer
    ON work_orders(customer_id, received_at DESC);

CREATE INDEX idx_orders_vehicle
    ON work_orders(vehicle_id, received_at DESC);

CREATE INDEX idx_orders_due
    ON work_orders(due_at)
    WHERE is_archived = FALSE;

CREATE INDEX idx_order_items_order
    ON work_order_items(work_order_id, sort_order);

CREATE INDEX idx_appointments_start
    ON appointments(starts_at);

CREATE INDEX idx_appointments_customer
    ON appointments(customer_id, starts_at);

CREATE INDEX idx_attachments_order
    ON work_order_attachments(work_order_id, created_at DESC);


-- =========================================================
-- 11. AUTOMATYCZNA AKTUALIZACJA updated_at
-- DEFAULT NOW() ustawia wartosc tylko przy INSERT.
-- =========================================================

CREATE FUNCTION set_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_customers_updated_at
BEFORE UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_vehicles_updated_at
BEFORE UPDATE ON vehicles
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_services_updated_at
BEFORE UPDATE ON services
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_work_orders_updated_at
BEFORE UPDATE ON work_orders
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

CREATE TRIGGER trg_appointments_updated_at
BEFORE UPDATE ON appointments
FOR EACH ROW EXECUTE FUNCTION set_updated_at();

COMMIT;
