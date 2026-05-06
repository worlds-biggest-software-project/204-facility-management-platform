# Facility Management Platform — Feature & Functionality Survey

> Candidate #204 · Researched: 2026-05-03

## Solutions Analysed

| Tool | Type | Licence / Model | URL |
|------|------|-----------------|-----|
| IBM Maximo | Commercial SaaS / On-Premise | Proprietary; from $164/user/month | https://www.ibm.com/products/maximo |
| Archibus | Commercial Cloud / On-Premise | Proprietary; custom quote | https://www.archibus-phil.com/ |
| Planon | Commercial SaaS | Proprietary; custom quote | https://planonsoftware.com/us/software/iwms/ |
| IBM TRIRIGA | Commercial SaaS | Proprietary; custom quote | https://www.ibm.com/products/tririga |
| ServiceNow Workplace Service Delivery | Commercial SaaS | Proprietary; custom quote | https://www.servicenow.com/products/workplace-service-delivery.html |
| Facilio | Commercial SaaS | Proprietary; custom quote | https://facilio.com/ |
| FM:Systems | Commercial SaaS / Cloud | Proprietary; ~£20–£60/user/month | https://fmsystems.com/ |
| UpKeep | Commercial SaaS | Proprietary; from $45/user/month | https://upkeep.com/ |
| Fiix (Rockwell Automation) | Commercial SaaS | Proprietary; from $45/user/month | https://fiixsoftware.com/ |
| Spacewell | Commercial SaaS | Proprietary; custom quote | https://spacewell.com/ |

## Feature Analysis by Solution

### IBM Maximo

**Core features**
- Work order creation, assignment, and tracking with resource scheduling
- Preventive maintenance with frequency-based scheduling (time-based, meter-based, or combined triggers)
- Asset lifecycle management and IoT integration
- Job plan sequences for multi-level maintenance at varying intervals
- AI-powered anomaly detection and predictive analytics

**Differentiating features**
- Deep integration with Oracle and IBM enterprise suites
- Advanced asset performance monitoring across HVAC, lighting, elevators, plumbing, and more
- Industry-leading asset history and degradation tracking

**UX patterns**
- Complex, enterprise-focused interface requiring significant training
- Granular role-based access controls suited to large organisations
- Heavy configuration for custom workflows

**Integration points**
- APIs for third-party system integration
- Mobile app for work order management
- IoT sensor connectivity for real-time monitoring

**Known gaps**
- Dated user interface requiring modernisation
- Long implementation cycles (6–18 months typical)
- Limited space planning and real estate capabilities

**Licence / IP notes**
- Proprietary SaaS and on-premise options
- IBM-ecosystem dependency; pricing tied to user count and modules

---

### Archibus

**Core features**
- Comprehensive space planning and forecasting from portfolio to room level
- Real estate and lease management integrated with space utilisation data
- Maintenance management with work order and asset tracking
- Capital planning and facility operations
- Building operations and sustainability reporting

**Differentiating features**
- Unified view of leases alongside space use for strategic planning
- Colour-coded space allocation and team grouping tools
- "What-if" scenario planning for facility reconfigurations
- Strong market presence in large enterprise and campus environments

**UX patterns**
- Complex interface supporting multiple concurrent workflows
- Hierarchical space and asset organisation mirrors typical corporate FM structures
- Visual floor plan editors with drag-and-drop space definition

**Integration points**
- APIs for external data sources
- Real estate data syndication
- Multi-site portfolio consolidation

**Known gaps**
- Complex deployment and configuration
- Limited cloud-native agility and modern DevOps integration
- Smaller innovation cycle compared to SaaS-native tools

**Licence / IP notes**
- Proprietary on-premise and cloud options
- Custom quote pricing model; significant consulting costs

---

### Planon

**Core features**
- Space and workplace services management
- Real estate management with lease tracking
- Maintenance management workflows
- Energy management and sustainability dashboards
- IoT integration for smart building operations

**Differentiating features**
- Bidirectional Building Information Model (BIM) integration for automatic space data synchronisation
- Planon Connect for BIM enables direct AutoCAD integration with automatic sync
- IoT workplace sensor connectivity for occupancy, movement, and environmental monitoring
- Automated lighting and temperature control via IoT integration

**UX patterns**
- Progressive disclosure of complexity based on user role
- Visual space and asset browsers with geospatial context
- Occupancy-analytics dashboards with drill-down capabilities

**Integration points**
- BIM data exchange (bidirectional with AutoCAD and Revit)
- IoT platform connectivity (occupancy, environmental sensors)
- API for third-party integrations
- Energy management system APIs

