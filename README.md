# task-03-database-migrations

require("dotenv").config();

const express = require("express");
const { z } = require("zod");
const pool = require("./db");

const app = express();
app.use(express.json());

const userSchema = z.object({
  name: z.string().trim().min(2).max(100),
  email: z.string().trim().email().max(255),
  age: z.number().int().min(18),
});

class AppError extends Error {
  constructor(message, statusCode, details) {
    super(message);
    this.statusCode = statusCode;
    this.details = details;
  }
}

// Health check
app.get("/", (req, res) => {
  res.json({
    success: true,
    message: "Database API is running",
  });
});

// Get all users
app.get("/users", async (req, res, next) => {
  try {
    const result = await pool.query(
      "SELECT id, name, email, age, created_at FROM users ORDER BY id"
    );

    res.json({
      success: true,
      data: result.rows,
    });
  } catch (error) {
    next(error);
  }
});

// Create a user
app.post("/users", async (req, res, next) => {
  try {
    const parsed = userSchema.safeParse(req.body);

    if (!parsed.success) {
      throw new AppError(
        "Validation failed",
        400,
        parsed.error.issues.map((issue) => ({
          field: issue.path.join("."),
          message: issue.message,
        }))
      );
    }

    const { name, email, age } = parsed.data;

    const result = await pool.query(
      `INSERT INTO users (name, email, age)
       VALUES ($1, $2, $3)
       RETURNING id, name, email, age, created_at`,
      [name, email, age]
    );

    res.status(201).json({
      success: true,
      data: result.rows[0],
    });
  } catch (error) {
    next(error);
  }
});

// Delete a user
app.delete("/users/:id", async (req, res, next) => {
  try {
    if (!/^[1-9]\d*$/.test(req.params.id)) {
      throw new AppError("Invalid user ID", 400);
    }

    const result = await pool.query(
      "DELETE FROM users WHERE id = $1 RETURNING id",
      [Number(req.params.id)]
    );

    if (result.rowCount === 0) {
      throw new AppError("User not found", 404);
    }

    res.json({
      success: true,
      message: "User deleted successfully",
    });
  } catch (error) {
    next(error);
  }
});

// Unknown routes
app.use((req, res, next) => {
  next(new AppError("Route not found", 404));
});

// Central error handler
app.use((err, req, res, next) => {
  if (res.headersSent) {
    return next(err);
  }

  if (err instanceof SyntaxError && err.status === 400 && "body" in err) {
    return res.status(400).json({
      success: false,
      error: "Invalid JSON request body",
    });
  }

  let statusCode = err.statusCode || 500;
  let message = statusCode >= 500
    ? "Internal server error"
    : err.message;

  if (err.code === "23505") {
    statusCode = 409;
    message = "A user with this email already exists";
  }

  if (statusCode >= 500) {
    console.error(err.message);
  }

  res.status(statusCode).json({
    success: false,
    error: message,
    ...(err.details && { details: err.details }),
  });
});

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
