# How to Use MySQL with Sequelize

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MySQL, Sequelize, Node.js, ORM, JavaScript

Description: Learn how to set up Sequelize ORM with MySQL in Node.js, define models, create associations, run migrations, and query data with Sequelize's fluent API.

---

## Introduction

Sequelize is a promise-based ORM for Node.js that supports MySQL natively. It provides model definitions, associations, migrations via Sequelize CLI, and a powerful query interface. This guide covers setting up Sequelize with MySQL and performing common database operations.

## Installation

```bash
npm install sequelize mysql2
npm install -D sequelize-cli
```

## Connecting to MySQL

```javascript
const { Sequelize } = require('sequelize');

const sequelize = new Sequelize('mydb', 'root', 'password', {
  host: 'localhost',
  port: 3306,
  dialect: 'mysql',
  logging: console.log,
  pool: {
    max: 10,
    min: 0,
    acquire: 30000,
    idle: 10000,
  },
  define: {
    charset: 'utf8mb4',
    collate: 'utf8mb4_unicode_ci',
  },
});

async function testConnection() {
  await sequelize.authenticate();
  console.log('MySQL connection established');
}
```

## Defining Models

```javascript
const { DataTypes, Model } = require('sequelize');

class Category extends Model {}
Category.init({
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  name: { type: DataTypes.STRING(100), allowNull: false, unique: true },
}, { sequelize, modelName: 'Category', tableName: 'categories' });

class Product extends Model {}
Product.init({
  id: { type: DataTypes.INTEGER, primaryKey: true, autoIncrement: true },
  categoryId: { type: DataTypes.INTEGER, allowNull: false, field: 'category_id' },
  name: { type: DataTypes.STRING(200), allowNull: false },
  price: { type: DataTypes.DECIMAL(10, 2), allowNull: false },
  stock: { type: DataTypes.INTEGER, defaultValue: 0 },
  active: { type: DataTypes.BOOLEAN, defaultValue: true },
}, {
  sequelize,
  modelName: 'Product',
  tableName: 'products',
  indexes: [{ fields: ['category_id', 'price'] }],
});

// Association
Category.hasMany(Product, { foreignKey: 'category_id' });
Product.belongsTo(Category, { foreignKey: 'category_id' });
```

## Syncing Models and CRUD

```javascript
const { Op } = require('sequelize');

// Sync schema (use migrations in production)
await sequelize.sync({ alter: true });

// Create
const category = await Category.create({ name: 'Electronics' });
const product = await Product.create({
  categoryId: category.id,
  name: 'Laptop',
  price: 999.99,
  stock: 50,
});

// Read with associations
const products = await Product.findAll({
  where: {
    price: { [Op.lte]: 1000 },
    stock: { [Op.gt]: 0 },
  },
  include: [{ model: Category }],
  order: [['price', 'ASC']],
  limit: 20,
  offset: 0,
});

// Update
await Product.update({ active: false }, { where: { stock: 0 } });

// Delete
await Product.destroy({ where: { price: { [Op.gt]: 50000 } } });
```

## Using Sequelize CLI for Migrations

Initialize Sequelize CLI config:

```bash
npx sequelize-cli init
```

Generate and run a migration:

```bash
npx sequelize-cli migration:generate --name create-products
npx sequelize-cli db:migrate
```

Migration file:

```javascript
module.exports = {
  up: async (queryInterface, Sequelize) => {
    await queryInterface.createTable('products', {
      id: { type: Sequelize.INTEGER, primaryKey: true, autoIncrement: true },
      name: { type: Sequelize.STRING(200), allowNull: false },
      price: { type: Sequelize.DECIMAL(10, 2), allowNull: false },
      stock: { type: Sequelize.INTEGER, defaultValue: 0 },
      created_at: { type: Sequelize.DATE, allowNull: false },
      updated_at: { type: Sequelize.DATE, allowNull: false },
    });
  },
  down: async (queryInterface) => {
    await queryInterface.dropTable('products');
  },
};
```

## Summary

Sequelize connects to MySQL via the `mysql2` driver. Define models with `Model.init()`, create associations with `hasMany/belongsTo`, and query using Sequelize's `Op` operators for complex conditions. Use Sequelize CLI for version-controlled migrations in production environments rather than `sequelize.sync()`.
