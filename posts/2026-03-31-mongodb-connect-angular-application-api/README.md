# How to Connect MongoDB to an Angular Application via API

Author: [nawazdhandala](https://www.github.com/nawazdhandala)

Tags: MongoDB, Angular, API, Node.js, TypeScript

Description: Learn how to connect MongoDB to an Angular application through a Node.js Express API using Angular's HttpClient and environment-based configuration.

---

## Architecture

Angular runs in the browser and cannot connect to MongoDB directly. You need a Node.js backend API that Angular communicates with via HTTP. MongoDB credentials stay server-side and never reach the browser.

```text
Angular (browser)  -->  Express REST API  -->  MongoDB
```

## Setting Up the Node.js API

```bash
mkdir mongo-api && cd mongo-api
npm init -y
npm install express mongoose cors dotenv
```

```javascript
// server.js
const express = require('express');
const mongoose = require('mongoose');
const cors = require('cors');
require('dotenv').config();

const app = express();
app.use(cors({ origin: process.env.CLIENT_URL }));
app.use(express.json());

mongoose.connect(process.env.MONGODB_URI)
  .then(() => console.log('Connected to MongoDB'));

const ItemSchema = new mongoose.Schema({
  name: { type: String, required: true },
  description: String,
  createdAt: { type: Date, default: Date.now }
});

const Item = mongoose.model('Item', ItemSchema);

app.get('/api/items', async (req, res) => {
  const items = await Item.find().sort({ createdAt: -1 });
  res.json(items);
});

app.post('/api/items', async (req, res) => {
  try {
    const item = await Item.create(req.body);
    res.status(201).json(item);
  } catch (err) {
    res.status(400).json({ error: err.message });
  }
});

app.delete('/api/items/:id', async (req, res) => {
  await Item.findByIdAndDelete(req.params.id);
  res.status(204).end();
});

app.listen(process.env.PORT || 5000);
```

```text
MONGODB_URI=mongodb://localhost:27017/angularapp
CLIENT_URL=http://localhost:4200
```

## Creating the Angular Application

```bash
ng new frontend --routing --style=scss
cd frontend
```

Configure the API URL in Angular environments:

```typescript
// src/environments/environment.ts
export const environment = {
  production: false,
  apiUrl: 'http://localhost:5000'
};
```

```typescript
// src/environments/environment.prod.ts
export const environment = {
  production: true,
  apiUrl: 'https://api.yourdomain.com'
};
```

## Creating the API Service

```typescript
// src/app/services/item.service.ts
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../environments/environment';

export interface Item {
  _id?: string;
  name: string;
  description?: string;
  createdAt?: string;
}

@Injectable({ providedIn: 'root' })
export class ItemService {
  private apiUrl = `${environment.apiUrl}/api/items`;

  constructor(private http: HttpClient) {}

  getItems(): Observable<Item[]> {
    return this.http.get<Item[]>(this.apiUrl);
  }

  createItem(item: Partial<Item>): Observable<Item> {
    return this.http.post<Item>(this.apiUrl, item);
  }

  deleteItem(id: string): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/${id}`);
  }
}
```

## Using the Service in a Component

```typescript
// src/app/components/item-list/item-list.component.ts
import { Component, OnInit } from '@angular/core';
import { ItemService, Item } from '../../services/item.service';

@Component({
  selector: 'app-item-list',
  template: `
    <div *ngIf="loading">Loading...</div>
    <div *ngIf="error" class="error">{{ error }}</div>
    <ul>
      <li *ngFor="let item of items">
        {{ item.name }}
        <button (click)="delete(item._id!)">Delete</button>
      </li>
    </ul>
  `
})
export class ItemListComponent implements OnInit {
  items: Item[] = [];
  loading = true;
  error = '';

  constructor(private itemService: ItemService) {}

  ngOnInit(): void {
    this.itemService.getItems().subscribe({
      next: (data) => { this.items = data; this.loading = false; },
      error: (err) => { this.error = err.message; this.loading = false; }
    });
  }

  delete(id: string): void {
    this.itemService.deleteItem(id).subscribe(() => {
      this.items = this.items.filter(i => i._id !== id);
    });
  }
}
```

Register `HttpClientModule` in your `AppModule`:

```typescript
// src/app/app.module.ts
import { HttpClientModule } from '@angular/common/http';

@NgModule({
  imports: [HttpClientModule, ...]
})
export class AppModule {}
```

## CORS and Proxy Configuration for Development

To avoid CORS issues during development, configure Angular's proxy:

```json
// proxy.conf.json
{
  "/api": {
    "target": "http://localhost:5000",
    "secure": false,
    "changeOrigin": true
  }
}
```

```json
// angular.json (in serve options)
"proxyConfig": "proxy.conf.json"
```

Then in development, Angular proxies `/api` requests to the Express server, eliminating CORS headers entirely.

## Summary

Connecting MongoDB to Angular follows the same API pattern as other frameworks: Angular's `HttpClient` calls a Node.js/Express REST API that uses Mongoose to query MongoDB. Use Angular's `environment.ts` files for API URL configuration, create a typed service with Observable return types, and configure a dev proxy to simplify local development. Never include MongoDB connection details in Angular environment files - those values are bundled into the browser-delivered JavaScript.
