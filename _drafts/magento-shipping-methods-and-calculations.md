---
layout: post
title:  "Magento Shipping Methods and Calculations"
categories: magento
---

- `Mage_Sales_Model_Quote::collectTotals()`
    - `Mage_Sales_Model_Quote_Address::collectTotals()`
        - `Mage_Sales_Model_Quote_Address_Total_Collector` collects total models from `global/sales/quote/totals`, `collect()` is called on each
        - The model to collect shipping totals is `sales/quote_address_total_shipping`, which is `Mage_Sales_Model_Quote_Address_Total_Shipping`
        - This model calculates the free shipping quantity and weight before and sets these values on the address/items
        - `\Mage_Sales_Model_Quote_Address::collectShippingRates()` is called, which removes shipping rates and then:
            - Calls `\Mage_Sales_Model_Quote_Address::requestShippingRates()`, which creates the rate request (`Mage_Shipping_Model_Rate_Request`) first
                - `\Mage_Shipping_Model_Shipping::collectRates()` is called
                    - Which calls `collectCarrierRates()`, this calls `composePackagesForCarrier()` to split packages, then assigns weights to each, then:
                        - `collectRates()` is called on the shipping method, which returns a `Mage_Shipping_Model_Rate_Result` with `Mage_Shipping_Model_Rate_Result_Method` composed on to it
                    - The handling fee is added to each rate (I think, line 198)
                    - The rates are sorted by price
            - Back in `\Mage_Sales_Model_Quote_Address::requestShippingRates()`, each rate rate result method is saved to the `sales_flat_quote_shipping_rate` table
            - Then the rate selected is set as the shipping price
        - Back
    - Back
- The quote now updates the totals, which will include the shipping

So this is the general architecture:

- Total collection is triggered
- A rate request is generated, which has source and destination, and all items, and issued to the shipping method
- A rate result is created, which has one or more rate result methods composed on to it
- The resulting methods are stored in the `sales_flat_quote_shipping_rate` table
- The currently active rates price is added to the order total
