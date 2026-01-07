# PFPI Comment Tracker

A searchable, public library of PFPI's government comments and testimony. This application allows the public to browse past documents while providing a secure, restricted "Admin Panel" for PFPI staff to add new entries directly to a Google Sheet.

## ⚡ Features

* **Public Library:** Fast, read-only view of comments using a card-based layout.
* **Search & Filter:** Real-time filtering by Topic, Target Agency, and Jurisdiction.
* **Admin Portal:** Secure Google Sign-In (OAuth 2.0) allows authorized staff to append new rows.
* **Hybrid Architecture:**
* **Reads** from a public CSV endpoint (fast, no API limits).
* **Writes** via the Google Sheets API (secure, authenticated).



## 🛠️ Setup & Installation

### 1. Clone and Install

```bash
git clone https://github.com/YOUR_USERNAME/pfpi-tracker.git
cd pfpi-tracker
npm install

```

### 2. Configure Environment Variables

Create a file named `.env.local` in the root directory. **Do not commit this file.**

```properties
# .env.local

# 1. The ID of your Google Sheet (from the URL)
VITE_GOOGLE_SHEET_ID=your_sheet_id_here

# 2. The OAuth 2.0 Client ID (from Google Cloud Console)
VITE_GOOGLE_CLIENT_ID=your_client_id_here.apps.googleusercontent.com

# 3. The "Publish to Web" CSV link (File > Share > Publish to web)
VITE_CSV_URL=https://docs.google.com/spreadsheets/d/e/.../pub?output=csv

```

### 3. Run Locally

```bash
npm run dev

```

## 📊 Google Sheet Structure

The application expects a Google Sheet with the following **exact headers** in Row 1:

| Column Header | Description |
| --- | --- |
| **Date** | Format: YYYY-MM-DD |
| **Type** | e.g., Comments, Joint Comments, Sign On |
| **Author** | e.g., PFPI, Center for Biological Diversity |
| **Jurisdiction** | e.g., United States, US - California |
| **Target** | e.g., EPA, BLM, PUC |
| **Topic** | Comma-separated list (e.g., "Bioenergy, Pollution") |
| **Summary** | Paragraph describing the document |
| **Keywords** | Comma-separated tags |
| **File Link** | URL to the PDF |
| **2nd Link** | (Hidden in app) |
| **3rd Link** | (Hidden in app) |
| **Notes** | (Hidden in app) |

> **Note:** The app splits `Topic` and `Keywords` by comma to create badges/tags.

## ☁️ Deployment (Netlify)

This project is optimized for Netlify.

1. Connect your GitHub repository to Netlify.
2. Set the **Build Command** to `npm run build`.
3. Set the **Publish Directory** to `dist`.
4. **Crucial:** Go to **Site Settings > Environment Variables** and add the three keys from your `.env.local` file (`VITE_GOOGLE_SHEET_ID`, etc.).

## 🔐 Authentication Setup

To make the "Admin" tab work:

1. Go to the [Google Cloud Console](https://console.cloud.google.com/).
2. Create a project and enable the **Google Sheets API**.
3. Create an **OAuth 2.0 Client ID** (Web Application).
4. Add your domains to **Authorized JavaScript origins**:
* `http://localhost:5173` (for local dev)
* `https://your-site-name.netlify.app` (for production)



## 📝 Tech Stack

* **Framework:** Vue 3 (Composition API)
* **Build Tool:** Vite
* **Styling:** Custom CSS (Desktop-first responsive grid)
* **State:** Vue Reactivity System (no external store needed)