# Standards & API Reference

> Project: Facility Management Platform · Generated: 2026-05-03

## Industry Standards & Specifications

### ISO Standards

**ISO 41001:2018 — Facility Management: Management Systems — Requirements with Guidance for Use**
- URL: https://www.iso.org/standard/68021.html
- The primary international standard for facility management systems. Specifies requirements for an FM management system that supports the objectives of the demand organisation, consistently meets the needs of interested parties, and achieves sustainable delivery. A revision (ISO/CD 41001) is in progress. Any compliant FM platform should support the governance and reporting workflows this standard mandates.

**ISO 55000:2024 — Asset Management: Vocabulary, Overview and Principles**
- URL: https://www.iso.org/standard/83053.html
- Foundation of the ISO 55000 family (with ISO 55001 and ISO 55002). Provides the vocabulary, overview, and principles for physical asset lifecycle management covering acquisition, operation, maintenance, and decommissioning. Directly applicable to the CMMS and asset-tracking modules of any FM platform; asset data models should align with the lifecycle stages defined here.

**ISO 55001:2014 — Asset Management: Management Systems — Requirements**
- URL: https://www.iso.org/standard/55089.html
- Specifies requirements for establishing and operating an asset management system. Complementary to ISO 41001; together they define the governance and operational frameworks an enterprise FM platform must support.

**ISO 50001:2018 — Energy Management Systems — Requirements with Guidance for Use**
- URL: https://www.iso.org/standard/69426.html
- Specifies requirements for establishing, implementing, maintaining, and improving an energy management system. Applicable to any FM platform offering energy monitoring dashboards and sustainability reporting. A 2024 amendment (Amd 1:2024) adds climate-action changes.

**ISO 16739-1:2024 — Industry Foundation Classes (IFC) for Data Sharing in the Construction and Facility Management Industries**
- URL: https://www.iso.org/standard/84123.html
- Defines the IFC4.3 data schema for sharing building information across the construction and FM industries. The schema underpins BIM-to-FM data handover (asset registers, space layouts, equipment). Any platform offering BIM integration should consume and export IFC4.3 files.

**ISO 16484-5 / ISO 16484-6 (BACnet)**
- URL: https://www.ashrae.org/technical-resources/bookstore/bacnet
- The international standardisation of BACnet (Building Automation and Control Networks), originally ANSI/ASHRAE 135. Defines data communication protocols for HVAC, lighting, access control, elevators, and fire detection. FM platforms integrating with building management systems (BMS) must support BACnet read/write operations.

### W3C & IETF Standards

**RFC 7231 — HTTP/1.1 Semantics and Content**
- URL: https://www.rfc-editor.org/rfc/rfc7231
- Defines the semantics of HTTP methods, status codes, and headers used by all REST APIs in this domain. All FM platform REST APIs should conform to RFC 7231 request/response semantics.

**RFC 6749 — OAuth 2.0 Authorization Framework**
- URL: https://www.rfc-editor.org/rfc/rfc6749
- The standard authorisation framework used by every major FM platform API (Maximo, Planon, FM:Systems, Facilio). An AI-native FM platform should support both the Authorization Code flow (user-facing apps) and the Client Credentials flow (machine-to-machine integrations).

**RFC 7519 — JSON Web Tokens (JWT)**
- URL: https://www.rfc-editor.org/rfc/rfc7519
- JWT is used as the access-token format across FM API ecosystems for encoding identity, permissions, and expiry. Required for secure, stateless API authentication.

**RFC 9110 — HTTP Semantics (replaces RFC 7231)**
- URL: https://www.rfc-editor.org/rfc/rfc9110
- The 2022 update to HTTP semantics that supersedes RFC 7231. REST API design for the FM platform should comply with RFC 9110.

### Data Model & API Specifications

**OpenAPI 3.1 Specification**
- URL: https://spec.openapis.org/oas/v3.1.0
- The dominant standard for describing REST APIs. Both Planon and Facilio expose OpenAPI documents for their REST APIs. An AI-native FM platform should publish a versioned OpenAPI 3.1 spec so integrators can generate clients in any language.

