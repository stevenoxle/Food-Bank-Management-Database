-- Food Bank Management Databse
-- INFO 2323 Group Project
-- Steven Le  100326241
-- Database ran for Java implementation

CREATE DATABASE food_bank_simple
    CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE food_bank_simple;

-- 1/11 ACCOUNTS: all users in one table, with optional role-specific fields.
-- Role COORDINATOR = manager/admin.  Only coordinator has full app access.
-- Recipient signups start PENDING; coordinator approval changes to ACTIVE.
CREATE TABLE accounts (
    account_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    full_name VARCHAR(150) NOT NULL,
    email VARCHAR(255) NOT NULL,
    phone VARCHAR(30) NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role ENUM('COORDINATOR','VOLUNTEER','DONOR','RECIPIENT') NOT NULL,
    status ENUM('PENDING','ACTIVE','REJECTED','DISABLED') NOT NULL DEFAULT 'PENDING',
    -- Only recipients use these fields:
    household_size SMALLINT UNSIGNED NULL,
    dietary_restrictions TEXT NULL,
    allergies TEXT NULL,
    emergency_contact_name VARCHAR(150) NULL,
    emergency_contact_phone VARCHAR(30) NULL,
    approved_by BIGINT UNSIGNED NULL,
    -- Only volunteers use these fields:
    profile_image_path VARCHAR(500) NULL,
    availability_notes VARCHAR(500) NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (account_id),
    UNIQUE KEY uq_accounts_email (email),
    KEY idx_accounts_role_status (role, status),
    CONSTRAINT fk_accounts_approved_by FOREIGN KEY (approved_by)
        REFERENCES accounts(account_id) ON DELETE SET NULL,
    CONSTRAINT chk_household_size CHECK (household_size IS NULL OR household_size > 0)
) ENGINE=InnoDB;

-- 2/11 INVENTORY CATEGORIES: coordinator maintains the list.
CREATE TABLE inventory_categories (
    category_id SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
    category_name VARCHAR(100) NOT NULL,
    PRIMARY KEY (category_id),
    UNIQUE KEY uq_category_name (category_name)
) ENGINE=InnoDB;

INSERT INTO inventory_categories (category_name) VALUES
    ('Produce'), ('Dairy'), ('Grains'), ('Canned Goods'), ('Protein'),
    ('Frozen Foods'), ('Baby Supplies'), ('Hygiene'), ('Other');

