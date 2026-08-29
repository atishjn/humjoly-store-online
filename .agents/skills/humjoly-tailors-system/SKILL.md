---
name: humjoly-tailors-system
description: Comprehensive architecture, data models, AI extraction prompts, OMS order lifecycle, and customer storefront specification for Humjoly Tailors.
---

# Humjoly Tailors — System Architecture & Skill Memory

This document serves as the persistent architectural reference, system schema, AI parsing specification, and implementation guide for the **Humjoly Tailors Ecosystem**. It enables seamless system migration, multi-repository deployment, and scalable expansion.

---

## 1. System Overview

Humjoly Tailors operates a dual-product ecosystem sharing a unified backend schema:

```text
                                  ┌───────────────────────────┐
                                  │      YOUTUBE CHANNEL      │
                                  │   (Product Acquisition)   │
                                  └─────────────┬─────────────┘
                                                │
                        ┌───────────────────────┴───────────────────────┐
                        ▼                                               ▼
     ┌────────────────────────────────────┐          ┌────────────────────────────────────┐
     │      DIY CUSTOMER STOREFRONT       │          │          WHATSAPP DIRECT           │
     │      (Catalog & Lead Gen)          │          │      (Screenshots & Videos)       │
     └─────────────────┬──────────────────┘          └─────────────────┬──────────────────┘
                       │                                               │
                       │ Pre-filled Order                              │ Conversation & Proof
                       ▼                                               ▼
     ┌────────────────────────────────────────────────────────────────────────────────────┐
     │                      INTERNAL ORDER MANAGEMENT TOOL (OMS)                          │
     │                                                                                    │
     │  ┌───────────────────────┐   ┌──────────────────────┐   ┌───────────────────────┐  │
     │  │ AI Extraction Engine │   │ Manual Order Entry   │   │ Order Lifecycle       │  │
     │  └───────────┬───────────┘   └──────────┬───────────┘   └───────────┬───────────┘  │
     │              └──────────────────────────┼───────────────────────────┘              │
     │                                         ▼                                          │
     │                       ┌───────────────────────────────────┐                        │
     │                       │     Central Data Store / Ledger   │                        │
     │                       └─────────────────┬─────────────────┘                        │
     └─────────────────────────────────────────┼──────────────────────────────────────────┘
                                               │
                                               ▼
                                 ┌───────────────────────────┐
                                 │    GOOGLE SHEETS SYNC     │
                                 │   (Backup / Operations)   │
                                 └───────────────────────────┘
```

---

## 2. Core Schemas & Data Models

### A. Order Model
```typescript
interface Order {
  id: string; // e.g. "HT-1042"
  date: string; // ISO date or "29/07/2026"
  customer: {
    name: string;
    phone: string;
    whatsappNumber?: string;
    address: string;
    city?: string;
    pincode?: string;
  };
  videoAttribution: string; // e.g. "Video no 13", "Video no 18", "Shop visit"
  source: 'YouTube Storefront' | 'WhatsApp Direct' | 'Shop Visit';
  items: Array<{
    productId: string;
    productName: string;
    fabricDescription: string;
    quantity: number;
    unit: 'pc' | 'meter' | 'set';
    price: number;
    subtotal: number;
  }>;
  totalAmount: number;
  amountPaid: number;
  balanceAmount: number;
  paymentStatus: 'PAID' | 'PARTIAL' | 'PENDING';
  paymentMethod: 'UPI' | 'Bank Transfer' | 'Cash' | 'COD';
  orderStatus: 'DRAFT' | 'CONFIRMED' | 'PAYMENT_PENDING' | 'PROCESSING' | 'READY_TO_DISPATCH' | 'DISPATCHED' | 'DELIVERED' | 'CANCELLED';
  courierDetails?: {
    carrierName?: string;
    trackingNumber?: string;
    dispatchDate?: string;
  };
  aiExtractionMetadata?: {
    confidenceScore: number;
    sourceType: 'screenshot' | 'chat_export' | 'video';
    rawExtractedText?: string;
  };
  createdAt: string;
  updatedAt: string;
}
```

### B. Product Model
```typescript
interface Product {
  id: string;
  sku: string; // e.g. "HT-SOK-123"
  name: string; // e.g. "Soktas Premium Shirt Checks"
  brand: 'Soktas' | 'Raymond' | 'Siyaram' | 'Solino' | 'D&J' | 'Generic';
  category: 'Pure Cotton' | 'Pure Linen' | 'Cotton Linen' | 'Shirting' | 'Trouser' | 'Pant Shirt Combo';
  fabricType: string;
  color?: string;
  pattern?: string;
  sellingPrice: number; // In INR
  costPrice?: number;
  unit: 'pc' | 'meter' | 'suit length';
  currentStock: number;
  lowStockThreshold: number;
  featuredInVideos: string[]; // e.g. ["Video no 13", "Video no 18"]
  imageUrl: string;
  isActive: boolean;
}
```