**COBie (Construction Operations Building Information Exchange) — NBIMS-US v3**
- URL: https://nibs.org/nbims/v3/cobie/
- A Model View Definition (MVD) of the IFC schema standardising asset and space data handover from construction to FM operations. COBie defines 19 spreadsheet worksheets (Facility, Floor, Space, Zone, Type, Component, System, Assembly, Connection, Spare, Resource, Job, PM Schedule, Contact, Document, Attribute, Coordinate, Issue, and Impact). FM platforms that ingest COBie files can bootstrap their asset registers directly from construction project data.

**OmniClass / UniFormat — CSI Classification Systems**
- URL: https://www.wbdg.org/resources/omniclass
- OmniClass is a taxonomy for the built environment based on ISO 12006-2. Table 21 (Elements, based on UniFormat) and Table 22 (Work Results, based on MasterFormat) are the most relevant for FM asset and work-order classification. Adopting OmniClass codes in asset and work-order records enables cross-platform data exchange and reporting consistency.

**gbXML (Green Building XML) Schema**
- URL: https://www.gbxml.org/
- An open, de-facto standard schema for transferring building geometry and construction data between BIM authoring tools and energy analysis software. Supported by Autodesk, Trimble, Graphisoft, and Bentley. Relevant for the energy management module of an FM platform where building geometry informs simulation and benchmarking.

**Project Haystack 4 — Semantic Tagging for Smart Building IoT Data**
- URL: https://www.project-haystack.org/
- An open-source semantic modelling standard that applies consistent tags to IoT data points from HVAC, lighting, and occupancy sensors. Haystack 4 adds ontology and taxonomy layers enabling machine-readable equipment relationships, inferred data flows, and automated generation of dashboards and analytics. FM platforms consuming IoT data should support Haystack tag conventions to reduce integration complexity.

**MQTT Protocol (OASIS Standard)**
- URL: https://mqtt.org/
- MQTT (Message Queuing Telemetry Transport) is a lightweight publish-subscribe messaging protocol standard (OASIS MQTT v5.0) widely used for building IoT sensor data ingestion. FM platforms with IoT capabilities (occupancy, HVAC, energy sensors) must expose or consume an MQTT broker. Typically integrated with BAS and edge-device gateways.

### Security & Authentication Standards

**OAuth 2.0 (RFC 6749) / OpenID Connect 1.0**
- URL: https://openid.net/specs/openid-connect-core-1_0.html
- OpenID Connect extends OAuth 2.0 to provide identity federation (single sign-on). All major FM platforms support SSO via OIDC. An AI-native FM platform targeting enterprise clients must support OIDC for integration with Microsoft Entra ID, Okta, and similar identity providers.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x11-t10/
- OWASP's ranked list of the most critical API security risks, including broken object-level authorisation, excessive data exposure, and security misconfiguration. FM platforms handle sensitive occupancy, maintenance, and real-estate data; API design must address the OWASP Top 10.

**NIST Cybersecurity Framework (CSF) 2.0**
- URL: https://www.nist.gov/cyberframework
- The widely adopted US framework for managing cybersecurity risk across Identify, Protect, Detect, Respond, and Recover functions. Enterprise FM buyers increasingly require CSF alignment from software vendors, particularly for platforms that connect to building control systems.

### Energy Benchmarking & Sustainability

**ANSI/ASHRAE 135-2024 — BACnet Data Communication Protocol**
- URL: https://store.accuristech.com/standards/ashrae-135-2024
- The current version of the BACnet standard, maintained by ASHRAE SSPC 135. Specifies object-oriented data communication for building automation covering HVAC, lighting, access control, and elevators. FM platforms integrating with BMS controllers use BACnet/IP or BACnet MS/TP.

**BOMA 2017 — Standard Methods of Measurement (ANSI/BOMA Z65.1-2017)**
- URL: https://boma.org/boma-standards/
- The global standard for measuring gross, useable, and rentable floor area in commercial buildings. FM platforms offering space management and lease administration must support BOMA 2017 area calculations to align with landlord/tenant reporting expectations.

**IFMA Competency Framework & ISO/TC 267 FM Standards**
- URL: https://www.ifma.org/
- IFMA's globally validated competency framework (11 competency areas including Operations & Maintenance, Real Estate, Technology, Environmental Stewardship) serves as the functional specification for the scope of features an FM platform should cover. IFMA also collaborates with ISO/TC 267 on developing international FM standards.

---

