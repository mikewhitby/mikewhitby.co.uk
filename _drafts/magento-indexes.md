---
layout: post
title:  "Magento Indexes"
categories: magento
---

Magento stores some of its data in a complex EAV format, which is great for
extensibility, but not great for performance. In order to keep the extensibility
but also realize performance benefits, Magento will perform data de-normalization
on several key peices of data. This involves reading the data from the complex
formats and duplicating it in to simple formats, this is done in its entirety
when requested, or on an item-by-item basis when the source data is updated.


## Index Types

There are 7 index types, as can be seen from System > Index Management, I'll
detail what each does below.


### Product Attributes

This is really compromised of 2 distinct indexes:

- Flat Category
- Flat Product

This indexer takes the values that are in the EAV tables for both of these
entity types and puts it in to simple tables. These two indexes are actually
only ever used for showing the category nav, and for the information on the
category listing pages - the product view page itself and the admin interface
use the source EAV data from the product. For the product data, only attributes
that are marked as  "Used in product listing" are included in the index, in
order to avoid having a huge flat table full of every single attribute, of which
many may well not be needed to display the listing. It would make sense for
Magento to have used the flat data for product pages for stores that have few
attributes and attribute sets, but imagine a huge store with thousands of
attributes - MySQL has a column limit of 4096 columns per table, and in practice
is a lot less - so having such a 'wide' table is not an option, hence Magento
decided to use this index for the listing only.


### Product Prices

Pricing in Magento is based on various factors, the elements that can have an
affect on pricing (and that are indexed) are:

@TODO are all the below correct? I assumed a lot!

- Standard price, per-website if applicable
- Special price
- Tiered prices
- Customer group prices
- Catalog promotion rules (@TODO IS THIS INDEXED?)
- Bundle products
- Configurable products
- Custom options


### Catalog URL Rewrites

URL rewrites can come from a variety of sources in Magento, i.e. Products,
Categories, custom rewrites and third-party modules, to inspect each entity type
when a request comes in would be overly time-consuming, hence all possible
rewrites are combined in to this table.


### Category Products

