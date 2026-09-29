# Technical Architecture

## Authentication and Authorization

### 1. Overview
The MLB solution should follow a layered authentication model that separates human user access from system-to-system integration access. This reduces risk, preserves least-privilege access, and makes security reviews easier to manage across the Salesforce platform and connected systems.

### 2. Identity Model
- Human users authenticate through the Salesforce Identity layer using company-managed single sign-on (SSO) where available.
- Multi-factor authentication (MFA) is enforced for all interactive users to meet enterprise security requirements.
- External/internal service integrations use dedicated non-human accounts with restricted permissions and rotation policies.
- Access is granted through permission sets and profiles rather than broad org-wide admin access.

### 3. Authentication Patterns
#### User Authentication
- Preferred pattern: SAML or OpenID Connect based SSO with the identity provider used by the organization.
- Salesforce users should not rely on shared credentials for regular access.
- Session timeout, IP restrictions, and MFA policies should be configured centrally and reviewed periodically.

#### Integration Authentication
- API integrations use OAuth 2.0 or a secure named credential pattern instead of embedded credentials in Apex or configuration files.
- External services should use client credentials, JWT, or refresh-token flows depending on the integration type and provider policy.
- Named credentials and external credentials should be used where possible for outbound integrations to keep secrets out of source code.

### 4. Authorization Model
- Roles define broad access boundaries for data visibility and business operations.
- Permission sets grant specific functional rights such as read/write access to records, API usage, or automation execution.
- Apex classes and custom UI components must enforce object-level and field-level access checks using the platform security model.
- No automation should assume elevated access without validating the current user context and permitted actions.

### 5. Platform Security Controls
- Use custom metadata or protected configuration values for non-secret settings only.
- Store secrets in secure platform-managed storage, not in Apex source, static resources, or plain metadata files.
- Restrict Apex classes that access external systems to only the integration users or service principals required for the function.
- Review org-wide defaults, sharing rules, and role hierarchy to ensure data visibility remains consistent with business needs.

### 6. Secure API Design
- All outbound connections should use TLS and validate certificates as required by the target system.
- Avoid exposing sensitive tokens in logs, debug statements, or exception messages.
- Use a dedicated integration user or OAuth app per external system to isolate risk and simplify revocation.
- Log authentication failures and access anomalies without leaking private tokens or payload contents.

### 7. Governance and Operations
- Rotate integration secrets on a defined schedule and immediately after suspected compromise.
- Regularly review user access, permission assignments, and connected app authorizations.
- Maintain a clear list of who can access which external integrations and why.
- Include authentication and authorization checks in release validation and security review activities before deployment.

### 8. Recommended Baseline for MLB
For this project, the recommended baseline is:
- SSO enforced for all human users
- MFA required for all interactive logins
- Named credentials for outbound system integrations
- Dedicated integration users with least-privilege permissions
- Permission sets rather than ad hoc profile changes
- Centralized review of access and secret rotation before production rollout

This design gives the organization a secure, auditable, and scalable auth model while staying aligned with Salesforce best practices and enterprise governance standards.