### C. Inventory Ledger Model
```typescript
interface InventoryMovement {
  id: string;
  productId: string;
  productName: string;
  date: string;
  action: 'PURCHASE' | 'ORDER_DEDUCTION' | 'RETURN' | 'ADJUSTMENT' | 'DAMAGE';
  quantityChange: number; // +50 or -2
  balanceAfter: number;
  referenceId?: string; // Order ID or Invoice #
  notes?: string;
}
```

---

## 3. Google Sheets Integration Specification

The OMS maintains compatibility with Humjoly's original Google Sheet layout:

| Sheet Column | Target Order Field | Example Value |
| :--- | :--- | :--- |
| `IE` | Serial ID / Index | `1`, `2`, `3` |
| `NAME` | `customer.name` | `Shivkumar` |
| `PHONE NUMBER` | `customer.phone` | `9440975560` |
| `DATE` | `date` | `29/7/26` |
| `VIDEO NO. CODE` | `videoAttribution` | `Video no 8` |
| `FABRIC DESCRIPTION` | `items[].fabricDescription` | `Corduroy pant and cotton shirt` |
| `QUANTITY` | `items[].quantity` + unit | `2 pc` |
| `ADDRESS` | `customer.address` | `Avs book stall, 8/2/296 bellary road...` |
| `AMOUNT PAID` | `totalAmount` / `amountPaid` | `₹1895/-` |
| `PAYMENT STATUS` | `paymentStatus` | `Received` |

---

## 4. AI Screenshot & Video Order Extraction System

### Prompt Template for Gemini Multimodal Extraction
When an internal user uploads a WhatsApp screenshot or screen recording video:

```text
System Role: You are an expert AI data extraction engine for Humjoly Tailors OMS.
Task: Extract structured order information from WhatsApp chat screenshots or screen recordings.

Target JSON Schema:
{
  "customerName": "string or null",
  "phoneNumber": "string (10 digits) or null",
  "shippingAddress": "string or null",
  "videoReference": "string (e.g. 'Video no 13') or null",
  "items": [
    {
      "fabricDescription": "string",
      "quantity": "number",
      "unit": "pc | meter",
      "price": "number or null"
    }
  ],
  "totalAmount": "number or null",
  "amountPaid": "number or null",
  "paymentStatus": "PAID | PENDING | PARTIAL",
  "confidenceScore": "number (0 to 100)"
}

Rules:
1. Always return a DRAFT order. Never mark as confirmed directly.
2. If phone number is partial, extract whatever digits are visible.
3. Infer product match from catalog if fabric names match (e.g., 'Soktas checks' -> Soktas Premium Shirt Checks).
```

---

## 5. DIY Customer Storefront Design & WhatsApp Pre-fill Logic

### Pre-filled WhatsApp Query Format
When a customer clicks **"Order on WhatsApp"** on any product card or detail view, the application generates a pre-formatted `https://wa.me/` URI:

```text
https://wa.me/919040000000?text=Hi%20Humjoly%20Tailors%2C%20I%20would%20like%20to%20order%3A%0A%0A%F0%9F%A7%B5%20*Product*%3A%20Soktas%20Shirt%20Checks%0A%F0%9F%93%8F%20*Quantity*%3A%202%20pc%0A%F0%9F%92%B5%20*Total%20Price*%3A%20%E2%82%B91600%0A%F0%9F%8E%A5%20*Reference*%3A%20Video%20no%2013%0A%0APlease%20confirm%20availability%20and%20payment%20details.
```

Unencoded Text Structure:
> Hi Humjoly Tailors, I would like to order:
>
> 🧵 *Product*: Soktas Shirt Checks
> 📏 *Quantity*: 2 pc
> 💵 *Total Price*: ₹1600
> 🎥 *Reference*: Video no 13
>
> Please confirm availability and payment details.

---

## 6. Enterprise Migration & Scaling Guide

To migrate from the local prototype to a cloud production setup:
1. **Frontend**: Host the compiled Vite React build on Firebase Hosting or Vercel.
2. **Backend / Database**: Connect to Cloud Firestore with security rules for real-time order sync between multiple tailors/staff.
3. **Google Sheets Sync**: Use Google Sheets API v4 via a Cloud Function webhook triggering on Firestore `orders/{orderId}` creation.
4. **WhatsApp Cloud API Integration**: Replace manual screenshot uploading with direct webhooks from WhatsApp Business Cloud API.