**Known gaps**
- Primarily European product; lower market penetration in North America
- Implementation complexity remains high
- Less tightly integrated with CMMS workflows than pure-play CMMS tools

**Licence / IP notes**
- Proprietary SaaS; custom quote pricing
- No apparent open-source components or licensing concerns

---

### IBM TRIRIGA

**Core features**
- Real estate portfolio optimisation and space management
- Capital project tracking and budgeting
- Lease accounting (ASC 842 compliant)
- Facilities and workplace management
- Real estate analytics and forecasting

**Differentiating features**
- Deep lease accounting integration for financial consolidation
- Multi-property portfolio analytics and cost allocation
- Occupancy forecasting tied to financial models
- Strong alignment with IBM enterprise stack (SAP, Oracle)

**UX patterns**
- Finance-oriented dashboards and reporting
- Lease-centric workflows with audit trails
- Consolidated view of real estate spend across the organisation

**Integration points**
- APIs for financial system integration
- Mobile app for space bookings and asset tracking
- CAM and occupancy cost reconciliation

**Known gaps**
- Maintenance management capabilities weaker than dedicated CMMS
- Primarily designed for real estate finance, not operational excellence
- IBM-ecosystem dependency limits flexibility

**Licence / IP notes**
- Proprietary SaaS; custom quote pricing
- IBM suite licensing often bundled with other products

---

### ServiceNow Workplace Service Delivery

**Core features**
- Work order and asset management integrated with IT ticketing
- Space management and desk booking
- Visitor management and access control
- Preventive and corrective maintenance scheduling
- Employee service requests and self-service portal

**Differentiating features**
- Unified IT + FM ticketing and SLA tracking
- Integrated workplace safety compliance tooling
- Real-time analytics and dashboards
- Strong employee experience focus

**UX patterns**
- Modern, mobile-first interface
- Contextual request classification and auto-routing
- Deep IT integration (Single Sign-On, user directory sync)

**Integration points**
- APIs for enterprise tool connectivity (Slack, Teams, Outlook, SAP)
- Mobile app for technician work orders and space bookings
- Webhook support for workflow automation

**Known gaps**
- Not purpose-built for FM; IT-first architecture can limit FM-specific workflows
- Space and real estate capabilities less mature than dedicated IWMS tools
- Maintenance planning less advanced than pure CMMS solutions

**Licence / IP notes**
- Proprietary SaaS; per-user pricing (custom quote)
- Recently named Leader in IDC MarketScape for cloud-enabled FM

---

### Facilio

**Core features**
- IoT-connected maintenance management (CMMS)
- Sustainability and energy management
- Tenant experience portal with service request management
- Real-time building operations dashboards
- Compliance and vendor management

**Differentiating features**
- Modern, mobile-first user experience
- IoT-first architecture; native sensor data ingestion
- AI-powered predictive maintenance using IoT streams
- Tenant transparency and engagement portal
- End-to-end property operations in a single platform

**UX patterns**
- Dashboard-centric view with drill-down analytics
- Mobile app for technicians and tenants
- Real-time alert and escalation workflows
- Conversational AI assistant for tenant requests

**Integration points**
- IoT sensor connectivity (MQTT, REST APIs)
- API for building management systems (BMS)
- Third-party vendor and contractor portals
- Sustainability reporting APIs

**Known gaps**
- Smaller market presence outside North America
- Less mature space planning compared to IWMS-focused tools
- Newer platform with less extensive enterprise feature breadth

**Licence / IP notes**
- Proprietary SaaS; custom quote pricing
- Series B funding (2023) indicates growth trajectory
- No known IP encumbrances

---

### FM:Systems

**Core features**
- Space management with interactive floor plans and utilisation analytics
- Occupancy studies and workspace opportunity identification
- Integrated workplace management (IWMS) for multi-location portfolios
- Asset and move management
- Lease and property tracking

**Differentiating features**
- Advanced occupancy and space utilisation analytics
- Multi-dimensional analysis across facility datasets
- Manages over 3 billion square feet for 1400+ global organisations
- Strong space planning and move management workflows
- FMS:Analytics for actionable facility insights

**UX patterns**
- Web-based interface with interactive floor plan visualisations
- Drill-down analytics from portfolio to individual workspaces
- Colour-coded space utilisation heatmaps
- Mobile-friendly interfaces for occupancy studies

**Integration points**
- APIs for CAD and BIM integration
- Third-party data import (card access, occupancy sensors)
- Lease data integration from property management systems

**Known gaps**
- Weaker maintenance and CMMS capabilities than dedicated tools
- Limited IoT integration compared to modern solutions
- Less emphasis on sustainability reporting

**Licence / IP notes**
- Proprietary SaaS / cloud; ~£20–£60/user/month
- 80+ countries, 1400+ organisations; mature, widely deployed solution

