# config.business.lua

Business banking (company accounts + TPE terminal item).

File: `shared/config.business.lua`

## Business

```lua
Config.Business = {
    Enabled = true,
    CreationCost = 50000,           -- Cost to create a business
    MaxBusinessesPerPlayer = 1,
    TPECost = 5000,                 -- Cost of a business-linked TPE
    WithdrawFee = 0,                -- 0 = free, 0.05 = 5%
    AccessLevel = 'boss',           -- 'boss' | 'manager' | 'employee' (min role to open the business dashboard)
    NpcModel = 'a_m_y_business_03',
    NpcLocations = {
        { coords = vector4(254.1678, 222.6417, 106.2868, 158.0187), label = 'Business Bank' },
    },
    Blips = {
        Enabled = true,
        Sprite = 431,
        Color = 46,
        Scale = 0.7,
        Display = 4,
        ShortRange = true,
        Label = 'Business Bank',
    },
    OxTarget = {
        Icon = 'fas fa-briefcase',
        Label = 'Business bank',
        Distance = 2.5,
    },
}
```

### Role hierarchy

When a player opens the business dashboard the bridge compares their role to `AccessLevel`:

| Role | Level |
|---|---|
| `boss` | 3 |
| `manager` | 2 |
| `employee` | 1 |

With `AccessLevel = 'manager'`, both bosses and managers can open the dashboard; employees cannot.

## TPE

```lua
Config.TPE = {
    RequirePhysicalCard = false,     -- Force physical card item in inventory for physical-format cards

    PlayerToPlayer = {
        Enabled = true,
        Item = 'nash_tpe',
        MaxAmount = 50000,
        Fee = 0.02,                  -- 2% fee taken from the seller's receipt
        Distance = 3.0,              -- Max distance to the buyer (meters)
    },

    -- Optional server-side hooks to route TPE income to an external business system.
    -- Both default to nil, in which case Nash Banking credits the receipt internally
    -- (nash_businesses.balance for business TPEs, seller's personal bank account otherwise).
    BusinessDepositHook = nil,
    EmployeeDepositHook = nil,
}
```

### External business system integration

By default, when a TPE payment is accepted, Nash Banking credits the money either to `nash_businesses.balance` (when the TPE was bought through the built-in business panel) or to the seller's personal bank account. If your server manages businesses through an external script (`esx_society`, `qb-management`, a custom system, ...) you can plug into these two hooks to route the money to your own accounts.

Both hooks are called server-side, right after a TPE payment is validated. Return `true` from the hook to tell Nash Banking that your code handled the credit, in which case Nash Banking skips both its internal balance credit and its transaction log. Return `false`, `nil`, or throw and the default behavior applies unchanged.

**`BusinessDepositHook`** fires when the TPE used was purchased through Nash Banking's business panel. Useful if you also want the money from these payments to end up in your external system instead of `nash_businesses.balance`.

```lua
Config.TPE.BusinessDepositHook = function(source, xPlayer, amount, ctx)
    -- ctx = { businessId, businessName, jobName, buyerName, buyerIdentifier, description }
    if ctx.jobName then
        exports['my_business_script']:AddSocietyMoney(ctx.jobName, amount)
        return true
    end
    return false
end
```

**`EmployeeDepositHook`** fires when the seller used a generic TPE item (no Nash Banking business behind it). This is the hook to use if you don't use Nash Banking's business panel at all and just want TPE payments to be routed by job.

```lua
Config.TPE.EmployeeDepositHook = function(source, xPlayer, amount, ctx)
    -- ctx = { jobName, buyerName, buyerIdentifier, description }
    if ctx.jobName and ctx.jobName ~= 'unemployed' then
        exports['my_business_script']:AddSocietyMoney(ctx.jobName, amount)
        return true
    end
    return false
end
```

When a hook returns `true`, your code is fully responsible for crediting the money and (if you want traceability) logging the transaction in your own system. Nash Banking will not write to `nash_transactions` or `nash_business_transactions` for that payment. Notifications to the buyer and the seller, and the Discord log, are still emitted regardless.

See [Guides › Create a business](../guides/create-business.md) and [Guides › Integrate TPE](../guides/integrate-tpe.md).