## Similar Products — Developer Documentation & APIs

### IBM Maximo (Maximo Application Suite)
- **Description:** Enterprise asset management and CMMS platform covering work orders, preventive maintenance, and IoT asset monitoring. Market leader for large industrial and commercial deployments.
- **API Documentation:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/
- **Official IBM Docs (MAS):** https://www.ibm.com/docs/en/mio?topic=api-data-rest-developers-guide
- **IBM API Hub:** https://developer.ibm.com/apis/catalog/maximo--maximo-manage-rest-api/
- **SDKs/Libraries:** REST/JSON; Python and Java client examples available via IBM API Hub; Automation Script framework for server-side extensions
- **Developer Guide:** https://ibm-maximo-dev.github.io/maximo-restapi-documentation/overview/overview/
- **Standards:** REST/JSON; supports bulk operations via POST with `x-method-override: BULK`; integrated with Maximo Integration Framework (MIF) for enterprise message exchange
- **Authentication:** API key or session token (MAXAUTH header); OAuth 2.0 supported in MAS releases

### Planon IWMS
- **Description:** Enterprise IWMS for space, maintenance, real estate, and energy management with IoT and BIM connectivity. Strong European market presence.
- **API Documentation:** https://webhelp.planoncloud.com/en/REST%20API/c_about_the_planon_rest_api.html
- **OpenAPI Endpoint:** https://webhelp.planoncloud.com/en/REST%20API/c_endpoint_openapi.html
- **SDKs/Libraries:** Python examples on GitHub — https://github.com/planon-community/planon-rest-api-examples; community asset example — https://github.com/planon-community/planon-restapi-assetexample
- **Developer Guide:** https://webhelp.planoncloud.com/en/REST%20API/REST%20API_2.pdf
- **Standards:** REST/JSON; documented via OpenAPI (Swagger-compatible); POST-only API using CRUD Business Object Methods (BOMs)
- **Authentication:** Session token; OAuth 2.0 via OIDC for enterprise SSO

### ServiceNow Workplace Service Delivery
- **Description:** IT-native platform extended to workplace service requests, desk/room booking, visitor management, and maintenance scheduling. Unified IT + FM ticketing.
- **API Documentation:** https://www.servicenow.com/docs/bundle/zurich-api-reference/page/build/applications/concept/api-rest.html
- **REST API Explorer:** Available within each ServiceNow instance at `<instance>.service-now.com/api_explorer`
- **SDKs/Libraries:** ServiceNow SDK for JavaScript (server-side Glide API); community .NET SDK — https://github.com/panoramicdata/ServiceNow.Api
- **Developer Guide:** https://developer.servicenow.com/dev.do
- **Standards:** REST/JSON; Table API for CRUD on any table; Scripted REST APIs for custom endpoints; supports GraphQL queries
- **Authentication:** Basic auth, API key, or OAuth 2.0 (Client Credentials and Authorization Code flows)

### Facilio
- **Description:** Modern IoT-connected CMMS and facility operations platform with predictive maintenance, tenant experience, and sustainability modules. IoT-first architecture.
- **API Documentation:** https://facilio.com/developers/docs/api-reference/
- **Developer Hub:** https://facilio.com/developers/
- **SDKs/Libraries:** JavaScript Apps SDK for embedded connected apps; REST API and MQTT API for IoT data ingestion
- **Developer Guide:** https://facilio.in/guide/article-categories/facilio-api-guide/
- **Standards:** REST/JSON; MQTT for IoT sensor data push; API key authentication; OpenAPI documentation available via developer portal
- **Authentication:** API key passed as `x-api-key` header for IoT data; OAuth 2.0 for user-facing integrations

### UpKeep CMMS
- **Description:** Mobile-first CMMS for maintenance teams managing work orders, assets, and preventive maintenance. Ranked #1 CMMS by Capterra and Gartner.
- **API Documentation:** https://developers.onupkeep.com/
- **REST API Integration Page:** https://upkeep.com/integrations/rest-api/
- **SDKs/Libraries:** REST/JSON API; Zapier integration for no-code workflows; webhook support
- **Developer Guide:** https://help.onupkeep.com/en/articles/2477562-how-do-i-use-upkeep-s-public-apis
- **Standards:** REST/JSON; standard CRUD endpoints for Work Orders, Assets, Parts, and Locations
- **Authentication:** Session token obtained via login endpoint; Enterprise Plan required for API access

