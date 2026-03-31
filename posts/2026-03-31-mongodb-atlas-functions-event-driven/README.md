# How to Write Atlas Functions for Event-Driven Logic in MongoDB

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Atlas Functions, Serverless

Description: Write and deploy Atlas Functions in MongoDB to execute server-side JavaScript logic in response to database triggers, scheduled events, and HTTPS endpoints.

---

Atlas Functions are server-side JavaScript (Node.js runtime) functions that run in the MongoDB Atlas cloud. They execute in response to triggers (database changes, schedules, authentication events) or via direct HTTPS calls. They have built-in access to your Atlas cluster without managing connection strings or credentials.

## Writing Your First Atlas Function

Create a function in the Atlas UI under App Services - Functions, or via the Atlas CLI:

```javascript
// Function: processNewOrder
exports = async function(order) {
  // context.services gives access to linked Atlas cluster
  const mongodb = context.services.get("mongodb-atlas");
  const db = mongodb.db("production");

  // Validate the order
  if (!order.customerId || !order.items || order.items.length === 0) {
    throw new Error("Invalid order: missing customerId or items");
  }

  // Calculate total
  const total = order.items.reduce((sum, item) => {
    return sum + (item.price * item.quantity);
  }, 0);

  // Insert enriched order
  const result = await db.collection("orders").insertOne({
    ...order,
    total,
    status: "pending",
    createdAt: new Date()
  });

  // Send notification
  await context.functions.execute("sendOrderConfirmation", {
    orderId: result.insertedId.toString(),
    customerId: order.customerId,
    total
  });

  return { orderId: result.insertedId.toString(), total };
};
```

## Accessing the Database

```javascript
exports = async function() {
  const client = context.services.get("mongodb-atlas");
  const db = client.db("myDatabase");
  const collection = db.collection("users");

  // Standard CRUD operations
  const user = await collection.findOne({ email: "user@example.com" });
  await collection.updateOne(
    { _id: user._id },
    { $set: { lastSeen: new Date() } }
  );

  return user;
};
```

## Calling External HTTP APIs

```javascript
exports = async function(userId) {
  // Fetch user profile from external service
  const response = await context.http.get({
    url: `https://api.myservice.com/users/${userId}`,
    headers: {
      "Authorization": [`Bearer ${context.values.get("API_KEY")}`]
    }
  });

  if (response.statusCode !== 200) {
    throw new Error(`External API error: ${response.statusCode}`);
  }

  const profile = EJSON.parse(response.body.text());
  return profile;
};
```

`context.values.get("API_KEY")` reads a secret value configured in Atlas App Services.

## Sending Emails with Atlas

```javascript
exports = async function(recipientEmail, subject, body) {
  const response = await context.http.post({
    url: "https://api.sendgrid.com/v3/mail/send",
    headers: {
      "Authorization": [`Bearer ${context.values.get("SENDGRID_KEY")}`],
      "Content-Type": ["application/json"]
    },
    body: JSON.stringify({
      personalizations: [{ to: [{ email: recipientEmail }] }],
      from: { email: "noreply@example.com" },
      subject,
      content: [{ type: "text/plain", value: body }]
    })
  });

  return { statusCode: response.statusCode };
};
```

## Calling One Function from Another

```javascript
exports = async function(orderId) {
  const order = await context.functions.execute("getOrder", orderId);
  const emailSent = await context.functions.execute("sendOrderConfirmation", order);
  return { order, emailSent };
};
```

## Logging and Debugging

Atlas Functions write to the App Services Logs, visible in the Atlas UI:

```javascript
exports = async function(input) {
  console.log("Processing input:", JSON.stringify(input));

  try {
    const result = await doSomething(input);
    console.log("Success:", result);
    return result;
  } catch (err) {
    console.error("Function failed:", err.message, err.stack);
    throw err;
  }
};
```

Access logs in Atlas UI under App Services - Logs, or via the Atlas Admin API.

## Summary

Atlas Functions run server-side JavaScript in response to triggers, schedules, or HTTPS calls. They have built-in access to your Atlas cluster via `context.services`, can call external APIs with `context.http`, read secrets via `context.values`, and invoke other functions with `context.functions.execute`. Write focused, single-responsibility functions and use structured logging to simplify debugging through the Atlas Logs UI.
