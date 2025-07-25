# 🏆 Backend Challenge 1 — Solution Documentation: Express API & Database Foundation

This solution implements the foundational API for **CollabNote**, a collaborative note-taking app. The backend is built using **Express.js**, **Prisma ORM**, and **PostgreSQL**, with full CRUD operations and frontend‑ready JSON responses.

---

## 📦 Project Structure

```
project/backend/
├── prisma/
│   └── schema.prisma
├── src/
│   ├── Controllers/
│   │   └── notesController.js
│   ├── generated/
│   ├── routes/
│   │   └── notes.js
│   ├── utils/
│   │   ├── errorHandler.js
│   │   ├── noteFormatter.js
│   │   ├── noteValidator.js
│   │   └── prisma.js
│   └── app.js
├── .env
├── .env.example
├── .gitignore
├── package.json
├── package-lock.json
└── README.md
```

---

## ✅ Goals Achieved

### 📓️ Task 1: Database Setup with Prisma

- ✅ Installed Prisma & dependencies
- ✅ Set up `.env` with PostgreSQL `DATABASE_URL`
- ✅ Defined the `Note` model in `prisma/schema.prisma`
- ✅ Generated Prisma client and pushed schema to DB

---

## ⚙️ Setup Breakdown

### 1⃣ PostgreSQL & Prisma

- Installed PostgreSQL via the [official installer](https://www.postgresql.org/download/)
- Default user: `postgres`, custom password set during setup
- Verified existing users using:

```bash
psql -U postgres
\du
```

### 2⃣ Installed Dependencies

```bash
npm install prisma @prisma/client cors dotenv
npx prisma init
```

### 3⃣ Environment Configuration

Created `.env` (based on `.env.example`):

```env
DATABASE_URL="postgresql://postgres:yourpassword@localhost:5432/collabnote_db"
```

### 4⃣ Prisma Schema Definition

`prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
  output   = "../src/generated/prisma"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model Note {
  id         Int      @id @default(autoincrement())
  title      String   @db.VarChar(100)  
  content    String   @db.VarChar(1000) 
  authorName String   @default("Unknown") @db.VarChar(100) 
  isPublic   Boolean  @default(true)
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt
}
```

Then generated the client and synced the DB:

```bash
npx prisma generate
npx prisma db push
```

---

## 🔌 Express Server Setup

All setup and middleware are configured inside `src/app.js`, and the server waits for the database to connect before starting.

### ✅ `app.js` Code Highlights

```js
import express from "express";
import cors from "cors";
import dotenv from "dotenv";
import notesRouter from './routes/notes.js';
import { errorHandler } from "./utils/errorHandler.js";
import { prisma } from "./utils/prisma.js";

dotenv.config();

const PORT = process.env.PORT || 5000;
const app = express();

// Middleware
app.use(
  cors({
    origin: "http://localhost:5173", // Vite frontend URL
    credentials: true,
  }),
);
app.use(express.json());

// Routes
app.get("/", (_, res) => {
  res.json({ message: "CollabNote API is running!" });
});

app.use("/api/notes", notesRouter);

// Global error handler
app.use(errorHandler);

// Start server with DB connection handling
const startServer = async () => {
  try {
    await prisma.$connect();
    app.listen(PORT, () => {
      console.log(`🚀 Server running at http://localhost:${PORT}`);
    });
  } catch (error) {
    console.error('❌ Failed to connect to the database');
    console.error(error);
    process.exit(1); // Stop the process if DB fails
  }
};

startServer();
```

---

## 🧹 Notes Controller Overview

All logic for handling notes — including retrieving, creating, updating, and deleting — is encapsulated in `notesController.js` under the `src/Controllers/` directory.

### Key Highlights

- 📅 `` — supports search, sort, pagination
- 📤 `` — validates and creates a new note
- 🔍 `` — fetches a specific note by ID
- ✏️ `` — updates note fields selectively
- ❌ `` — removes a note after existence check

Each function is wrapped in `try/catch` and forwards errors to the global middleware using `next()`.

---

## 🧾 Note Formatting Utility

Located in `utils/noteFormatter.js`, this ensures that notes are returned in a frontend-friendly and consistent structure.

- ✅ `formatNote(note)` — Formats a single note object with standardized structure and ISO date strings.
- ✅ `formatNotes(notes)` — Maps over an array of notes and formats each using `formatNote()`.

This keeps response formatting centralized and reusable.

---

## 🌐 Routes and Middleware Integration

Defined in `src/routes/notes.js`, all routes follow REST principles and use middleware for validation:

```js
import express from 'express';
import {
  getNotes,
  getNoteById,
  createNote,
  updateNote,
  deleteNote,
} from '../Controllers/notesController.js';

import {
  createNoteValidator,
  updateNoteValidator,
  validateNoteId,
} from '../utils/noteValidator.js';

