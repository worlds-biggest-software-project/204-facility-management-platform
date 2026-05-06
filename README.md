# Facility Management Platform

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for space planning, maintenance work orders, and vendor management across multi-site building portfolios.

The Facility Management Platform unifies CMMS, IWMS, and occupant-experience workflows into a single system designed from the ground up around AI. It targets corporate real estate and facilities teams, healthcare and campus operators, and property managers who need predictive maintenance, space optimisation, and natural-language service delivery without enterprise-scale implementation cycles.

---

## Why Facility Management Platform?

- Enterprise IWMS incumbents (Archibus, Planon, IBM TRIRIGA) typically cost £30,000–£150,000+ per year and require multi-year implementations, locking out mid-market operators.
- IBM Maximo starts at ~$164/user/month and is widely cited for a dated UX and 6–18 month rollout cycles.
- CMMS-focused tools (UpKeep, Fiix) at $45/user/month are easy to adopt but lack space planning, real estate, and occupancy analytics.
- ServiceNow Workplace Service Delivery is IT-first and not purpose-built for FM depth; pure-play IWMS tools like FM:Systems and Spacewell are weaker on maintenance.
- No incumbent is built bottom-up as AI-first; AI is layered onto rule-based CMMS foundations rather than being native to the architecture.

---

## Key Features

### Work Order & Maintenance Management

- Work order creation, assignment, status tracking, and mobile completion
- Preventive maintenance scheduling with time-based and meter-based triggers
- Asset registry with lifecycle, age, and maintenance history tracking
- Technician mobile app with offline capability for field use
- Real-time work order analytics dashboard (cycle time, reliability, PM compliance)

### Space, Real Estate & Occupancy

- Space and occupancy analytics with utilisation heatmaps
- Multi-location portfolio support across sites and buildings
- Move management and space reconfiguration workflows
- Occupancy forecasting for hybrid work planning
- Lease and warranty tracking with renewal alerting

### IoT, Energy & Sustainability

- IoT sensor integration for temperature, humidity, and occupancy
- Energy management dashboard with consumption tracking
- Real-time energy anomaly detection and root-cause hypotheses
- Sustainability and ESG reporting automation
- BIM/CAD integration for automatic space and asset data synchronisation

### Occupant & Tenant Experience

- Self-service occupant portal for issue reporting and service requests
- Conversational AI assistant for occupants via web, mobile, Teams, and Slack
- Space and desk booking integrated into FM workflows
- Vendor and contractor portals for external service coordination

---

## AI-Native Advantage

Predictive maintenance fuses IoT sensor streams, historical work-order data, and equipment age to forecast failures days or weeks ahead. Natural-language work-order triage reads free-text fault reports, classifies severity, routes to the nearest qualified technician, and drafts work instructions automatically. AI-driven space optimisation analyses badge, calendar, and sensor data to recommend reconfigurations, while real-time energy anomaly detection identifies consumption outliers across an entire portfolio and proposes root causes for facilities engineers.

---

## Tech Stack & Deployment

The platform is expected to support both cloud-hosted and self-hosted deployments to accommodate enterprise data-residency requirements. Integration is anchored on open standards including ISO 41001 (FM systems), ISO 55000 (asset management), BOMA space measurement, COBie for built-asset data exchange, and ENERGY STAR / ISO 50001 for energy reporting. IoT connectivity follows MQTT and REST patterns; BIM and CAD synchronisation aligns with the bidirectional patterns used by Planon Connect with AutoCAD and Revit.

---

## Market Context

The global FM software market is estimated at USD 26.3 billion in 2025, projected to reach USD 81.8 billion by 2035 at a 12% CAGR (Future Market Insights, 2025). The US market alone is projected to grow from USD 14.9 billion in 2025 to USD 30.6 billion by 2030 (MarketsandMarkets, 2025). Primary buyers are corporate real estate and facilities directors, healthcare and campus operators, multi-site property managers, and sustainability officers driven by ESG reporting mandates.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
