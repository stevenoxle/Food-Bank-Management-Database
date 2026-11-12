# Food-Bank-Management-Database


Java JDBC URL: jdbc:mysql://localhost:3306/food_bank_simple



ACCESS RULES (must be enforced in Java, NOT by merely storing the role):

COORDINATOR: full application access; creates volunteer accounts; approve

recipients; manages users/shifts/inventory/reports/alerts.

VOLUNTEER:   process donations, manage inventory, assigned distributions.

DONOR:       own account and own donations only. Filter donor_id = session ID.

RECIPIENT:   own account/requests/distribution history; only ACTIVE can request.
