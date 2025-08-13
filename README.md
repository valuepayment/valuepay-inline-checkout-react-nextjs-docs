# ValuePay Checkout Integration Guide for React/Next.js

Welcome to the ValuePay integration guide! This documentation will help you integrate ValuePay's payment gateway into your React or Next.js application to accept payments from your customers.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Basic Integration](#basic-integration)
- [Advanced Features](#advanced-features)
- [API Reference](#api-reference)
- [Best Practices](#best-practices)
- [Troubleshooting](#troubleshooting)

## Prerequisites

Before you begin, make sure you have:

- A ValuePay merchant account
- Your ValuePay public key
- A React or Next.js application
- Node.js version 16 or higher

## Installation

### 1. No SDK Installation Required

ValuePay uses inline script integration, so no npm package installation is needed. The ValuePay JavaScript library is loaded dynamically.

### 2. Environment Setup

Create a `.env.local` file in your project root:

```env
NEXT_PUBLIC_VALUEPAY_PUBLIC_KEY=KP_your_public_key_here
```

## Quick Start

### 1. Create a Script Loading Hook

First, create a custom hook to load the ValuePay script:

```tsx
// app/hooks/useScripts.ts
import { useEffect } from "react";

const useScript = (src: string) => {
  useEffect(() => {
    if (!src) return;
    const script = document.createElement("script");
    script.src = src;
    script.async = true;
    document.body.appendChild(script);
    return () => {
      document.body.removeChild(script);
    };
  }, [src]);
};

export default useScript;
```

### 2. Add TypeScript Declarations

Add the ValuePay global type declaration:

```tsx
// Add this to your component or a global types file
declare global {
  interface Window {
    ValuepayCheckout: (input: unknown) => void;
  }
}
```

### 3. Create a Payment Component

```tsx
// components/CheckoutForm.tsx
"use client";

import { MouseEvent, useState } from "react";
import useScript from "../hooks/useScripts";
import { makeId } from "../helper/makeid";

declare global {
  interface Window {
    ValuepayCheckout: (input: unknown) => void;
  }
}

export default function CheckoutForm() {
  const [amount, setAmount] = useState<number | null>(null);

  // Load the ValuePay script
  useScript(
    "https://www.valuepayng.com/js/vp-v1.js?v=1.000436060005554300660040066"
  );

  const handleSubmit = (e: MouseEvent) => {
    if (
      typeof window === "undefined" ||
      window.ValuepayCheckout === undefined
    ) {
      return;
    }

    e.preventDefault();

    if (amount && !isNaN(amount) && amount > 0) {
      const paymentData = {
        public_key: process.env.NEXT_PUBLIC_VALUEPAY_PUBLIC_KEY,
        transactionRef: makeId(15),
        amount: amount,
        currency: "NGN",
        channels: ["card", "transfer", "qrcode", "ussd"],
        redirect_url: "https://your-domain.com/payment/success",
        customer: {
          email: "customer@example.com",
          firstName: "John",
          lastName: "Doe",
        },
        onclose: function () {
          // Handle modal close
          console.log("Payment modal closed");
        },
        customisedCheckout: {
          title: "Complete Your Payment",
          description:
            "Thank you for your purchase. Please complete your payment to proceed.",
          logoLink: "https://your-domain.com/logo.png",
        },
      };

      window.ValuepayCheckout(paymentData);
    } else {
      alert("Please enter a valid amount");
    }
  };

  return (
    <div className="max-w-md mx-auto p-6 bg-white rounded-lg shadow-md">
      <h2 className="text-2xl font-bold mb-4">Complete Your Payment</h2>

      <div className="mb-4">
        <label className="block text-sm font-medium mb-2">Amount (NGN)</label>
        <input
          type="number"
          value={amount != null && amount > 0 ? amount : ""}
          onChange={(e) => setAmount(Number(e.target.value))}
          className="w-full px-3 py-2 border border-gray-300 rounded-md focus:outline-none focus:ring-2 focus:ring-blue-500"
          placeholder="0.00"
          step="0.01"
          min="0"
        />
      </div>

      <button
        onClick={handleSubmit}
        disabled={!amount || amount <= 0}
        className="w-full bg-blue-600 text-white py-2 px-4 rounded-md hover:bg-blue-700 disabled:opacity-50 disabled:cursor-not-allowed"
      >
        Pay Now
      </button>
    </div>
  );
}
```

### 4. Create a Helper Function for Transaction References

```tsx
// app/helper/makeid.ts
export const makeId = (length = 24) => {
  let text = "";
  const possible =
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

  for (let i = 0; i < length; i++) {
    text += possible.charAt(Math.floor(Math.random() * possible.length));
  }

  return text;
};
```

### 5. Use the Component in Your Page

```tsx
// app/page.tsx
import CheckoutForm from "@/components/CheckoutForm";

export default function Home() {
  return (
    <div className="min-h-screen bg-gray-50 py-12">
      <div className="container mx-auto">
        <h1 className="text-3xl font-bold text-center mb-8">
          ValuePay Integration Demo
        </h1>
        <CheckoutForm />
      </div>
    </div>
  );
}
```

## Basic Integration

### Payment Object Structure

```typescript
interface PaymentData {
  public_key: string; // Your ValuePay public key
  transactionRef: string; // Unique transaction reference
  amount: number; // Payment amount
  currency: string; // Currency code (e.g., 'NGN', 'USD')
  channels: string[]; // Payment channels
  redirect_url: string; // URL to redirect after payment
  customer: {
    email: string; // Customer email
    firstName: string; // Customer first name
    lastName: string; // Customer last name
  };
  onclose?: () => void; // Callback when modal closes
  customisedCheckout?: {
    title: string; // Custom checkout title
    description: string; // Custom checkout description
    logoLink: string; // Your logo URL
  };
  callback?: (data: any) => void; // General callback
  onsuccess?: (data: any) => void; // Success callback
}
```

### Available Payment Channels

```typescript
const channels = [
  "card", // Credit/Debit cards
  "transfer", // Bank transfers
  "qrcode", // QR code payments
  "ussd", // USSD payments
];
```

### Handling Payment Responses

```tsx
// components/PaymentHandler.tsx
"use client";

import { useEffect } from "react";

export default function PaymentHandler() {
  useEffect(() => {
    // Listen for payment success
    const handlePaymentSuccess = (event: MessageEvent) => {
      if (event.data.type === "valuepay_payment_success") {
        console.log("Payment successful:", event.data);
        // Handle successful payment
        // Redirect to success page
        // Update order status
      }
    };

    // Listen for payment failure
    const handlePaymentFailure = (event: MessageEvent) => {
      if (event.data.type === "valuepay_payment_failed") {
        console.log("Payment failed:", event.data);
        // Handle failed payment
        // Show error message
      }
    };

    window.addEventListener("message", handlePaymentSuccess);
    window.addEventListener("message", handlePaymentFailure);

    return () => {
      window.removeEventListener("message", handlePaymentSuccess);
      window.removeEventListener("message", handlePaymentFailure);
    };
  }, []);

  return null;
}
```

## Advanced Features

### 1. Custom Styling and Branding

```tsx
const paymentData = {
  // ... other payment data
  customisedCheckout: {
    title: "Complete Your Order",
    description:
      "You're just one step away from completing your purchase. Please provide your payment details to continue.",
    logoLink: "https://your-domain.com/assets/logo.png",
  },
};
```

### 2. Multiple Payment Channels

```tsx
// Enable specific payment channels
const channels = ["card", "transfer"]; // Only card and transfer

// Or enable all channels
const allChannels = ["card", "transfer", "qrcode", "ussd"];
```

### 3. Callback Functions

```tsx
const paymentData = {
  // ... other payment data
  callback: function (data: any) {
    console.log("Payment callback:", data);
    // Handle any payment event
  },
  onsuccess: function (data: any) {
    console.log("Payment successful:", data);
    // Handle successful payment
    window.location.href = "/payment/success";
  },
  onclose: function () {
    console.log("Payment modal closed");
    // Handle modal close
  },
};
```

## API Reference

### ValuepayCheckout Function

```typescript
window.ValuepayCheckout(paymentData: PaymentData): void
```

### PaymentData Interface

```typescript
interface PaymentData {
  public_key: string; // Your ValuePay public key
  transactionRef: string; // Unique transaction reference
  amount: number; // Payment amount
  currency: string; // Currency code
  channels: string[]; // Payment channels
  redirect_url: string; // Redirect URL after payment
  customer: {
    email: string; // Customer email
    firstName: string; // Customer first name
    lastName: string; // Customer last name
  };
  onclose?: () => void; // Modal close callback
  customisedCheckout?: {
    title: string; // Custom title
    description: string; // Custom description
    logoLink: string; // Logo URL
  };
  callback?: (data: any) => void; // General callback
  onsuccess?: (data: any) => void; // Success callback
}
```

### Payment Response

```typescript
interface PaymentResponse {
  transactionRef: string;
  status: "success" | "failed" | "pending";
  amount: number;
  currency: string;
  customer: {
    email: string;
    firstName: string;
    lastName: string;
  };
  paymentMethod: string;
  timestamp: string;
}
```

## Best Practices

### 1. Security

- **Never expose your secret key** in client-side code
- Always verify webhook signatures
- Use HTTPS in production
- Validate all input data
- Generate unique transaction references

### 3. User Experience

- Show loading states during script loading
- Provide clear error messages
- Implement proper form validation
- Use appropriate success/error pages
- Handle script loading failures gracefully

### 4. Transaction Reference Generation

```tsx
// Generate unique transaction references
const generateTransactionRef = () => {
  const timestamp = Date.now();
  const random = Math.random().toString(36).substring(2, 15);
  return `TXN_${timestamp}_${random}`;
};
```

## Troubleshooting

### Common Issues

1. **"ValuepayCheckout is not defined" Error**

   - Ensure the script is loaded before calling the function
   - Check if the script URL is accessible
   - Verify the script is loaded in the correct order

2. **Script Not Loading**

   - Check network connectivity
   - Verify the script URL is correct
   - Ensure no Content Security Policy (CSP) blocking the script

3. **Payment Modal Not Opening**

   - Check browser console for JavaScript errors
   - Verify all required fields are provided
   - Ensure the public key is valid

4. **Transaction Reference Errors**
   - Ensure transaction references are unique
   - Check if the reference format is correct
   - Verify no special characters in the reference

### Support

If you encounter issues not covered in this guide:

1. Check the [ValuePay API Documentation](https://docs.valuepayng.com)
2. Review your browser's developer console for errors
3. Contact ValuePay support with your merchant ID and error details

## Example Project

You can find a complete working example in the `payment-test` folder of this repository. The example demonstrates:

- Script loading with custom hook
- Basic payment integration
- Custom styling and branding
- Error handling
- Transaction reference generation

To run the example:

```bash
cd payment-test
npm install
npm run dev
```

Visit `http://localhost:3000` to see the integration in action.

---

**Need Help?** Contact our support team at support@valuepayng.com or visit our [developer portal](https://developers.valuepayng.com) for additional resources.
