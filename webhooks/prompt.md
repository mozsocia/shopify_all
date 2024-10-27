
### Shopify app development node template, how to use webhooks ?


```
//shopify.js

import { BillingInterval, LATEST_API_VERSION } from "@shopify/shopify-api";
import { shopifyApp } from "@shopify/shopify-app-express";
import { SQLiteSessionStorage } from "@shopify/shopify-app-session-storage-sqlite";
import { restResources } from "@shopify/shopify-api/rest/admin/2024-07";

const DB_PATH = `${process.cwd()}/database.sqlite`;

// The transactions with Shopify will always be marked as test transactions, unless NODE_ENV is production.
// See the ensureBilling helper to learn more about billing in this template.
const billingConfig = {
  "My Shopify One-Time Charge": {
    // This is an example configuration that would do a one-time charge for $5 (only USD is currently supported)
    amount: 5.0,
    currencyCode: "USD",
    interval: BillingInterval.OneTime,
  },
};

const shopify = shopifyApp({
  api: {
    apiVersion: LATEST_API_VERSION,
    restResources,
    future: {
      customerAddressDefaultFix: true,
      lineItemBilling: true,
      unstable_managedPricingSupport: true,
    },
    billing: undefined, // or replace with billingConfig above to enable example billing
  },
  auth: {
    path: "/api/auth",
    callbackPath: "/api/auth/callback",
  },
  webhooks: {
    path: "/api/webhooks",
  },
  // This should be replaced with your preferred storage strategy
  sessionStorage: new SQLiteSessionStorage(DB_PATH),
});

export default shopify;

```

```
//index.js


// @ts-check
import { join } from "path";
import { readFileSync } from "fs";
import express from "express";
import serveStatic from "serve-static";

import shopify from "./shopify.js";
import productCreator from "./product-creator.js";
import PrivacyWebhookHandlers from "./privacy.js";

const PORT = parseInt(
  process.env.BACKEND_PORT || process.env.PORT || "3000",
  10
);

const STATIC_PATH =
  process.env.NODE_ENV === "production"
    ? `${process.cwd()}/frontend/dist`
    : `${process.cwd()}/frontend/`;

const app = express();

// Set up Shopify authentication and webhook handling
app.get(shopify.config.auth.path, shopify.auth.begin());
app.get(
  shopify.config.auth.callbackPath,
  shopify.auth.callback(),
  shopify.redirectToShopifyOrAppRoot()
);
app.post(
  shopify.config.webhooks.path,
  shopify.processWebhooks({ webhookHandlers: PrivacyWebhookHandlers })
);

// If you are adding routes outside of the /api path, remember to
// also add a proxy rule for them in web/frontend/vite.config.js

app.use("/api/*", shopify.validateAuthenticatedSession());

app.use(express.json());

app.get("/api/products/count", async (_req, res) => {
  const client = new shopify.api.clients.Graphql({
    session: res.locals.shopify.session,
  });

  const countData = await client.request(`
    query shopifyProductCount {
      productsCount {
        count
      }
    }
  `);

  res.status(200).send({ count: countData.data.productsCount.count });
});

app.post("/api/products", async (_req, res) => {
  let status = 200;
  let error = null;

  try {
    await productCreator(res.locals.shopify.session);
  } catch (e) {
    console.log(`Failed to process products/create: ${e.message}`);
    status = 500;
    error = e.message;
  }
  res.status(status).send({ success: status === 200, error });
});

app.use(shopify.cspHeaders());
app.use(serveStatic(STATIC_PATH, { index: false }));

app.use("/*", shopify.ensureInstalledOnShop(), async (_req, res, _next) => {
  return res
    .status(200)
    .set("Content-Type", "text/html")
    .send(
      readFileSync(join(STATIC_PATH, "index.html"))
        .toString()
        .replace("%VITE_SHOPIFY_API_KEY%", process.env.SHOPIFY_API_KEY || "")
    );
});

app.listen(PORT);


```




