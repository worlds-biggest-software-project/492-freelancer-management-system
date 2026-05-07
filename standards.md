# Standards & API Reference

> Project: Freelancer Management System · Generated: 2026-05-07

## Industry Standards & Specifications

### ISO Standards

**ISO 27001:2022 — Information Security Management Systems**
- URL: https://www.iso.org/standard/27001
- The de-facto enterprise security certification required by mid-market and enterprise buyers when procuring contractor management platforms. Covers information asset management, access control, and incident response. Platforms such as Deel and Remote hold ISO 27001 certification as a procurement prerequisite.

**ISO 27701:2019 — Privacy Information Management (Extension to ISO 27001)**
- URL: https://www.iso.org/standard/71670.html
- Extends ISO 27001 with privacy-specific controls for the processing of personal data. Directly relevant to contractor platforms that store sensitive tax data (SSN, TIN, bank details) and must demonstrate GDPR and CCPA alignment through a verifiable Privacy Information Management System (PIMS).

**ISO 20022 — Financial Services Universal Financial Industry Message Scheme**
- URL: https://www.iso20022.org/
- Global XML-based messaging standard for financial transactions. NACHA has published an ISO 20022-to-ACH mapping guide enabling domestic ACH contractor payments to align with ISO 20022 pain.001 (credit) and pain.008 (debit) message formats. Increasingly required for cross-border payment interoperability in global FMS platforms.

**ISO/IEC 18013-5 — Mobile Driving Licence (mDL) and Digital Identity**
- URL: https://www.iso.org/standard/69084.html
- Underpins emerging digital identity verification standards referenced by the EU Digital Identity Wallet (EUDI) initiative. Relevant to portable contractor identity and KYC re-use scenarios where onboarding credentials are verified once and reused across multiple client engagements.

---

### W3C & IETF Standards

**RFC 6749 — The OAuth 2.0 Authorization Framework**
- URL: https://datatracker.ietf.org/doc/html/rfc6749
- The foundational authorisation standard used by all major FMS APIs (Deel, Remote, YunoJuno). Platforms use the Authorization Code flow for user-delegated access and Client Credentials flow for service-to-service API integration. OAuth 2.1 (forthcoming) deprecates Implicit and Password grant types.

**RFC 7519 — JSON Web Token (JWT)**
- URL: https://datatracker.ietf.org/doc/html/rfc7519
- Used for bearer token representation in OAuth 2.0 flows. FMS API access tokens are typically JWTs with 15–60 minute expiry and refresh token rotation for security.

**RFC 7807 — Problem Details for HTTP APIs**
- URL: https://datatracker.ietf.org/doc/html/rfc7807
- Standard error response format for REST APIs. Relevant when designing the FMS public API to return machine-readable error objects distinguishing compliance failures, validation errors, and payment rejections.

**RFC 8288 — Web Linking (Link Header)**
- URL: https://datatracker.ietf.org/doc/html/rfc8288
- Used for pagination in REST APIs returning contractor lists, invoice histories, or payment records. Standard `Link` header with `rel=next`, `rel=prev` values.

**W3C Verifiable Credentials Data Model 2.0 (Recommendation, May 2025)**
- URL: https://www.w3.org/TR/vc-data-model-2.0/
- Cryptographically secure, privacy-respecting credential format. Directly applicable to portable contractor identity: a contractor completes KYC once and receives a verifiable credential reusable across multiple clients without repeating the full onboarding process. Adoption reduces onboarding cost by 30–60% per engagement according to 2026 research.

**W3C Decentralised Identifiers (DID) 1.0**
- URL: https://www.w3.org/TR/did-core/
- Companion standard to Verifiable Credentials enabling self-sovereign contractor identity. Contractors control their own DID-anchored identity record, which clients verify without a centralised intermediary.

**SCIM 2.0 (RFC 7642/7643/7644) — System for Cross-domain Identity Management**
- URL: https://datatracker.ietf.org/doc/html/rfc7644
- Standard protocol for provisioning and de-provisioning contractor identities across SSO systems. Used by Deel and Remote for enterprise identity lifecycle management. Relevant for FMS platforms integrating with Okta, Microsoft Entra ID, and Google Workspace.

---

### Data Model & API Specifications

**OpenAPI Specification 3.2.0 (September 2025)**
- URL: https://spec.openapis.org/oas/v3.2.0.html
- The current stable version of the OpenAPI standard for describing REST APIs. Adds structured tag nesting, streaming media type support (SSE, JSON Lines), native QUERY HTTP method, and OAuth 2.0 Device Authorization Flow. Enterprise buyers in 2026 expect OpenAPI specs as deliverables alongside working API code. All major FMS vendors (Deel, Remote, YunoJuno) publish OpenAPI-compatible documentation.

**JSON Schema Draft 2020-12**
- URL: https://json-schema.org/specification.html
- The JSON data validation standard, now natively compatible with OpenAPI 3.1+. Relevant for defining contractor profile schemas, invoice data models, and payment instruction objects in a portable, toolchain-agnostic format.

