# config.bankinfo.lua

Values shown in the **Bank details** modal of the desktop banking app (the IBAN / BIC / bank name and address block that players see when they open **Bank details**).

None of these values change the script's logic, they are purely display fields. Tweak them to match the country your RP server is set in.

File: `shared/config.bankinfo.lua`

## Full block

```lua
Config.BankInfo = {
    -- 2-letter country code prepended to the displayed IBAN
    -- Ex: 'FR' (France), 'US' (USA), 'GB' (UK), 'DE' (Germany), 'CH' (Switzerland)
    IbanCountry = 'FR',

    -- IBAN body displayed after the country code (check digits + account number)
    -- Rendered as-is in the RIB. Whitespace is preserved for readability.
    -- Ex FR: '76 2823 3000 0114 5658 4261 360'
    -- Ex US: '89 0210 0001 9000 0000 1234 567'
    IbanBody = '76 2823 3000 0114 5658 4261 360',

    -- Bank BIC / SWIFT
    BIC = 'NASHFRP2',

    -- Legal bank name (displayed above the address)
    BankName = 'Nash Banking SA',

    -- Bank headquarters address (displayed under the name)
    BankAddress = '42 avenue des Champs-Élysées, 75008, Paris, France',

    -- Correspondent bank BIC (used for international transfers)
    CorrespondentBIC = 'CHASDEFX',
}
```

## Where it shows

The whole block is sent to the NUI every time the banking is opened. The **Bank details** modal reads it and displays each field as a copy-to-clipboard row. The displayed IBAN is the concatenation `IbanCountry + IbanBody`.

## Ready-made examples

### US server

```lua
Config.BankInfo = {
    IbanCountry = 'US',
    IbanBody = '89 0210 0001 9000 0000 1234 567',
    BIC = 'NASHUS33',
    BankName = 'Nash Banking Inc.',
    BankAddress = '350 5th Avenue, New York, NY 10118, USA',
    CorrespondentBIC = 'CHASUS33',
}
```

### UK server

```lua
Config.BankInfo = {
    IbanCountry = 'GB',
    IbanBody = '29 NASH 6016 1331 9268 19',
    BIC = 'NASHGB2L',
    BankName = 'Nash Banking Ltd.',
    BankAddress = '25 Bank Street, Canary Wharf, London E14 5JP, UK',
    CorrespondentBIC = 'CHASGB2L',
}
```

### German server

```lua
Config.BankInfo = {
    IbanCountry = 'DE',
    IbanBody = '89 3704 0044 0532 0130 00',
    BIC = 'NASHDEFF',
    BankName = 'Nash Banking AG',
    BankAddress = 'Bockenheimer Landstraße 24, 60323 Frankfurt am Main, Germany',
    CorrespondentBIC = 'CHASDEFX',
}
```