-- 3/11 DONATIONS: one record per donor submission.
CREATE TABLE donations (
    donation_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    donor_id BIGINT UNSIGNED NOT NULL,
    status ENUM('SUBMITTED','RECEIVED','CANCELLED') NOT NULL DEFAULT 'SUBMITTED',
    submitted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    received_at DATETIME NULL,
    received_by BIGINT UNSIGNED NULL,
    notes TEXT NULL,
    PRIMARY KEY (donation_id),
    KEY idx_donations_donor_date (donor_id, submitted_at),
    CONSTRAINT fk_donations_donor FOREIGN KEY (donor_id)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT fk_donations_receiver FOREIGN KEY (received_by)
        REFERENCES accounts(account_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- 4/11 DONATION ITEMS + INVENTORY: no separate inventory_lots table.
-- 'quantity' is the declared/accepted amount. When volunteer receives donation,
-- Java sets quantity_available = quantity and donation.status = 'RECEIVED'
-- together in ONE transaction. Later distribution/waste reduces available stock.
CREATE TABLE donation_items (
    donation_item_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    donation_id BIGINT UNSIGNED NOT NULL,
    category_id SMALLINT UNSIGNED NOT NULL,
    item_name VARCHAR(150) NOT NULL,
    quantity DECIMAL(12,2) NOT NULL,
    unit VARCHAR(30) NOT NULL,
    expiry_date DATE NULL,
    quantity_available DECIMAL(12,2) NOT NULL DEFAULT 0,
    quantity_wasted DECIMAL(12,2) NOT NULL DEFAULT 0,
    storage_location VARCHAR(120) NULL,
    PRIMARY KEY (donation_item_id),
    KEY idx_items_category (category_id),
    KEY idx_items_expiry (expiry_date),
    CONSTRAINT fk_items_donation FOREIGN KEY (donation_id)
        REFERENCES donations(donation_id) ON DELETE RESTRICT,
    CONSTRAINT fk_items_category FOREIGN KEY (category_id)
        REFERENCES inventory_categories(category_id) ON DELETE RESTRICT,
    CONSTRAINT chk_item_qty CHECK (quantity > 0),
    CONSTRAINT chk_item_stock CHECK (
        quantity_available >= 0 AND quantity_wasted >= 0
        AND quantity_available + quantity_wasted <= quantity
    )
) ENGINE=InnoDB;

-- 5/11 DISTRIBUTION SHIFTS: coordinator creates scheduled tasks.
CREATE TABLE distribution_shifts (
    shift_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    shift_start DATETIME NOT NULL,
    shift_end DATETIME NOT NULL,
    description VARCHAR(500) NOT NULL,
    status ENUM('PLANNED','ACTIVE','COMPLETED','CANCELLED') NOT NULL DEFAULT 'PLANNED',
    created_by BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (shift_id),
    KEY idx_shifts_start (shift_start),
    CONSTRAINT fk_shifts_creator FOREIGN KEY (created_by)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT chk_shift_time CHECK (shift_end > shift_start)
) ENGINE=InnoDB;

-- 6/11 VOLUNTEER SHIFT ASSIGNMENTS: a shift can have many volunteers.
CREATE TABLE volunteer_shift_assignments (
    shift_id BIGINT UNSIGNED NOT NULL,
    volunteer_id BIGINT UNSIGNED NOT NULL,
    task_description VARCHAR(500) NULL,
    PRIMARY KEY (shift_id, volunteer_id),
    KEY idx_shift_volunteer (volunteer_id),
    CONSTRAINT fk_assignment_shift FOREIGN KEY (shift_id)
        REFERENCES distribution_shifts(shift_id) ON DELETE RESTRICT,
    CONSTRAINT fk_assignment_volunteer FOREIGN KEY (volunteer_id)
        REFERENCES accounts(account_id) ON DELETE RESTRICT
) ENGINE=InnoDB;

-- 7/11 HAMPER REQUESTS: pickup appointment + final distribution event.
-- Only ACTIVE recipients can request hampers (enforce in Java).
-- Fulfilled means actual_pickup_at is filled and items are recorded.
CREATE TABLE hamper_requests (
    request_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    recipient_id BIGINT UNSIGNED NOT NULL,
    requested_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    scheduled_pickup_at DATETIME NULL,
    assigned_shift_id BIGINT UNSIGNED NULL,
    status ENUM('REQUESTED','SCHEDULED','FULFILLED','MISSED','DELAYED','CANCELLED')
        NOT NULL DEFAULT 'REQUESTED',
    actual_pickup_at DATETIME NULL,
    processed_by BIGINT UNSIGNED NULL,
    notes TEXT NULL,
    PRIMARY KEY (request_id),
    KEY idx_request_recipient (recipient_id, requested_at),
    KEY idx_request_pickup (status, scheduled_pickup_at),
    CONSTRAINT fk_request_recipient FOREIGN KEY (recipient_id)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT fk_request_shift FOREIGN KEY (assigned_shift_id)
        REFERENCES distribution_shifts(shift_id) ON DELETE SET NULL,
    CONSTRAINT fk_request_processor FOREIGN KEY (processed_by)
        REFERENCES accounts(account_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- 8/11 DISTRIBUTION ITEMS: multiple food items per hamper.
-- Java must ensure each quantity <= available stock, then decrement stock
-- and insert items in ONE database transaction (SELECT ... FOR UPDATE).
CREATE TABLE distribution_items (
    distribution_item_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    request_id BIGINT UNSIGNED NOT NULL,
    donation_item_id BIGINT UNSIGNED NOT NULL,
    quantity DECIMAL(12,2) NOT NULL,
    PRIMARY KEY (distribution_item_id),
    UNIQUE KEY uq_distribution_request_item (request_id, donation_item_id),
    KEY idx_distribution_food_item (donation_item_id),
    CONSTRAINT fk_distribution_request FOREIGN KEY (request_id)
        REFERENCES hamper_requests(request_id) ON DELETE RESTRICT,
    CONSTRAINT fk_distribution_food_item FOREIGN KEY (donation_item_id)
        REFERENCES donation_items(donation_item_id) ON DELETE RESTRICT,
    CONSTRAINT chk_distribution_qty CHECK (quantity > 0)
) ENGINE=InnoDB;

-- 9/11 INVENTORY WASTE: preserve loss history for monthly/yearly reports.
-- Java must decrement quantity_available and increment quantity_wasted
-- in the SAME transaction as inserting this record.
CREATE TABLE inventory_waste (
    waste_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    donation_item_id BIGINT UNSIGNED NOT NULL,
    quantity DECIMAL(12,2) NOT NULL,
    reason ENUM('EXPIRED','SPOILED','DAMAGED','OTHER') NOT NULL,
    wasted_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    recorded_by BIGINT UNSIGNED NOT NULL,
    PRIMARY KEY (waste_id),
    KEY idx_waste_date (wasted_at),
    CONSTRAINT fk_waste_item FOREIGN KEY (donation_item_id)
        REFERENCES donation_items(donation_item_id) ON DELETE RESTRICT,
    CONSTRAINT fk_waste_user FOREIGN KEY (recorded_by)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT chk_waste_qty CHECK (quantity > 0)
) ENGINE=InnoDB;

-- 10/11 PICKUP STATUS EVENTS: history is necessary for missed/delayed report
-- even when a late pickup eventually becomes FULFILLED.
CREATE TABLE pickup_status_events (
    event_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    request_id BIGINT UNSIGNED NOT NULL,
    status ENUM('REQUESTED','SCHEDULED','FULFILLED','MISSED','DELAYED','CANCELLED') NOT NULL,
    changed_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    changed_by BIGINT UNSIGNED NULL,
    PRIMARY KEY (event_id),
    KEY idx_pickup_request_status (request_id, status),
    CONSTRAINT fk_event_request FOREIGN KEY (request_id)
        REFERENCES hamper_requests(request_id) ON DELETE RESTRICT,
    CONSTRAINT fk_event_actor FOREIGN KEY (changed_by)
        REFERENCES accounts(account_id) ON DELETE SET NULL
) ENGINE=InnoDB;

-- 11/11 SYSTEM ALERTS: one row per person to email (no delivery join table).
-- Java generates expiry/missed/delayed alerts and sends real emails.
-- For expiry risk, the subject/body can include the rule-based calculation.
CREATE TABLE system_alerts (
    alert_id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    alert_type ENUM('PICKUP_MISSED','PICKUP_DELAYED','INVENTORY_AT_RISK') NOT NULL,
    request_id BIGINT UNSIGNED NULL,
    donation_item_id BIGINT UNSIGNED NULL,
    recipient_account_id BIGINT UNSIGNED NOT NULL,
    email_to VARCHAR(255) NOT NULL,
    subject VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    delivery_status ENUM('QUEUED','SENT','FAILED') NOT NULL DEFAULT 'QUEUED',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    sent_at DATETIME NULL,
    last_error VARCHAR(500) NULL,
    PRIMARY KEY (alert_id),
    KEY idx_alert_delivery (delivery_status, created_at),
    CONSTRAINT fk_alert_request FOREIGN KEY (request_id)
        REFERENCES hamper_requests(request_id) ON DELETE RESTRICT,
    CONSTRAINT fk_alert_item FOREIGN KEY (donation_item_id)
        REFERENCES donation_items(donation_item_id) ON DELETE RESTRICT,
    CONSTRAINT fk_alert_recipient FOREIGN KEY (recipient_account_id)
        REFERENCES accounts(account_id) ON DELETE RESTRICT,
    CONSTRAINT chk_alert_target CHECK (
        (alert_type IN ('PICKUP_MISSED','PICKUP_DELAYED')
         AND request_id IS NOT NULL AND donation_item_id IS NULL)
        OR (alert_type = 'INVENTORY_AT_RISK'
            AND donation_item_id IS NOT NULL AND request_id IS NULL)
    )
) ENGINE=InnoDB;

-- REPORTING VIEWS: optional helpers so the Java team can SELECT instead of
-- writing long JOIN queries for each report.

-- Current inventory: only PHYSICALLY RECEIVED donations count as stock.
CREATE OR REPLACE VIEW vw_current_inventory AS
SELECT di.donation_item_id, di.item_name, c.category_name,
       di.quantity, di.quantity_available, di.quantity_wasted,
       di.unit, di.expiry_date, di.storage_location,
       d.donation_id, d.donor_id
FROM donation_items di
JOIN donations d ON d.donation_id = di.donation_id
JOIN inventory_categories c ON c.category_id = di.category_id
WHERE d.status = 'RECEIVED';

-- Donation totals per category: don't add incompatible measurement units.
CREATE OR REPLACE VIEW vw_monthly_donations AS
SELECT YEAR(d.received_at) AS report_year,
       MONTH(d.received_at) AS report_month,
       c.category_name, di.unit,
       SUM(di.quantity) AS total_quantity_received
FROM donations d
JOIN donation_items di ON di.donation_id = d.donation_id
JOIN inventory_categories c ON c.category_id = di.category_id
WHERE d.status = 'RECEIVED' AND d.received_at IS NOT NULL
GROUP BY YEAR(d.received_at), MONTH(d.received_at), c.category_name, di.unit;

CREATE OR REPLACE VIEW vw_monthly_hampers AS
SELECT YEAR(actual_pickup_at) AS report_year,
       MONTH(actual_pickup_at) AS report_month,
       COUNT(*) AS total_hampers_distributed
FROM hamper_requests
WHERE status = 'FULFILLED' AND actual_pickup_at IS NOT NULL
GROUP BY YEAR(actual_pickup_at), MONTH(actual_pickup_at);

-- Count distinct impacted requests, not repeated attempts/events for one request.
CREATE OR REPLACE VIEW vw_monthly_pickup_issues AS
SELECT YEAR(hr.scheduled_pickup_at) AS report_year,
       MONTH(hr.scheduled_pickup_at) AS report_month,
       hr.recipient_id, a.full_name AS recipient_name,
       COUNT(DISTINCT CASE WHEN e.status = 'MISSED' THEN hr.request_id END) AS missed_pickups,
       COUNT(DISTINCT CASE WHEN e.status = 'DELAYED' THEN hr.request_id END) AS delayed_pickups,
       COUNT(DISTINCT CASE WHEN e.status IN ('MISSED','DELAYED') THEN hr.request_id END)
           AS total_affected_pickups
FROM hamper_requests hr
JOIN accounts a ON a.account_id = hr.recipient_id
JOIN pickup_status_events e ON e.request_id = hr.request_id
WHERE hr.scheduled_pickup_at IS NOT NULL
GROUP BY YEAR(hr.scheduled_pickup_at), MONTH(hr.scheduled_pickup_at),
         hr.recipient_id, a.full_name
HAVING total_affected_pickups > 0;

CREATE OR REPLACE VIEW vw_monthly_waste AS
SELECT YEAR(w.wasted_at) AS report_year,
       MONTH(w.wasted_at) AS report_month,
       c.category_name, di.unit, SUM(w.quantity) AS total_quantity_wasted
FROM inventory_waste w
JOIN donation_items di ON di.donation_item_id = w.donation_item_id
JOIN inventory_categories c ON c.category_id = di.category_id