### Fiix CMMS (Rockwell Automation)
- **Description:** Cloud CMMS for maintenance planning, work orders, parts inventory, and condition-based maintenance. Over 4,100 maintenance teams; free tier available.
- **API Documentation:** https://fiixlabs.github.io/api-documentation/
- **Developer Guide:** https://fiixlabs.github.io/api-documentation/guide.html
- **SDKs/Libraries:** Java examples — https://github.com/fiixlabs/fiix-cmms-api-java-examples; Python and JavaScript examples available via fiixlabs GitHub
- **Standards:** REST/JSON; supports CRUD and Remote Procedure Calls; v2.48.1 reference
- **Authentication:** API key; session-based authentication for user-context calls

### FM:Systems (FMS:Connect)
- **Description:** Space and workplace management platform with occupancy analytics, move management, and IWMS features. Manages 3+ billion sq ft for 1400+ organisations.
- **API Documentation:** https://www.fmsdocumentation.com/apis/
- **FMS:Connect Overview:** https://amsworkplace.com/fmsconnect/
- **SDKs/Libraries:** REST/JSON bidirectional API; integrations with Workday, ServiceNow, and occupancy sensor vendors (Xovis)
- **Developer Guide:** Contact FM:Systems partner channel for full Swagger specification
- **Standards:** REST/JSON; OAuth 2.0 for authentication; JSON data format with UTF-8 encoding
- **Authentication:** OAuth 2.0 (Client Credentials); API key issued per web user for legacy endpoints

### ENERGY STAR Portfolio Manager (US EPA)
- **Description:** EPA's national building energy benchmarking platform covering over 300,000 properties. Provides ENERGY STAR scores and source EUI metrics widely required for sustainability reporting.
- **API Documentation:** https://portfoliomanager.energystar.gov/webservices/home/
- **Web Services Overview:** https://www.energystar.gov/buildings/resources_audience/utilities_program_sponsors/pm_web_servs
- **SDKs/Libraries:** REST/XML web services; no official SDK; integration via REST calls with XML payloads
- **Developer Guide:** https://www.energystar.gov/buildings/resources-audience/service-product-providers/existing-buildings/benchmarking-clients/use-pm-web-services
- **Standards:** REST/XML; HTTPS required; supports property, meter, and energy data exchange; returns ENERGY STAR score and EUI metrics
- **Authentication:** HTTP Basic Auth with Portfolio Manager account credentials; test environment available for development

---

## Notes

### Emerging & Evolving Standards

- **Brick Schema** (https://brickschema.org/) is an emerging open-source ontology for describing building systems (HVAC, lighting, electrical) in a machine-readable graph format. It is complementary to Project Haystack and increasingly referenced alongside IFC for semantic interoperability. Not yet an ISO standard but gaining industry traction.
- **Digital Twin standards** — ISO/IEC 30173 (Digital twin — Concepts and terminology) and the DTDL (Digital Twins Definition Language, used in Microsoft Azure Digital Twins) are emerging reference points for FM platforms that offer building digital twin capabilities.
- **ASHRAE 223P** (Proposed standard for semantic tagging of HVAC equipment using Brick/Haystack) is under development and may converge the Brick, Haystack, and BACnet ecosystems into a unified tagging specification.
- The **Model Context Protocol (MCP)** developed by Anthropic may become relevant as FM platforms expose AI-callable tool surfaces; an MCP server wrapping FM work-order and asset APIs would allow AI agents to query and update FM data in natural language. No FM platform currently publishes an MCP server, representing an early-mover opportunity.

### Gaps in Available Documentation

- **Archibus** and **IBM TRIRIGA** do not publish open developer documentation; API access requires a vendor contract and NDA. Integration documentation is available only to licensed customers.
- **Spacewell** REST API documentation is not publicly indexed; developer documentation is available via partner portal only.
- The MQTT topic naming conventions used by building automation vendors are not standardised; each vendor (Schneider EcoStruxure, Siemens Desigo, Honeywell) uses proprietary topic hierarchies, creating integration friction. Project Haystack's SkySpark connector and Brick Schema may bridge this gap over the next 2–3 years.
