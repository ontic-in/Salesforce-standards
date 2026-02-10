# Salesforce Development Standards & Best Practices

This documentation serves as the authoritative guide for all Salesforce development projects. These standards ensure consistency, maintainability, and quality across all implementations.

## Document Structure

```
salesforce-standards/
├── README.md                           # This file - Index and overview
├── standards/                          # Mandatory implementation standards
│   └── error-handling.md               # STD-001: Error Handling Standard
└── best-practices/                     # Recommended practices and guidelines
    └── error-messages.md               # Best practices for error messaging
```

## Standards Index

| ID | Standard | Description | Status |
|----|----------|-------------|--------|
| STD-001 | [Error Handling](standards/error-handling.md) | Centralized error logging and handling pattern | Active |

## Best Practices Index

| Topic | Document | Description |
|-------|----------|-------------|
| Error Messages | [Error Messages](best-practices/error-messages.md) | Creating readable, actionable error messages |

## Compliance Requirements

All new Salesforce projects MUST implement:
1. **STD-001**: Error Handling Standard - Deploy the Error_Log__c custom object and implement error logging in all automations

## Version History

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-02-04 | - | Initial release with Error Handling Standard |

## Contributing

When adding new standards or best practices:
1. Create the document in the appropriate folder (`standards/` or `best-practices/`)
2. Use the established template format
3. Update this README with the new entry
4. Standards require ID assignment (STD-XXX format)
