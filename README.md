# Google-Analytics-GA4-Integration-with-BigCommerce-Enhanced-Ecommerce-Tracking

Step 1. Copy GA4 code from your Google Analytics admin dashboard
The GA4 Code will look like this

 <!-- Global site tag (gtag.js) - Google Analytics -->
  <!-- Please Replace XXXXXXXXXX with your GA4 Code -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'G-XXXXXXXXXX');
  </script>

  Step 2: Adding GA4 Global Script to Bigcommerce
In this step, we’ll install the GA4 script across all the site pages (including checkout and order confirmation pages). BigCommerce provides a pretty easy interface to include scripts sitewide. Please go to your BigCommerce admin dashboard and go to the script manager to create a new script with following attributes:

Name of Script: GA4 Global Script
Description: Custom GA4 Global Script
Location: Head
Select pages where script will be added: All pages
Script Category: Analytics
Script Type: Script
Script Content: Now copy the script from GA4 Code, paste it into the script content area, and click Save.

![image](https://github.com/xeniusdigital/Google-Analytics-GA4-Integration-with-BigCommerce-Enhanced-Ecommerce-Tracking/assets/5832613/c673cc65-9f61-48ba-a81e-bf0593973bd4)

Step 3: Set GA4 User properties and add page specific events
Google Analytics allows us to track very detailed information on the site’s user activity. The full list of events can be found here. As BigCommerce is a SaaS platform, I’ll be using events which are mostly related to the eCommerce industry, eg: view_item, view_cart, add_to_cart, etc. I have not included a remove_from_cart event because that will require adding code to the theme JS file (which requires local theme setup). I tried to keep this installation simple so that we can easily manage it from the BigCommerce admin panel.

To setup custom user properties and add page events, we’ll need to create another script in BigCommerce from the script manager with the following attributes:

Name of Script: GA4 User Properties and Page Events
Description: Custom Events
Location: Footer
Select pages where script will be added: All pages
Script Category: Analytics
Script Type: Script
Script Content: Please copy the code given below and paste into script content section

    <script>
    {{#if customer }}
        gtag('set', 'user_id', '{{customer.id}}');
        gtag('set', 'user_properties', {
            'customer_shipping_country': '{{customer.shipping_address.country}}',
            'customer_shipping_state': '{{customer.shipping_address.state}}',
            'customer_shipping_city': '{{customer.shipping_address.city}}',
            'customer_shipping_address_type': '{{customer.shipping_address.destination}}'
        });
    {{/if}}
    {{#if customer.customer_group_name }}
        gtag('set', 'user_properties', {
            'customer_group': '{{customer.customer_group_name}}'
        });
    {{/if}}
    {{#if page_type '===' 'createaccount_thanks' }}
        gtag("event", "sign_up", {
            method: "Website"
        });
    {{/if}}

    {{#if page_type '===' 'login' }}
        document.querySelector("form[action='/login.php?action=check_login']").addEventListener("submit", function () {
            gtag("event", "login", { method: "Website" });
        });   
    {{/if}}

    {{#if page_type '===' 'product'}}
        gtag("event", "view_item", {
            currency: "{{currency_selector.active_currency_code}}",
            value: {{#or customer (unless theme_settings.restrict_to_login)}}{{#if product.price.with_tax}}parseFloat({{product.price.with_tax.value}}){{else}}parseFloat({{product.price.without_tax.value}}){{/if}}{{else}}"{{lang 'common.login_for_pricing'}}"{{/or}},
            items: [
                {
                    item_id: "{{#if product.sku}}{{product.sku}}{{else}}{{product.id}}{{/if}}",
                    item_name: "{{product.title}}",
                    currency: "{{currency_selector.active_currency_code}}",
                    discount: parseFloat({{product.price.saved.value}}),
                    item_brand: "{{product.brand.name}}",
                    price: {{#or customer (unless theme_settings.restrict_to_login)}}{{#if product.price.with_tax}}parseFloat({{product.price.with_tax.value}}){{else}}parseFloat({{product.price.without_tax.value}}){{/if}}{{else}}"{{lang 'common.login_for_pricing'}}"{{/or}},
                    quantity: 1,
                    {{#each product.category}}{{#if @first}}item_category:"{{this}}"{{else}},item_category{{@index}}:"{{this}}"{{/if}}{{/each}}
                }
            ]
        });
    {{/if}}
    {{#if page_type '===' 'cart'}}
        gtag("event", "view_cart", {
            currency: "{{currency_selector.active_currency_code}}",
            value: parseFloat({{cart.sub_total.value}}),
            items: [
                {{#each cart.items}}
                {
                    item_id: "{{#if sku}}{{sku}}{{else}}{{product_id}}{{/if}}",
                    item_name: "{{name}}",
                    item_brand: "{{brand.name}}",                    
                    price: parseFloat({{price.value}}),
                    quantity: parseInt({{quantity}})
                },
                {{/each}}                                        
        ]});
    {{/if}}
    {{#if page_type '===' 'search'}}
        gtag('set', 'page_title', 'Search');
        gtag("event", "search", {
            search_term: "{{sanitize forms.search.query}}"
        });
    {{/if}}
    </script>

    The screenshot below shows recommended settings:

    
![image](https://github.com/xeniusdigital/Google-Analytics-GA4-Integration-with-BigCommerce-Enhanced-Ecommerce-Tracking/assets/5832613/fea9102a-0816-43d3-9701-6756ea55f25d)

Step 4: Add GA4 Checkout Steps events to the BigCommerce Optimized One-Page Checkout
In this steps we’ll configure, begin_checkout, add_shipping_info and add_payment_info events. BigCommerce checkout data is not available to stencil handlebar variables. So we’ll be using the BigCommerce Storefront API to fetch the cart details, and we’ll use Mutation Observer to to identify the customers’ progress on different steps within the checkout page.

As the checkout page is loading, we’ll trigger the begin_checkout event. Then we’ll search for an element with the class .paymentMethod. Please note that these class names may change with future BigCommerce checkout updates, so please add the appropriate selector when you handle implementation. I am using the paymentMethod selector to trigger add_shipping_info because it ensures that the user has selected a shipping method already and we’ll be able to push the shipping information data to Google Analytics. Similarly, for add_payment_info, I trigger this event when someone clicks on the place order button as their last step. It ensures that the user has selected a payment method and is ready to proceed. The only drawback of this approach is that if someone enters incorrect payment information, the event will be triggered still. You can create a new script in BigCommerce Script Editor as per following attributes and paste the code given below:

Name of Script: GA4 Checkout Page Events
Description: Custom Checkout Events
Location: Footer
Select pages where script will be added: Checkout
Script Category: Analytics
Script Type: Script
Script Content: Please copy the code given below and paste into script content section

  <script>
  (function(win) {
      'use strict';

      var listeners = [], 
      doc = win.document, 
      MutationObserver = win.MutationObserver || win.WebKitMutationObserver,
      observer;

      function ready(selector, fn) {
          // Store the selector and callback to be monitored
          listeners.push({
              selector: selector,
              fn: fn
          });
          if (!observer) {
              // Watch for changes in the document
              observer = new MutationObserver(check);
              observer.observe(doc.documentElement, {
                  childList: true,
                  subtree: true
              });
          }
          // Check if the element is currently in the DOM
          check();
      }

      function check() {
          // Check the DOM for elements matching a stored selector
          for (var i = 0, len = listeners.length, listener, elements; i < len; i++) {
              listener = listeners[i];
              // Query for elements matching the specified selector
              elements = doc.querySelectorAll(listener.selector);
              for (var j = 0, jLen = elements.length, element; j < jLen; j++) {
                  element = elements[j];
                  // Make sure the callback isn't invoked with the 
                  // same element more than once
                  if (!element.ready) {
                      element.ready = true;
                      // Invoke the callback with the element
                      listener.fn.call(element, element);
                  }
              }
          }
      }

      // Expose `ready`
      win.ready = ready;

  })(this);

  // Global Variable to store Items for each step
  window.orderData = window.orderData || [];

  gtag('set', 'page_title', 'Checkout');

  // Begin Checkout - Fire on page load
  fetch('/api/storefront/checkouts/{{checkout.id}}', {
          credentials: 'include'
      }).then(function (response) {
          return response.json();
      }).then(function (data) {
          window.orderData.push(data);
          const orderItems = window.orderData[0].cart.lineItems.physicalItems.map(item => {
              const container = {};
              container['item_id'] = Boolean(item.sku)? item.sku : item.productId;
              container['item_name'] = item.name;
              container['price'] = item.salePrice;
              container['quantity'] = item.quantity;
              return container;
          });
          gtag("event", "begin_checkout", {
            currency: window.orderData[0].cart.currency.code,
            value: window.orderData[0].subtotal,
            coupon: window.orderData[0].coupons.length > 0 ? window.orderData[0].coupons[0].code:'',
            items: orderItems
          });    
  });


  ready('.paymentMethod', function(element) {

  // Shipping Info Added event gets fired when payment section is loaded        
  fetch('/api/storefront/checkouts/{{checkout.id}}', {
      credentials: 'include'
  }).then(function (response) {
      return response.json();
  }).then(function (data) {
      window.orderData.push(data);
      const orderItems = window.orderData[1].cart.lineItems.physicalItems.map(item => {
          const container = {};
          container['item_id'] = Boolean(item.sku)? item.sku : item.productId;
          container['item_name'] = item.name;
          container['price'] = item.salePrice;
          container['quantity'] = item.quantity;
          return container;
      });

      gtag("event", "add_shipping_info", {
          currency: window.orderData[1].cart.currency.code,
          value: window.orderData[1].subtotal,
          coupon: window.orderData[1].coupons.length > 0 ? window.orderData[1].coupons[1].code:'',
          shipping_tier: window.orderData[1].consignments[0].selectedShippingOption.description,
          items: orderItems
      });    
  }); // fetch function ends

  // Add Payment Info event gets fired when Place Order Button is clicked
  var paymentButton = document.getElementById('checkout-payment-continue');
  if (paymentButton){
      paymentButton.addEventListener("click", function(e) {
          let paymentOptions = document.querySelector('.checkout-step--payment .optimizedCheckout-form-checklist-item--selected input[type="radio"]');

          const orderItems = window.orderData[1].cart.lineItems.physicalItems.map(item => {
              const container = {};
              container['item_id'] = Boolean(item.sku)? item.sku : item.productId;
              container['item_name'] = item.name;
              container['price'] = item.salePrice;
              container['quantity'] = item.quantity;
              return container;
          });
          gtag("event", "add_payment_info", {
              currency: window.orderData[1].cart.currency.code,
              payment_type: Boolean(paymentOptions.value) ? paymentOptions.value : 'default',
              value: window.orderData[1].subtotal,
              coupon: window.orderData[1].coupons.length > 0 ? window.orderData[1].coupons[1].code:'',
              shipping_tier: window.orderData[1].consignments[0].selectedShippingOption.description,
              items: orderItems
          }); 
      }); // click event ends
  } // if condition ends

  }); // Ready function ends
  </script>

  Step 5: Add GA4 eCommerce Conversion Tracking on BigCommerce order confirmation page
So far we have configured pretty detailed DataLayers and events for different user actions and page views. Now we’ll configure purchase event on the order confirmation page to add revenue tracking within Google Analytics. We’ll be using the BigCommerce storefront API to fetch the order data and then push that information to GA. You can create a new script in BigCommerce Script Editor as per following attributes and paste the code given below:

Name of Script: GA4 Conversion Tracking
Description: Custom GA4 Purchase Event
Location: Footer
Select pages where script will be added: Order confirmation
Script Category: Analytics
Script Type: Script
Script Content: Please copy the code given below and paste into script content section

  <script>    
  // Fetch Order Data
  gtag('set', 'page_title', 'Order Confirmation');    
  fetch('/api/storefront/order/{{checkout.order.id}}', {
    credentials: 'include'
  }).then(function (response) {
    return response.json();
  }).then(function (orderData) {
    const orderItems = orderData.lineItems.physicalItems.map(item => {
        const container = {};
        container['item_id'] = Boolean(item.sku)? item.sku : item.id;
        container['item_name'] = item.name;
        container['price'] = item.salePrice;
        container['quantity'] = item.quantity;
        return container;
    });        
    gtag("event", "purchase", {
        transaction_id: '{{checkout.order.id}}',
        value: orderData.orderAmount,
        tax: orderData.taxTotal,
        shipping: orderData.shippingCostTotal,
        currency: orderData.currency.code,
        coupon: orderData.coupons.length > 0 ? orderData.coupons[0].code:'',
        items: orderItems
    });

  });
  </script>


#Implementing advanced GA4 events in a BigCommerce Theme
In the previous steps, you implemented most of the eCommerce events within the Script manager. BigCommerce themes have some Ajax functionality like: Quick Search Results, Product Quick View Popup, Product Add to Cart popup, Quick Cart Popup in the header, etc. To track the user interaction with these, we’ll need to add some custom code to the BigCommerce theme files. The file paths are given according to Cornerstone theme structure. If you’re using a different theme, then you may have different file structure.

#Track Search Results query within Quick Results popup
If you’re using the BigCommerce Cornerstone theme, you’ll notice that if you type a search query in the search box, it will give you instant search results. There’s possibility that people don’t go to the search results page and just find out their result within in the quick results popup. To track search queries for quick search popup, please follow this path in your Cornerstone theme: /templates/components/search/quick-results.html. Open quick-results.html file and copy the code given below and paste it at the end of the file.

  <!-- File Path for Cornerstone Theme: /templates/components/search/quick-results.html -->
  <script>
  gtag("event", "search", {
      search_term: "{{sanitize forms.search.query}}"
  });
  </script>

#Track Product Views in Quick View
Sometimes customers may simply click on the Quick View button to see product information and don’t go to the product detail page at all. So it’s good idea to track the ‘product_view’ event for the Quick View popup. To track quick popups product views, please follow this path in your cornerstone theme: /templates/components/products/quick-view.html. Open quick-view.html file and copy the code given below and paste it at the end of the file.

 <!-- File Path for Cornerstone Theme: /templates/components/products/quick-view.html -->
  <script type="text/javascript">
  gtag("event", "view_item", {
    currency: "{{currency_selector.active_currency_code}}",
    value: {{#or customer (unless theme_settings.restrict_to_login)}}{{#if product.price.with_tax}}parseFloat({{product.price.with_tax.value}}){{else}}parseFloat({{product.price.without_tax.value}}){{/if}}{{else}}"{{lang 'common.login_for_pricing'}}"{{/or}},
    items: [
        {
            item_id: "{{#if product.sku}}{{product.sku}}{{else}}{{product.id}}{{/if}}",
            item_name: "{{product.title}}",
            currency: "{{currency_selector.active_currency_code}}",
            discount: parseFloat({{product.price.saved.value}}),
            item_brand: "{{product.brand.name}}",
            price: {{#or customer (unless theme_settings.restrict_to_login)}}{{#if product.price.with_tax}}parseFloat({{product.price.with_tax.value}}){{else}}parseFloat({{product.price.without_tax.value}}){{/if}}{{else}}"{{lang 'common.login_for_pricing'}}"{{/or}},
            quantity: 1,
            {{#each product.category}}{{#if @first}}item_category:"{{this}}"{{else}},item_category{{@index}}:"{{this}}"{{/if}}{{/each}}
        }
    ]
  });
  </script>


#Track Ajax Add to Cart Event
There are few different approaches to track Add to cart events. We can trigger the event when someone clicks the add to cart buttons or we can edit the theme JS files and trigger the event in success callback of add to cart function. To make things easier, I have added the gtag function within the Add to cart success popup. Please follow this path in your cornerstone theme: /templates/components/cart/preview.html. Open preview.html file and copy the code given below and paste it at the end of the file.

  <!-- File Path for Cornerstone Theme: /templates/components/cart/preview.html -->
  <script>
  gtag("event", "add_to_cart", {
    currency: "{{currency_selector.active_currency_code}}",
    value: parseFloat({{cart.added_item.price.value}}) * parseInt({{cart.added_item.quantity}}),
    items: [
      {
        item_id: "{{#if cart.added_item.sku}}{{cart.added_item.sku}}{{else}}{{cart.added_item.id}}{{/if}}",
        item_name: "{{cart.added_item.name}}",
        currency: "{{currency_selector.active_currency_code}}",
        item_brand: "{{cart.added_item.brand.name}}",
        price: parseFloat({{cart.added_item.price.value}}),
        quantity: parseInt({{cart.added_item.quantity}})
      }
    ]
  });
  </script>


  Source: https://byaman.com/ga4-integration-with-bigcommerce-enhanced-ecommerce-tracking/

  