**HR Open Standards — Worker Data Model**
- URL: https://www.hropenstandards.org/
- The HR Open Standards Consortium (founded 1999) maintains the only independent, non-profit suite of specifications for HR data exchange. Specifications cover worker onboarding (WorkerOnBoarding), person profiles, payroll, and contingent worker engagement. Adoption enables interoperability with Workday, SAP SuccessFactors, and Oracle HCM integrations.

**NACHA ACH Rules — Operating Rules and Guidelines**
- URL: https://www.nacha.org/content/standards
- The rules governing Automated Clearing House payments for US contractor payouts. Relevant for direct deposit, off-cycle payments, and same-day ACH disbursements. The NACHA ISO 20022-to-ACH Mapping Guide enables international message interoperability.

**IRS IRIS API — Information Returns Intake System**
- URL: https://www.irs.gov/e-file-providers/filing-information-returns-electronically-fire
- The IRS IRIS system (replacing FIRE permanently on December 31, 2026) uses modern XML-based filing with a REST-like API interface for 1099 series submissions. Contractors paid USD 600+ annually require 1099-NEC filing. Integration requires an IRS Transmitter Control Code (TCC) obtained via the IRIS Transmitter Portal.

**IRS TIN Matching Program**
- URL: https://www.irs.gov/e-services/tin-matching-program
- Bulk and real-time TIN (Taxpayer Identification Number) verification service used to validate contractor W-9 data before payment and before 1099 filing. Required by platforms processing large volumes of contractor payments to avoid B-notices and backup withholding obligations.

---

### Security & Authentication Standards

**OAuth 2.0 / OAuth 2.1**
- URL: https://oauth.net/2/
- Standard authentication framework used by all FMS REST APIs. OAuth 2.1 deprecates Implicit and Password grant types. All new FMS API implementations should use Authorization Code with PKCE for user-facing flows and Client Credentials for server-to-server integration.

**OpenID Connect 1.0 (OIDC)**
- URL: https://openid.net/connect/
- Identity layer on top of OAuth 2.0. Used by FMS platforms for SSO login (Okta, Microsoft Entra ID, Google Workspace). Relevant for enterprise contractor platforms where admins authenticate via corporate identity providers.

**OWASP API Security Top 10 (2023)**
- URL: https://owasp.org/API-Security/editions/2023/en/0x00-header/
- The authoritative checklist for API security risks. Critical for FMS APIs handling contractor PII, financial data, and tax documents. Key risks include Broken Object Level Authorisation (BOLA) and Security Misconfiguration, which are acute in multi-tenant contractor platforms.

**GDPR — General Data Protection Regulation (EU 2016/679)**
- URL: https://gdpr.eu/
- Governs the storage and processing of personal data for contractors in EU member states. Requires a legal basis for processing (contract performance, legal obligation), data minimisation, right to erasure, and Data Processing Agreements (DPAs) with sub-processors. FMS platforms acting as data processors must sign DPAs with every client organisation.

**CCPA/CPRA — California Consumer Privacy Act / California Privacy Rights Act**
- URL: https://oag.ca.gov/privacy/ccpa
- US state privacy law applicable to contractor data for California-based contractors or California-based businesses. Requires disclosure of data categories collected, opt-out of data sale, and deletion rights. Platforms serving US contractors must address CCPA alongside IRS compliance requirements.

**ISO/IEC 27701:2019 — Privacy Information Management**
- See ISO Standards section above. Provides the PIMS certification pathway that harmonises GDPR and CCPA obligations into a single auditable framework.

---

### MCP Server Specifications

The Model Context Protocol (MCP) is potentially relevant for AI-native FMS implementations where LLM agents require structured access to contractor records, compliance rules, and payment status. An FMS MCP server could expose:

- **Contractor profile context** — structured access to onboarding status, document checklist, and classification flags for AI classification agents
- **Invoice and payment status** — enabling AI agents to answer contractor queries about payment timing and reconciliation
- **Compliance rule retrieval** — jurisdiction-specific rules surfaced as MCP tool calls for AI-driven contract generation

MCP specification: https://modelcontextprotocol.io/specification

---

## Similar Products — Developer Documentation & APIs

### Deel

- **Description:** Global contractor and EOR platform supporting 200+ countries, handling contracts, payments, compliance, and 1099/tax filing.
- **API Documentation:** https://developer.deel.com/api/contractors/introduction
- **SDKs/Libraries:** REST API; no official SDK; community TypeScript/Python clients exist on GitHub
- **Developer Guide:** https://developer.deel.com/docs/getting-started
- **Standards:** REST/JSON; OpenAPI-compatible documentation; sandbox environment available
- **Authentication:** OAuth 2.0 (API token via Developer Center); org admin or IT Developer Admin role required

### Remote

