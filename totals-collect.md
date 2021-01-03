# Magento 2 - collect totals
Totals collection is an important process in magento.

It exists for the following object : 
* quote
* order_invoice
* order_creditmemo

We will look in detail the example of the Quote object which is the one you will be more frequently interacting with.

The totals collection is triggered at different times.
For example in Quote Model afterLoad event with following code :
`$quote->collectTotals()`

Who has what ?
* The `Quote` object has a `TotalsCollector`.
* The `TotalsCollector` object has a list of `TotalCollector`.

The `TotalsCollector` is responsible for :
* looping on addresses (in many cases, 2 addresses, one billing and one shipping), see `\Magento\Quote\Model\Quote\TotalsCollector::collect`.
* executing a list of total collectors for each address, see `\Magento\Quote\Model\Quote\TotalsCollector::collectAddressTotals`.

The `TotalsCollector` initiates a global `Total` object (`\Magento\Quote\Model\Quote\Address\Total`).

```
/** @var \Magento\Quote\Model\Quote\Address\Total $total */
$total = $this->totalFactory->create(\Magento\Quote\Model\Quote\Address\Total::class);
```

For each address looped, the `TotalsCollector` will get a new `Total` object that will come to "enrich" the global `Total` object.

This is done by calling the `collect` method on each `TotalCollector`, the resulting `Total` contains the data for the current address.

These are the data :
* `shipping_amount`
* `base_shipping_amount`
* `shipping_description`
* `subtotal`
* `base_subtotal`
* `subtotal_with_discount`
* `base_subtotal_with_discount`
* `grand_total`
* `base_grand_total`

Depending on the data, it can be used to increment the same data on the main `Total` object. This is the case for `subtotal`for example : 

`$total->setSubtotal((float)$total->getSubtotal() + $addressTotal->getSubtotal());`

Or it can simply replace the same data in the main `Total` object. This is the case for `shipping_amount` for example : 

`$total->setShippingAmount($addressTotal->getShippingAmount());`

There is the list of `TotalCollector` object that the `TotalsCollector`natively holds : 
* `Magento\Quote\Model\Quote\Address\Total\Subtotal`
* `Magento\Tax\Model\Sales\Total\Quote\Subtotal`
* `Magento\Weee\Model\Total\Quote\Weee`
* `Magento\SalesRule\Model\Quote\Discount`
* `Magento\Quote\Model\Quote\Address\Total\Shipping`
* `Magento\Tax\Model\Sales\Total\Quote\Shipping`
* `Magento\SalesRule\Model\Quote\Address\Total\ShippingDiscount`
* `Magento\Tax\Model\Sales\Total\Quote\Tax`
* `Magento\Weee\Model\Total\Quote\WeeeTax`
* `Magento\Quote\Model\Quote\Address\Total\Grand`
  
The list of `TotalCollector` is loaded from xml config
See `\Magento\Quote\Model\Quote\Address\Total\Collector::__construct`
and `\Magento\Sales\Model\Config\Ordered::_initCollectors`.


As it can be defined for quote, order_invoice, and order_creditmemo, you will find different section in the XML files (sales.xml).

For order_invoice and order_creditmemo, please see the total collector used by default (see sales.xml file in Magento_Sales module) : 