Enumerating the products for a category isn't as simple as it might initially
seem due to the potential for category anchoring (whereby a parent catgory
"anchors" the products from it's child categories), so having an index which
allows quick resolution of the products in any one category makes a lot of
sense.


### Catalog Search Index

Search data must be compiled from a variety of product attributes (and
potentially other data, with third party modules) in to a data source which is
easy to search, community edition only offers MySQL fulltext, whereas enterprise
edition offers Solr, so it is posible that this index is not actually stored in
the database.


### Stock Status

@TODO I'm not totally sure of a reason for this one


### Tag Aggregation Data

@TODO never used these, so dunno


## How Indexing Works




## Notes

- Mage_Index implements the framework
- Each module holds it's own indexer class and defines it in in XML
- global/index/index_model -> Mage_Index_Model_Indexer
- Mage_Index_Model_Indexer::getProcessesCollection() gets all indexers
- Mage_Index_Model_Indexer::getProcessByCode() gets a single indexer
- Indexers must be added to DB table index_process
- They are referenced via the indexer_code:
    - catalog_product_attribute
    - catalog_product_price
    - catalog_url
    - catalog_product_flat
    - catalog_category_flat
    - catalog_category_product
    - catalogsearch_fulltext
    - cataloginventory_stock
    - tag_summary
- When the collection loads, each instance is of type Mage_Index_Model_Process (as you would expect)
- To reindex, you call reindexAll() or reindexEverything() on Mage_Index_Model_Process
- This locks the index first, which can be done via file or DB, default is file, it records start time too
- Then it reindexes, seems you can process "events" or a full reindex
    - @TODO what does events do? Seems to use index_process_event table, seems to be for deferred index of certain things
    - full does this:
        - calls getIndexer() on Mage_Index_Model_Process, which uses XML to get the indexer model
        - xml path is global/index/indexer/CODE i.e. global/index/indexer/catalog_product_attribute
        - getmodel on that is called
        - it's a subclass of Mage_Index_Model_Indexer_Abstract
        - reindexAll() is called on it
        - the above xml and model will be set in the module its related to, not the indexer
        - lets look at Mage_Tag_Model_Indexer_Summary:
            - so we come in on reindexAll()
            - this is defined in Mage_Index_Model_Indexer_Abstract and calls reindexAll() on the resource model
            - meaning each indexer has a model and resource, just like normal objects...
            - ...though this doesn't need to be true, e.g. the catalog url rewrite indexer does not
            - the model seems to be responsible for registering and naming the indexer:
                - getName()
                - getDescription()
            - the bulk of the model seems to be about event handling though:
                - $_matchedEntities property
                - _registerEvent()
                - _processEvent()
            - the resource model performs the actual reindexing, which seems largely unique to each index
                - aggregate() is called, this is unique to tags, it:
                    - starts a transaction
                    - deletes the contents of the tag summary table (the index)
                    - fetches all the tag relations, joining a bunch of tables as it does
                    - does some more uninportant stuff
                    - does an insert from select
                    - basically, does some expensive selects, then inserts them
                    - it only has one index table, so no idx and tmp, so none the wiser on those
                - idx/tmp - Mage_Index_Model_Resource_Abstract has behaviour to determine which to use
                    - seems like _idx is used for a full reindex
                    - tmp must therefore be used for partial reindexing via events?
                    - @TODO is this true?
                - the resource has some other bits:
                    - inserting from a select
                    - clearing temp tables
                    - disable/enable keys on tables
                    - sync data between tables (seems to go idx to main though, eh?)
- events:
    - process:
        - Mage_Index_Model_Indexer::logEvent()
        - Mage_Index_Model_Process::register()
        - Mage_Index_Model_Indexer_Abstract::register()
        - Mage_Index_Model_Indexer_Abstract::_registerEvent() - ABSTRACT
    - so individual entity indexers implement _registerEvent() in order to store event data
    - events are not 'for free' - logEvent has to be called from wherever you want to log an event
    - seems like logEvent() is called when indexing is to be deferred...
    - ...and if required immediatly, Mage_Index_Model_Indexer::processEntityAction() is called straight after...
    - ...(see Mage_CatalogInventory_Model_Stock_Item::afterCommitCallback())
    - ...so a DB row is stored in index_event and then read immediatly for immediate partial reindex, bit wasteful
    - so events are basically for deferred reindexing, but not sure when 'deferred' is used, as i thought it was immediate partial or full manual
    - oh i see, a partial reindex is forced when a reindex is requested, if events are present, see Mage_Index_Model_Process::reindexEverything()
    - when reindexing by event, this is the call trace:
        - Mage_Index_Model_Process::_processEventsCollection() is called once
        - Mage_Index_Model_Process::processEvent() is called for each event
        - Mage_Index_Model_Indexer_Abstract::processEvent()
        - Mage_Index_Model_Indexer_Abstract::_processEvent() - ABSTRACT
- interesting that indexing actually places its admin controller in its own module, not adminhtml
- indexes can depend on other indexes, so it's always fine to run exactly what you want:

    <global>
        <index>
            <indexer>
                <cataloginventory_stock>
                    <model>cataloginventory/indexer_stock</model>
                </cataloginventory_stock>
                <catalog_product_attribute>
                    <depends>
                        <cataloginventory_stock/>
                    </depends>
                </catalog_product_attribute>
                <catalog_product_price>
                    <depends>
                        <cataloginventory_stock/>
                    </depends>
                </catalog_product_price>
            </indexer>
        </index>
    </global>

- idx is full reindex, tmp is partial (a reminder)
- DB tables (that I know of):

catalog_category_anc_categs_index_idx - category full
catalog_category_anc_categs_index_tmp - category partial
catalog_category_anc_products_index_idx - anchor products full
catalog_category_anc_products_index_tmp - anchor products partial
catalog_category_product_index_enbl_idx - category -> product enabled full
catalog_category_product_index_enbl_tmp - category -> product enabled partial
catalog_category_product_index_idx - category -> product full
catalog_category_product_index_tmp - category -> product partial
catalog_product_index_eav_decimal_idx - @TODO
catalog_product_index_eav_decimal_tmp - @TODO
catalog_product_index_eav_idx - @TODO
catalog_product_index_eav_tmp - @TODO
catalog_product_index_price_bundle_idx - @TODO
catalog_product_index_price_bundle_opt_idx - @TODO
catalog_product_index_price_bundle_opt_tmp - @TODO
catalog_product_index_price_bundle_sel_idx - @TODO
catalog_product_index_price_bundle_sel_tmp - @TODO
catalog_product_index_price_bundle_tmp - @TODO
catalog_product_index_price_cfg_opt_agr_idx - @TODO
catalog_product_index_price_cfg_opt_agr_tmp - @TODO
catalog_product_index_price_cfg_opt_idx - @TODO
catalog_product_index_price_cfg_opt_tmp - @TODO
catalog_product_index_price_downlod_idx - @TODO
catalog_product_index_price_downlod_tmp - @TODO
catalog_product_index_price_final_idx - @TODO
catalog_product_index_price_final_tmp - @TODO
catalog_product_index_price_idx - @TODO
catalog_product_index_price_opt_agr_idx - @TODO
catalog_product_index_price_opt_agr_tmp - @TODO
catalog_product_index_price_opt_idx - @TODO
catalog_product_index_price_opt_tmp - @TODO
catalog_product_index_price_tmp - @TODO
cataloginventory_stock_status_idx - @TODO
cataloginventory_stock_status_tmp - @TODO
index_event - @TODO
index_process_event - @TODO

- looks like from the above that cateogry -> product is done via several indexes, probably dependant, @TODO check

Model:

    Mage_Core_Model_Abstract
        |-> Mage_Index_Model_Indexer_Abstract
            |-> Mage_Tag_Model_Indexer_Summary

Resource:

    Mage_Index_Model_Resource_Abstract
        |-> Mage_Catalog_Model_Resource_Product_Indexer_Abstract
            |-> Mage_Tag_Model_Indexer_Summary

Todo:

- start with the indexer shell command to show how indexes work
- show a list of tables, including idx and tmp, state what each is for
- look at price index, it's a beast!