- **Description:** Global EOR and contractor management platform with owned-entity compliance in 80+ countries, offering a unified REST API for contractor lifecycle management.
- **API Documentation:** https://developer.remote.com/reference/welcome-to-remote-api
- **SDKs/Libraries:** REST API; no official SDK; JSON-only responses
- **Developer Guide:** https://developer.remote.com/docs/quick-start-guide
- **Standards:** REST/JSON; OAuth 2.0; rate limit 20 requests/minute
- **Authentication:** OAuth 2.0 (Authorization Code for users; Client Credentials for service-to-service)

### YunoJuno

- **Description:** Freelancer management system and marketplace; highest-rated Leader in Everest Group PEAK Matrix for FEMS 2026; strong UK IR35 compliance engine.
- **API Documentation:** https://docs.yunojuno.com/welcome
- **SDKs/Libraries:** REST API; GET, POST, PATCH; JSON payloads
- **Developer Guide:** https://docs.yunojuno.com/
- **Standards:** REST/JSON; OpenAPI-compatible
- **Authentication:** API key authentication; OAuth 2.0 for enterprise SSO integrations

### Worksuite

- **Description:** Freelancer management platform with private talent pools, AI misclassification detection, and enterprise ERP/HCM integration services.
- **API Documentation:** https://documenter.getpostman.com/view/994834/SWTBcwoR
- **SDKs/Libraries:** REST API documented via Postman; token-based authentication
- **Developer Guide:** https://worksuite.com/resources/support-services/integrations
- **Standards:** REST/JSON; token-based auth via `/auth/login`
- **Authentication:** Access token (obtained via login endpoint); token-based session management

### Papaya Global

- **Description:** Enterprise global payroll and contractor management platform with a dedicated Payments API for multi-currency workforce disbursements across 180+ countries.
- **API Documentation:** https://docs.papayaglobal.com/
- **SDKs/Libraries:** REST API; JSON; dedicated Payments API
- **Developer Guide:** https://docs.papayaglobal.com/
- **Standards:** REST/JSON; OpenAPI-compatible; ISO 20022 payment messaging alignment
- **Authentication:** OAuth 2.0; API key for Payments API access

### Gusto Embedded Payroll

- **Description:** White-label payroll and contractor payment API enabling HR software partners to embed Gusto's 1099 filing and contractor payment infrastructure.
- **API Documentation:** https://docs.gusto.com/embedded-payroll/docs/contractors-intro
- **SDKs/Libraries:** REST API; official Ruby, Python, and Node.js SDKs
- **Developer Guide:** https://docs.gusto.com/embedded-payroll/docs/getting-started
- **Standards:** REST/JSON; OpenAPI specification published
- **Authentication:** OAuth 2.0 (Authorization Code flow); scoped access tokens per employer

### Wingspan Embed

- **Description:** Embeddable contractor management infrastructure for gig economy platforms, providing white-label onboarding, payments, 1099 filing, and wallet capabilities.
- **API Documentation:** https://docs.wingspan.app/
- **SDKs/Libraries:** REST API; UI component library; SDK for embedded integration
- **Developer Guide:** https://docs.wingspan.app/docs/spend-management-platforms
- **Standards:** REST/JSON; webhook-driven event model
- **Authentication:** API key; OAuth 2.0 for platform operator authentication

### Tax1099 / TaxBandits (IRS e-filing APIs)

- **Description:** Third-party IRS-authorised e-file partners exposing REST APIs for 1099-NEC, W-2, W-9, TIN matching, and ACA form filing. Used by FMS platforms to outsource IRS IRIS/FIRE filing obligations.
- **API Documentation (Tax1099):** https://www.tax1099.com/tax1099-api-integration
- **API Documentation (TaxBandits):** https://developer.taxbandits.com/
- **SDKs/Libraries:** REST API; JSON; sandbox test environments
- **Developer Guide:** Available at both developer portals above
- **Standards:** IRS Publication 1099 (2026); IRIS XML schema; TIN Matching Program
- **Authentication:** API key; OAuth 2.0 for enterprise accounts

---

## Notes

**FIRE system retirement:** The IRS FIRE (Filing Information Returns Electronically) system retires permanently on December 31, 2026. All FMS platforms performing 1099 e-filing must complete migration to the IRIS API before that date. This is a hard deadline with no extensions.

**OAuth 2.1 adoption:** OAuth 2.1 is near-final and deprecates the Implicit and Password grant types. New FMS API implementations should adopt 2.1-compliant patterns (Authorization Code + PKCE) to avoid future breaking changes.

**Verifiable Credentials momentum:** W3C Verifiable Credentials 2.0 (Recommendation, May 2025) and the EU eIDAS 2.0 deadline (December 31, 2026) are accelerating enterprise adoption of reusable digital identity. FMS platforms building contractor identity portability features should align with the VC 2.0 and DID 1.0 standards now to avoid proprietary lock-in.

**HR Open Standards gap:** While the HR Open Standards Consortium provides relevant worker data models, adoption across commercial FMS vendors is limited. The domain lacks a universally adopted contractor data interchange format, representing an opportunity for an open-source platform to define a portable contractor profile schema based on HR Open + JSON Schema.