I'll explain how to implement webhooks in your Shopify app using this template. Here's a step-by-step guide:

1. First, create a webhook handler file similar to the existing `privacy.js`. For example, let's create `webhookHandlers.js`:

```javascript
const webhookHandlers = {
  // Handler for product creation
  PRODUCTS_CREATE: {
    handler: async (topic, shop, body) => {
      console.log('Product created webhook received');
      const product = JSON.parse(body);
      // Add your product creation logic here
    },
  },
  
  // Handler for order creation
  ORDERS_CREATE: {
    handler: async (topic, shop, body) => {
      console.log('Order created webhook received');
      const order = JSON.parse(body);
      // Add your order creation logic here
    },
  },
};

export default webhookHandlers;
```

2. Register your webhooks when the app is installed. Create a new file called `webhookRegistration.js`:

```javascript
export const registerWebhooks = async (session) => {
  const webhookTopics = [
    'PRODUCTS_CREATE',
    'ORDERS_CREATE'
    // Add more webhook topics as needed
  ];

  for (const topic of webhookTopics) {
    try {
      await shopify.api.webhooks.addHandler({
        topic,
        path: '/api/webhooks',
        webhookHandler: async (topic, shop, body) => {
          console.log(`Webhook received: ${topic}`);
          // Your webhook handling logic here
        },
      });
      console.log(`Successfully registered ${topic} webhook`);
    } catch (error) {
      console.error(`Failed to register ${topic} webhook`, error);
    }
  }
};
```

3. Update your `index.js` to use the new webhook handlers:

```javascript
import webhookHandlers from "./webhookHandlers.js";

// Update the webhook processing line
app.post(
  shopify.config.webhooks.path,
  shopify.processWebhooks({ webhookHandlers })
);

// Add webhook registration after successful auth
app.get(
  shopify.config.auth.callbackPath,
  shopify.auth.callback(),
  async (req, res, next) => {
    const session = res.locals.shopify.session;
    await registerWebhooks(session);
    next();
  },
  shopify.redirectToShopifyOrAppRoot()
);
```

4. Common webhook topics you might want to use:

```javascript
// Examples of available webhook topics
const WEBHOOK_TOPICS = {
  PRODUCTS: {
    CREATE: 'PRODUCTS_CREATE',
    UPDATE: 'PRODUCTS_UPDATE',
    DELETE: 'PRODUCTS_DELETE'
  },
  ORDERS: {
    CREATE: 'ORDERS_CREATE',
    UPDATED: 'ORDERS_UPDATED',
    CANCELLED: 'ORDERS_CANCELLED'
  },
  CUSTOMERS: {
    CREATE: 'CUSTOMERS_CREATE',
    UPDATE: 'CUSTOMERS_UPDATE',
    DELETE: 'CUSTOMERS_DELETE'
  }
};
```

5. Example of a more detailed webhook handler:

```javascript
const webhookHandlers = {
  ORDERS_CREATE: {
    handler: async (topic, shop, body) => {
      try {
        const order = JSON.parse(body);
        
        // Example: Store order in your database
        await db.orders.create({
          shopifyOrderId: order.id,
          shopDomain: shop,
          totalPrice: order.total_price,
          customerEmail: order.email,
          createdAt: new Date()
        });

        // Example: Send notification
        await sendNotification({
          type: 'new_order',
          shop,
          orderId: order.id
        });

      } catch (error) {
        console.error('Error processing order webhook:', error);
      }
    },
  }
};
```

Remember:
- Webhook handlers should be idempotent as Shopify may send the same webhook multiple times
- Webhooks must respond within 5 seconds; for longer processing, queue the work
- Use try/catch blocks to handle errors gracefully
- Keep track of processed webhooks to avoid duplicate processing
- Ensure your webhook endpoint is accessible via HTTPS

To verify webhooks are working:
1. Make sure your app is accessible via HTTPS
2. Test by creating/updating resources in your Shopify store
3. Check your logs for webhook receipts
4. Use Shopify's webhook dashboard to monitor delivery status