---

### UpKeep

**Core features**
- Mobile-first CMMS with work order management
- Preventive maintenance scheduling (meter-based and time-based)
- Asset tracking and parts inventory
- Team collaboration with comments and status updates
- Mobile app for iOS and Android with offline capabilities
- Push notifications for work order updates

**Differentiating features**
- Easiest user adoption among CMMS tools
- Fastest time-to-value (weeks vs months)
- Barcode scanning and photo documentation
- Field technician-focused design
- Ranked #1 CMMS by Capterra and Gartner

**UX patterns**
- Simplified workflows for frontline technicians
- Visual task lists with drag-and-drop status management
- Photo and annotation tools for work order documentation
- Mobile-first; offline mode for field use

**Integration points**
- Mobile app with push notifications
- APIs for third-party integrations
- IoT sensor data for preventive maintenance triggers
- Integrations with asset management and inventory systems

**Known gaps**
- Limited space planning and real estate capabilities
- No BIM or advanced occupancy analytics
- Maintenance planning less granular than enterprise CMMS tools

**Licence / IP notes**
- Proprietary SaaS; from $45/user/month
- Series B funding; growing mid-market presence
- No known IP encumbrances

---

### Fiix (Rockwell Automation)

**Core features**
- Cloud-based CMMS with work order and asset management
- Preventive maintenance with date, time, meter, event, or condition-based triggers
- Mobile app with offline capability
- Job plan templates with task lists and parts suggestions
- Maintenance analytics and KPI reporting

**Differentiating features**
- Free CMMS tier available with no trial expiry
- Over 4,100+ maintenance teams use Fiix
- Condition-based maintenance triggers
- Integrated job plan library with repeatable processes
- Strong mid-market penetration

**UX patterns**
- Simplified, user-friendly interface
- Mobile app for real-time work order updates
- Template-based workflows for consistency
- Easy work order cloning for similar maintenance tasks

**Integration points**
- Mobile app with offline support
- APIs for asset data and work order integration
- Parts inventory management
- Rockwell Automation integration for plant equipment data

**Known gaps**
- Limited space and real estate capabilities
- No occupancy analytics or tenant experience features
- Smaller ecosystem compared to enterprise platforms

**Licence / IP notes**
- Proprietary SaaS; from $45/user/month
- Owned by Rockwell Automation; part of broader industrial automation strategy
- No known IP encumbrances

---

### Spacewell

**Core features**
- Integrated workplace management (IWMS)
- Space analytics and workplace reservations
- Maintenance management workflows
- Energy monitoring and sustainability reporting
- Occupancy sensing integration

**Differentiating features**
- Strong occupancy sensing and people analytics
- Integrated workplace space and maintenance in single platform
- European market leader; growing international presence
- Focus on hybrid work optimisation

**UX patterns**
- Dashboard-centric view with occupancy heatmaps
- Space booking interface for employees
- Technician mobile app for maintenance work orders
- Real-time building operations dashboards

**Integration points**
- IoT sensor connectivity for occupancy and environmental monitoring
- Third-party facility data integration
- API for calendar and booking system integration
- Energy management system APIs

**Known gaps**
- Less widely known outside Europe
- Smaller innovation and feature roadmap compared to larger incumbents
- Maintenance capabilities less deep than dedicated CMMS tools

**Licence / IP notes**
- Proprietary SaaS; custom quote pricing
- European-headquartered; growing global presence
- No known IP encumbrances

---

## Cross-Cutting Feature Themes

### Table-Stakes Features

Any facility management platform must include the following baseline capabilities to be viable:

- **Work order management** — creation, assignment, status tracking, and completion reporting
- **Preventive maintenance scheduling** — calendar- or meter-based triggers generating automatic work orders
- **Asset tracking** — inventory of equipment, location, age, and maintenance history
- **Mobile access** — technician app for work order management and documentation from the field
- **Reporting and analytics** — KPIs including work order cycle time, asset reliability, and preventive compliance
- **Multi-location support** — centralised management across multiple facilities and sites
- **Space or maintenance data** — at minimum, tracking of either occupancy/square footage or maintenance assets

### Differentiating Features

Capabilities present in leading solutions that provide competitive advantage:

- **Predictive maintenance** — AI-driven failure prediction using IoT sensor streams and historical work order data
- **Integrated IWMS** — space planning, real estate, and occupancy analytics alongside maintenance management
- **Modern UX and mobile-first design** — faster adoption and lower training burden versus complex enterprise platforms
- **IoT and building automation integration** — automatic trigger of maintenance from sensor anomalies or threshold breaches
- **Occupancy and space analytics** — heat maps, utilisation trends, and move recommendations for real estate optimisation
- **Energy management and sustainability reporting** — compliance with ESG mandates and carbon tracking
- **BIM and CAD integration** — automatic synchronisation of space and asset data from design systems
- **Tenant or occupant experience portal** — self-service issue reporting and space booking to reduce support burden
- **Conversational interfaces** — AI-powered chatbots or natural-language request submission reducing friction

