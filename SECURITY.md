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

If you identify a security weakness in repository content, proposed changes, or
automation:

- Do not publish sensitive exploit details in a public issue unless the repo
  owner explicitly requests that workflow.
- Notify the designated repository owner, security contact, or maintainers
  through the approved internal reporting path.
- Include enough detail to reproduce and assess the issue without attaching
  secrets or sensitive data.
- If the issue affects documented controls or evidence expectations, update the
  relevant documentation only after maintainers confirm the correct handling.
