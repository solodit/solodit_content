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
    - logo_256_256.png
        This logo will be used in the search list and the finding detail page.
        - type: png
        - size: 256px * 256px
        - background: transparent

        - sample: [Cyfrin square logo](./reports/Cyfrin/logo_256_256.png)
          
        ![Cyfrin_square](https://github.com/solodit/solodit_content/assets/129466917/8cfc4396-f309-44df-8a7b-a2bfe7a869b4)
      
    - logo_450_225.png
        This logo will be displayed on the Solodit landing page.
        - type: png
        - size: 450px * 225px
        - Put the brand name at the right side of the logo.
        - background color: #292634
        - logo color: #a4a4a4
        - name text color: #ffffff

        - sample: [Cyfrin horizontal logo](./reports/Cyfrin/logo_450_225.png)
     
        ![Cyfrin_horizontal (1)](https://github.com/solodit/solodit_content/assets/129466917/826f2646-2bb5-4bba-ad6f-70f448726663)
      
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
    │        ├── logo_450_225.png              # Horizontal logo image
    │        ├── {Date}-{Protocol}.md          # Report file.(e.g. `2023-06-01-sudoswap.md`)
    │        └── ...
    │   └── ...
    └── ...

✍️ Contributions Welcome
