# Solodit Content Repository
This repository functions as a content hub for the Solodit platform.
We strongly believe in the open-source ethos and value contributions significantly.

## [Categories of Protocol](./protocol_categories.md)
The foundational categories are derived from [DefiLlama](https://defillama.com/categories).

## [Fork information for Protocols](./forked_protocols.md)
The legacy protocols are derived from [DefiLlama](https://defillama.com/forks).

## [Tags for Reports](./report_tags.md)
Most of the initial tags have been manually appended by [Hans](https://github.com/hans-cyfrin).

## [Contribute Add New Reports](./reports)

### To add a new audit firm, follow these steps:
- Create a new subfolder under the "reports" folder, and name it after the audit firm or solo auditor.
- Include two logo images specifically designed for Solodit.

### To add a report from an existing audit firm, please follow these steps:
- Ensure that the report is correctly formatted. You can refer to the [Cyfrin reports](./reports/Cyfrin) as an example.
- Properly name the report file (`{Date}-{Protocol}.md`) and place it in the appropriate folder.

Please refer to the chart below for further guidance.

    .
    ├── ...
    ├── reports                                # Reports folder
    │   ├── Audit firm name                    # Root folder of your reports.
    │        ├── logo_256_256.png              # Square logo image
    │        ├── logo_450_225.png              # Horizontal logo image
    │        ├── {Date}-{Protocol}.md          # Report file.(e.g. `2023-06-01-sudoswap.md`)
    │        └── ...
    │   └── ...
    └── ...

✍️ Contributions Welcome