### Underserved Areas / Opportunities

Gaps identified across the solution space that represent genuine opportunities:

- **Truly AI-native architectures** — most platforms layer AI onto rule-based CMMS foundations; none are built bottom-up as AI-first from the start
- **Natural-language work order generation** — occupants report issues in free text; AI must classify, prioritise, route, and draft work instructions automatically
- **Cross-portfolio space optimisation** — ML could analyse badge, calendar, and sensor data across entire real estate portfolio to recommend reconfigurations
- **Real-time energy anomaly detection** — most platforms batch-process energy data; AI could identify consumption outliers in real time across building portfolios
- **Predictive occupancy forecasting** — hybrid work patterns unpredictable; AI could forecast space demand weeks ahead enabling proactive reconfigurations
- **Conversational space booking and building services** — no platform offers natural-language conversational interface for occupants to book, report issues, and get service status via Teams/Slack
- **Integration with occupant wellbeing data** — platforms track occupancy but not comfort, air quality, or noise; opportunities to predict and optimise for wellbeing
- **Autonomous technician routing and dispatch** — most platforms require manual assignment; AI could optimise multi-technician, multi-day scheduling globally

### AI-Augmentation Candidates

Manual or rule-based features in existing tools where AI could meaningfully improve outcomes:

- **Work order triage and prioritisation** — currently manual assessment; NLP could read fault reports and auto-classify severity, urgency, and resource requirements
- **Preventive maintenance schedules** — currently fixed calendar or meter intervals; ML could learn equipment-specific degradation curves and predict optimal replacement windows
- **Failure root-cause analysis** — technicians write narrative notes; ML could identify patterns (e.g., bearing pitting precedes motor failures) and automate diagnosis
- **Technician assignment** — currently rule-based (geography, skills); ML could optimise for travel time, skill match, availability, and learning opportunities
- **Energy consumption forecasting** — currently dashboard heatmaps; ML could predict consumption spikes and automate preventive action (e.g., chiller scheduling)
- **Compliance and audit reporting** — manual compilation of logs; AI could auto-generate audit trails, gap reports, and remediation checklists
- **Occupancy-driven maintenance** — current tools track occupancy separately from maintenance; AI could correlate occupancy patterns with facility failures (e.g., high-occupancy zones overheat)
- **Lease and warranty optimisation** — platforms track leases and assets separately; ML could monitor warranty expiration and lease overlap to optimise renewal timing

## Legal & IP Summary

No copyright, licensing, or patent conflicts were identified during research. All tools analysed are proprietary software with well-defined licensing models (SaaS or on-premise). No open-source solutions were evaluated in depth, as the solutions analysed represent the market leaders. The AI predictive maintenance techniques described are widely adopted across the industry (anomaly detection, failure prediction from sensor baselines) and do not appear to be the subject of active, enforced software patents. Facility management best practices and standards (ISO 41001, IFMA, COBie) are publicly documented and not proprietary. No IP or legal review required for proceeding with development; however, feature-by-feature competitive analysis should be conducted with legal counsel before launch to ensure no patented claims are infringed.

## Recommended Feature Scope

Based on the analysis above, a prioritised feature scope for an AI-native facility management platform:

**Must-have (MVP)**
- Work order management (creation, assignment, tracking, mobile completion)
- Preventive maintenance scheduling with time- and meter-based triggers
- Asset registry and lifecycle tracking
- Technician mobile app with offline capability
- Basic occupant service request portal (issue reporting)
- Real-time work order analytics dashboard

**Should-have (v1.1)**
- AI-powered work order triage and auto-classification from natural language
- Predictive maintenance using historical work order data and equipment age
- Space and occupancy analytics with utilisation heatmaps
- IoT sensor integration (temperature, humidity, occupancy)
- Energy management dashboard with consumption tracking
- Conversational AI assistant for occupant service requests (Teams/Slack integration)

**Nice-to-have (backlog)**
- BIM/CAD integration for automatic space data synchronisation
- Cross-portfolio space optimisation recommendations
- Real-time energy anomaly detection and root-cause analysis
- Occupancy forecasting for hybrid work planning
- Autonomous technician dispatch and route optimisation
- Lease and warranty renewal alerting
- Sustainability and ESG reporting automation
- Tenant wellbeing analytics (comfort, air quality, noise correlation)