const router = express.Router();

router.get('/', getNotes);
router.get('/:id', validateNoteId, getNoteById);
router.post('/', createNoteValidator, createNote);
router.put('/:id', validateNoteId, updateNoteValidator, updateNote);
router.delete('/:id', validateNoteId, deleteNote);

export default router;
```

---

## 📃 Note Validation with Zod

Validation logic is handled using `Zod`, located in `utils/noteValidator.js`. It ensures clean and safe input for both creation and update operations.

### 📁 Create Note Validation

- Title: 3–100 characters
- Content: 10–1000 characters
- Author: optional, max 100 characters
- `isPublic`: optional boolean

If validation fails, a structured 400 error is returned with detailed field messages.

### ✏️ Update Note Validation

- Accepts **any subset** of the fields (partial update)
- Requires **at least one** valid field to update
- Also validates `id` via the `validateNoteId` middleware (must be a positive integer)

### ⚖️ Middleware Functions

- `createNoteValidator`
- `updateNoteValidator`
- `validateNoteId`

These middleware functions are used directly in the route definitions to ensure clean request handling.

---

## ❗ Error Handling

Error handling is centralized via a global middleware (`utils/errorHandler.js`). It supports:

### 🔍 Prisma-Specific Errors

- **P2002** — Unique constraint violation (e.g., duplicate title)
- **P2025** — Record not found

Returns custom responses like:

```json
{
  "success": false,
  "message": "Note not found",
  "code": "P2025"
}
```

### 🛠 Custom Application Errors

Custom errors are thrown using a helper:

```js
httpError('Note not found', 404, 'NOT_FOUND')
```

And caught in the error middleware to return:

```json
{
  "success": false,
  "message": "Note not found",
  "code": "NOT_FOUND"
}
```

### ⚠️ Unknown Errors

All unexpected errors are caught and return:

```json
{
  "success": false,
  "message": "Internal server error"
}
```

This provides a consistent and informative error format across the entire API.

---

## 🔍 Advanced Query Features

### 🔎 Search by Title or Content

- You can search notes by including a `search` query param.
- Example: `GET /api/notes?search=meeting`

### 🔃 Sorting Options

- You can sort notes using the `sort` query param:
  - `newest` (default)
  - `oldest`
  - `title_asc`
  - `title_desc`
- Example: `GET /api/notes?sort=title_desc`

### 📄 Pagination Support

- Use `page` and `limit` query params for pagination.
- Default: `page=1`, `limit=3`
- Example: `GET /api/notes?page=2&limit=5`

### 🔗 Combined Example

```http
GET /api/notes?search=design&sort=title_asc&page=2&limit=4
```

This will return:

- Notes that match "design" in title/content
- Sorted alphabetically by title
- Paginated to page 2 with 4 notes per page

---

## ✅ Manual Test Scenarios with Screenshots

Screenshots of each test are saved under `./screenshots/`

| ✅ Test Case           | Description                                                   | Screenshot                                               |
| --------------------- | ------------------------------------------------------------- | -------------------------------------------------------- |
| 📄 **Create Note**    | Can create notes with valid data                              | ![Create Note](./screenshots/cantCreateNoteWithoutRequiredFields.png)      |
| ❌ **Validation**      | Cannot create notes without required fields                   | ![Invalid Create](./screenshots/cantCreateNoteWithoutRequiredFields.png) |
| 📋 **Fetch All**      | Can retrieve all notes (matching frontend sample data format) | ![All Notes](./screenshots/canGetAllNotes.png)            |
| 🔎 **Fetch One**      | Can retrieve specific note by ID                              | ![Note by ID](./screenshots/canRetrieveDataWithId.png)          |
| 🚫 **404 Not Found**  | Returns 404 for non-existent note IDs                         | ![404 Not Found](./screenshots/return404ForNoneExistingNote.png)       |
| ✏️ **Update**         | Can update existing notes                                     | ![Update Note](./screenshots/updateExistingNote.png)            |
| ❌ **Delete**          | Can delete notes                                              | ![Delete Note](./screenshots/CanDeleteNote.png)            |
| 🌐 **CORS**           | CORS allows frontend connections                              | ![CORS](./screenshots/CORSallowsfrontendconnections.png)                  |
| ⚠️ **Error Handling** | Proper error messages for invalid requests                    | ![Errors](./screenshots/Propererrormessagesforinvalidrequests.png)              |
| 🔌 DB Failure      | Gracefully handles database connection errors                 |

### 🔌 Handles Database Connection Errors

✅ Description: If the database connection fails (e.g., due to invalid credentials or unreachable server), the app logs a clear error message and exits gracefully to avoid running in a broken state.

📸 Screenshot:  
![DB Error](./screenshots/db-connection-error.png)

