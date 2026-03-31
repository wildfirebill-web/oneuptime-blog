# How to Use MongoDB with Strapi CMS

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Strapi, CMS

Description: Configure Strapi CMS to use MongoDB as the database backend, with connection setup, content type creation, and custom query examples.

---

## Strapi and MongoDB

Strapi is a headless CMS that supports multiple databases. MongoDB works well with Strapi when your content types benefit from flexible schemas or when you are already running a MongoDB infrastructure. Strapi v4 uses an abstraction layer called Strapi Database that supports MongoDB through its Mongoose connector.

## Creating a Strapi Project with MongoDB

```bash
npx create-strapi-app@latest my-cms --dbclient=mongo \
  --dbhost=localhost \
  --dbport=27017 \
  --dbname=strapidb \
  --dbusername=strapi \
  --dbpassword=strapipass
```

## Manual Database Configuration

Edit `config/database.js`:

```javascript
module.exports = ({ env }) => ({
  connection: {
    client: 'mongoose',
    connection: {
      host: env('DATABASE_HOST', 'localhost'),
      port: env.int('DATABASE_PORT', 27017),
      database: env('DATABASE_NAME', 'strapidb'),
      username: env('DATABASE_USERNAME', ''),
      password: env('DATABASE_PASSWORD', ''),
    },
    useNullAsDefault: true,
    options: {
      authenticationDatabase: 'admin',
      ssl: env.bool('DATABASE_SSL', false),
    },
  },
})
```

Set up `.env`:

```text
DATABASE_HOST=localhost
DATABASE_PORT=27017
DATABASE_NAME=strapidb
DATABASE_USERNAME=strapi
DATABASE_PASSWORD=strapipass
```

## Creating a Content Type

Use the Strapi admin panel or the CLI to create content types. Each content type maps to a MongoDB collection:

```bash
# Create an Article content type via CLI
npx strapi generate content-type article
```

The generated schema in `src/api/article/content-types/article/schema.json`:

```json
{
  "kind": "collectionType",
  "collectionName": "articles",
  "info": {
    "singularName": "article",
    "pluralName": "articles",
    "displayName": "Article"
  },
  "attributes": {
    "title": {
      "type": "string",
      "required": true,
      "maxLength": 200
    },
    "content": {
      "type": "richtext"
    },
    "publishedAt": {
      "type": "datetime"
    },
    "author": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "plugin::users-permissions.user"
    }
  }
}
```

## Custom MongoDB Query in a Strapi Service

When you need a query beyond Strapi's built-in filters, use the entity service or the underlying Mongoose model:

```javascript
// src/api/article/services/article.js
'use strict'

module.exports = {
  async findPopular() {
    const model = strapi.db.connection.model('Article')
    return model.find({
      publishedAt: { $ne: null },
    })
    .sort({ views: -1 })
    .limit(10)
    .select('title publishedAt views')
    .lean()
  },

  async countByAuthor(authorId) {
    const model = strapi.db.connection.model('Article')
    return model.countDocuments({ 'author._id': authorId })
  },
}
```

## Custom Controller Endpoint

```javascript
// src/api/article/controllers/article.js
'use strict'

module.exports = {
  async popular(ctx) {
    const articles = await strapi.service('api::article.article').findPopular()
    ctx.body = articles
  },
}
```

Add the route in `src/api/article/routes/article.js`:

```javascript
module.exports = {
  routes: [
    {
      method: 'GET',
      path: '/articles/popular',
      handler: 'article.popular',
      config: { policies: [] },
    },
  ],
}
```

## Summary

Strapi integrates with MongoDB through its Mongoose connector, configured via `config/database.js` and environment variables. Content types map directly to MongoDB collections, giving you Strapi's admin UI and REST/GraphQL APIs with MongoDB's flexible storage. For queries beyond Strapi's filter system, access the underlying Mongoose model through the entity service to write native MongoDB aggregations and queries.
