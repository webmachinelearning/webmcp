# Security Guidance

## Sensitive Material Handling

- Do not commit secrets, passwords, API keys, tokens, certificates, private
  keys, connection strings, or authentication artifacts.
- Do not commit production customer data, employee personal data, regulated
  data, or incident details that are not explicitly approved for repository
  storage.
- Use redaction and sanitization before adding evidence samples. Remove or mask
  account names, email addresses, hostnames, IP addresses, tenant identifiers, ticket links, and any data that is not required to demonstrate the point.
- Prefer representative templates or scrubbed examples over live artifacts.

## Evidence And Audit Integrity

- Never fabricate evidence, screenshots, approvals, test records, or audit
  outcomes.
- Never alter evidence in a way that changes its substantive meaning.
- If a sample is redacted, note that it is sanitized or representative.
- Preserve references to owners, approvers, dates, and source systems when they
  are needed for audit traceability.

## Reporting Security Issues

-The World Wide Web Consortium (W3C) recognizes that there may be some security issues in W3C standards and specifications. W3C appreciates researchers' feedback on our work because it is critical to keeping the Web secure. If you have found a security issue in a W3C specification, your help is very important.
-If you believe that you have spotted a security bug in a specification, we invite you to follow the [Standards Vulnerability Disclosure & Handling Process and Policy](https://w3c.github.io/security-disclosure/)
**It is important to note that reports of this type are about the standards themselves, not their implementations. It is possible, for example, to propose updates to Security Consideration Sections.**
-To report a security issue in W3C website, please refer to the [W3C security.txt file](https://w3.org/security.txt)