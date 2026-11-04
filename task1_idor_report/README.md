# Task 1: Black-Box IDOR/BOLA Discovery

Part of the Datheon Application Security & API Pentesting Internship.

Target: OWASP crAPI, run locally via Docker Compose (https://github.com/OWASP/crAPI), an intentionally vulnerable application built for this exact kind of training.

## Finding

A confirmed Broken Object Level Authorization (BOLA/IDOR) in the vehicle location endpoint. Full details, reproduction steps, impact, and a CVSS 3.1 score (6.5, Medium) are in `disclosure_report.md`.

## Files

    disclosure_report.md         the full vulnerability disclosure report
    vehicle_idor_results.txt     raw evidence: 3 vehicle IDs, 3 different users' data, one unrelated token
    full_auth_dashboard.json     baseline: authenticated dashboard request
    no_auth_dashboard.json       baseline: unauthenticated dashboard request (secondary finding)
    full_auth_vehicles.json      baseline: authenticated vehicle list (secure, for comparison)
    no_auth_vehicles.json        baseline: unauthenticated vehicle list (correctly rejected)
    screenshots/                 evidence screenshots

## How to reproduce

1. Clone and run OWASP crAPI locally: `docker compose up -d` from its `deploy/docker` folder.
2. Create two separate user accounts.
3. Follow the steps in `disclosure_report.md`.

## Ethical note

All testing was performed against a local, intentionally vulnerable training application on my own machine. No production system or real user was tested.
