# Create Customer Pseudo Code

A service that creates a customer if not present already in the CDP.

## CreateShopifyCustomer Service

- ### In Params
    - This service takes a Map with name **payload** as required input.

- ### Out Params
    - Returns partyId of created Customer and a messages map.

- ### Pseudo Code

    ```
    customer = payload.customer
        if (!customer) return error "Customer object cannot be empty."

        customerId = customer.id

        if (!customerId) return error "id field is missing in Customer"
    ```
    Shopify claims that same customer accross multiple stores have same **multipass_identifier** but different id and could have different personal details like name and addressed. This allows stores to have customized customer information while enabling a seamless login experience across stores.
    ```
    findShopifyCustomer(customerId) // This method finds party with customer role for a particular Shopify Store
    ```
    **Note** - Shopify Help Center Claims that the customer objects can exist without name fields in case where customer abandoned case on checkout or the customer didn't fill out the name fields. The questions here are:
    1. Will this situation rise for us when we synchronysing Customer in our System.
    2. Should we create customer without the fields like first_name or last_name because shopify allows.
    3. There are also some fields in Customer Object that are claims to be non-null in [Shopify Customer GraphQL API Documenation](https://shopify.dev/docs/api/admin-graphql/latest/objects/Customer#field-Customer.fields.firstName). State is one of the non-null fields in GraphQL api response but not mush is shared about response the [REST API Doc](https://shopify.dev/docs/api/admin-rest/latest/resources/customer#get-customers-customer-id). So should we create such customer and disable it and update it later when required if updated on Shopify. 
    ```
        customerEnabled = 'N'
        if (!customer.state) {
            customer.state = 'NA'
            add message "Customer missing state and is marked disabled in CDP"
        } else-if (customer.state == 'ENABLED') {
            customerEnabled = 'Y'
        }

        classifications = [] /** List of maps to store the classification elegible for communication mode an opt in level with the consent_updated_at datetime. **/ 

        if (customer.accepts_marketing == true) {
            add classfications of email and sms consent state and opt in level
            classifications.[classfication type mapped with marketing consent type and opt in level,  fromDate: consent_updated_at]
        }

        // Add Key Value(not null) Pairs in a list, will create PartyAttribute records.
        attributes: [state, orders_count, total_spent, last_order_id, verified_email, tags, last_order_name, currency, admin_graphql_api_id]

        postalAddresses = []
        phoneNumbers = []

        if (customer.addresses) {
            Iterate: customer.addresses as contact

            lookup: provice_code and the country_code in Geo entity, if found then assign provice_code and the country_code with respective GEO_ID.

            postalAddress = [
                toName: ${contact.firstName} ${contact.lastName},
                attnName: ${contact.name} ${contact.company},
                address1: contact.address1,
                address2: contact.address2,
                city: contact.city,
                postalCode: zip,
                provinceGeoId: provice_code,
            ]

            telecomNumber: function: validate and transform phone number format into contry code, area code and contact number(contact.phone)

            if (contact.default) {
                potaladdress.contactMechPurposeId: 'DefaultAddress',
                telecomNumber.contactMechPurposeId: 'DefaultPhone'
            } else {
                potaladdress.contactMechPurposeId: 'PrimaryAddress',
                telecomNumber.contactMechPurposeId: 'PrimaryPhone'
            }

            create-calls: for both telecomNumber and postalAddress 
        }
        if (customer.email) {
            function: validate email format
            create-call: contactMech with infoString = customer.email;
        }

        create:call by nested Map: Party, PartyIdentification Person, PartyRole, PartyAttribute, PartyRelationship

        service-call: creating PartyClassification, Appl, MarketSegmemt 


        // TODO: Add Tax Classifications, Party-Shop and Party-ProductStore assics.  
    ```



    