For order_invoice :
```
	<section name="order_invoice">
        <group name="totals">
            <item name="subtotal" instance="Magento\Sales\Model\Order\Invoice\Total\Subtotal" sort_order="50"/>
            <item name="discount" instance="Magento\Sales\Model\Order\Invoice\Total\Discount" sort_order="100"/>
            <item name="shipping" instance="Magento\Sales\Model\Order\Invoice\Total\Shipping" sort_order="150"/>
            <item name="tax" instance="Magento\Sales\Model\Order\Invoice\Total\Tax" sort_order="200"/>
            <item name="cost_total" instance="Magento\Sales\Model\Order\Invoice\Total\Cost" sort_order="250"/>
            <item name="grand_total" instance="Magento\Sales\Model\Order\Invoice\Total\Grand" sort_order="350"/>
        </group>
    </section>
```

 For order_creditmemo:
 ```
     <section name="order_creditmemo">
        <group name="totals">
            <item name="subtotal" instance="Magento\Sales\Model\Order\Creditmemo\Total\Subtotal" sort_order="50"/>
            <item name="discount" instance="Magento\Sales\Model\Order\Creditmemo\Total\Discount" sort_order="150"/>
            <item name="shipping" instance="Magento\Sales\Model\Order\Creditmemo\Total\Shipping" sort_order="200"/>
            <item name="tax" instance="Magento\Sales\Model\Order\Creditmemo\Total\Tax" sort_order="250"/>
            <item name="cost_total" instance="Magento\Sales\Model\Order\Creditmemo\Total\Cost" sort_order="300"/>
            <item name="grand_total" instance="Magento\Sales\Model\Order\Creditmemo\Total\Grand" sort_order="400"/>
        </group>
    </section>
```

For quote, please see the total collector loaded in Magento_Quote module : 
```
<section name="quote">
        <group name="totals">
            <item name="subtotal" instance="Magento\Quote\Model\Quote\Address\Total\Subtotal" sort_order="100"/>
            <item name="shipping" instance="Magento\Quote\Model\Quote\Address\Total\Shipping" sort_order="350"/>
            <item name="grand_total" instance="Magento\Quote\Model\Quote\Address\Total\Grand" sort_order="550"/>
        </group>
    </section>
```

And then in Magento_SalesRule module :
```
    <section name="quote">
        <group name="totals">
            <item name="discount" instance="Magento\SalesRule\Model\Quote\Discount" sort_order="300"/>
            <item name="shipping_discount" instance="Magento\SalesRule\Model\Quote\Address\Total\ShippingDiscount" sort_order="400"/>
        </group>
    </section>
```

 And then in Magento_Tax module : 
```
 <section name="quote">
        <group name="totals">
            <item name="tax_subtotal" instance="Magento\Tax\Model\Sales\Total\Quote\Subtotal" sort_order="200"/>
            <item name="tax_shipping" instance="Magento\Tax\Model\Sales\Total\Quote\Shipping" sort_order="375"/>
            <item name="tax" instance="Magento\Tax\Model\Sales\Total\Quote\Tax" sort_order="450"/>
        </group>
    </section>
```

And then in Magento_Weee module : 
```
    <section name="quote">
        <group name="totals">
            <item name="weee" instance="Magento\Weee\Model\Total\Quote\Weee" sort_order="225"/>
            <item name="weee_tax" instance="Magento\Weee\Model\Total\Quote\WeeeTax" sort_order="460"/>
        </group>
    </section>
```

I we cumulate these configuration, it will look like that :
```
<section name="quote">
        <group name="totals">
            <item name="subtotal" instance="Magento\Quote\Model\Quote\Address\Total\Subtotal" sort_order="100"/>
            <item name="tax_subtotal" instance="Magento\Tax\Model\Sales\Total\Quote\Subtotal" sort_order="200"/>
            <item name="weee" instance="Magento\Weee\Model\Total\Quote\Weee" sort_order="225"/>
            <item name="discount" instance="Magento\SalesRule\Model\Quote\Discount" sort_order="300"/>
            <item name="shipping" instance="Magento\Quote\Model\Quote\Address\Total\Shipping" sort_order="350"/>
            <item name="tax_shipping" instance="Magento\Tax\Model\Sales\Total\Quote\Shipping" sort_order="375"/>
            <item name="shipping_discount" instance="Magento\SalesRule\Model\Quote\Address\Total\ShippingDiscount" sort_order="400"/>
            <item name="tax" instance="Magento\Tax\Model\Sales\Total\Quote\Tax" sort_order="450"/>
            <item name="weee_tax" instance="Magento\Weee\Model\Total\Quote\WeeeTax" sort_order="460"/>
            <item name="grand_total" instance="Magento\Quote\Model\Quote\Address\Total\Grand" sort_order="550"/>
        </group>
    </section>
```
