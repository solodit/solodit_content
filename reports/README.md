# Contribute Add New Reports

### To add a new audit firm, follow these steps:
- Create a new subfolder under the "reports" folder, and name it after the audit firm or solo auditor.
- Include two logo images specifically designed for Solodit.
    - logo_256_256.png
        This logo will be used in the search list and the finding detail page.
        - type: png
        - size: 256px * 256px
        - background: transparent

        - sample: [Cyfrin square logo](./reports/Cyfrin/logo_256_256.png)

        ![Cyfrin_square](https://github.com/solodit/solodit_content/assets/129466917/8cfc4396-f309-44df-8a7b-a2bfe7a869b4)

### To add a report from an existing audit firm, please follow these steps:
- Properly name the report file (`{Date}-{Protocol}.md`) and place it in the appropriate folder.
- Add the auditor information at the beginning.
- `Finding` contents starts with `# Findings`.
- Vulnerabilities are supposed to be classified in 5 categories as below:
    - High Risk
    - Medium Risk
    - Low Risk
    - Gas Optimizations
    - Informational

Ensure that the report is correctly formatted. You can refer to the [Cyfrin reports](./reports/Cyfrin) as an example.

```
**Auditors**

[Giovanni Di Siena](https://twitter.com/giovannidisiena)

[Hans](https://twitter.com/hansfriese)

# Findings

## High Risk
### [Title of Finding-1]
[Content of Finding-1]

### [Title of Finding-2]
[Content of Finding-2]

........

## Medium Risk

........

## Low Risk

........

## Gas Optimizations

........

## Informational

........

```

Please refer to the chart below regarding the path structure.

    .
    ├── ...
    ├── reports                                # Reports folder
    │   ├── Audit firm name                    # Root folder of your reports.
    │        ├── logo_256_256.png              # Square logo image
    │        ├── {Date}-{Protocol}.md          # Report file.(e.g. `2023-06-01-sudoswap.md`)
    │        └── ...
    │   └── ...
    └── ...

### Automation of the Process
We can help you build an automation process to import your reports from your website or Github repo. Please contact us at support@solodit.xyz.
