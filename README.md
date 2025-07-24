📁 Repository Structure
game-panel/
├── backend/
│   ├── src/
│   ├── prisma/
│   ├── Dockerfile
│   └── docker-compose.yml
├── frontend/
│   ├── pages/
│   ├── components/
│   ├── styles/
│   ├── Dockerfile
│   └── .env.local.example
├── .github/
│   └── workflows/
│       └── deploy.yml
└── README.md
________________________________________
✅ 1. Backend – Production Ready
📁 backend/prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        Int      @id @default(autoincrement())
  email     String   @unique
  password  String
  createdAt DateTime @default(now())
  servers   Server[]
  payments  Payment[]
}

model Server {
  id          Int      @id @default(autoincrement())
  userId      Int
  name        String
  game        String
  packageType Int
  location    String
  status      String   @default("installing")
  createdAt   DateTime @default(now())
  user        User     @relation(fields: [userId], references: [id])
}

model Payment {
  id                  Int    @id @default(autoincrement())
  userId              Int
  serverId            Int
  stripePaymentIntentId String
  amount              Int
  currency            String
  status              String
  user                User   @relation(fields: [userId], references: [id])
}
________________________________________
📁 backend/src/index.ts

import express from "express";
import cors from "cors";
import helmet from "helmet";
import morgan from "morgan";
import dotenv from "dotenv";
import { PrismaClient } from "@prisma/client";
import Stripe from "stripe";
import jwt from "jsonwebtoken";
import bcrypt from "bcrypt";
import { body, validationResult } from "express-validator";
import { WebSocketServer } from "ws";
import http from "http";

dotenv.config();
const app = express();
const server = http.createServer(app);
const prisma = new PrismaClient();
const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!);
const wss = new WebSocketServer({ server });

app.use(helmet());
app.use(cors({ origin: process.env.FRONTEND_URL }));
app.use(morgan("combined"));
app.use(express.json());

const authenticate = (req: any, res: any, next: any) => {
  const token = req.headers.authorization?.split(" ")[1];
  if (!token) return res.status(401).json({ error: "No token" });
  try {
    (req as any).user = jwt.verify(token, process.env.JWT_SECRET!);
    next();
  } catch {
    res.status(401).json({ error: "Invalid token" });
  }
};

// WebSocket – real-time status
wss.on("connection", (ws) => {
  ws.on("message", async (data) => {
    const { serverId } = JSON.parse(data.toString());
    const server = await prisma.server.findUnique({ where: { id: serverId } });
    ws.send(JSON.stringify({ type: "status", server }));
  });
});

// Routes
app.post("/api/auth/register", [
  body("email").isEmail(),
  body("password").isLength({ min: 8 })
], async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  const { email, password } = req.body;
  const hashed = await bcrypt.hash(password, 12);
  const user = await prisma.user.create({ data: { email, password: hashed } });
  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET!, { expiresIn: "7d" });
  res.json({ token });
});

app.post("/api/auth/login", async (req, res) => {
  const { email, password } = req.body;
  const user = await prisma.user.findUnique({ where: { email } });
  if (!user || !(await bcrypt.compare(password, user.password))) return res.status(401).json({ error: "Invalid credentials" });
  const token = jwt.sign({ id: user.id }, process.env.JWT_SECRET!, { expiresIn: "7d" });
  res.json({ token });
});

app.get("/api/servers", authenticate, async (req, res) => {
  const servers = await prisma.server.findMany({ where: { userId: (req as any).user.id } });
  res.json({ servers });
});

app.post("/api/servers", authenticate, [
  body("game").isIn(["minecraft", "rust", "csgo"]),
  body("location").isIn(["fra", "nyc", "lon", "sgp"]),
  body("packageType").isInt({ min: 1, max: 3 })
], async (req, res) => {
  const errors = validationResult(req);
  if (!errors.isEmpty()) return res.status(400).json({ errors: errors.array() });

  const { game, location, packageType } = req.body;
  const packages = { 1: 300, 2: 500, 3: 900 };
  const amount = packages[packageType as keyof typeof packages];

  const server = await prisma.server.create({
    data: {
      userId: (req as any).user.id,
      name: `${game}-${Date.now()}`,
      game,
      location,
      packageType
    }
  });

  const paymentIntent = await stripe.paymentIntents.create({
    amount,
    currency: "usd",
    metadata: { serverId: server.id }
  });

  await prisma.payment.create({
    data: {
      userId: (req as any).user.id,
      serverId: server.id,
      stripePaymentIntentId: paymentIntent.id,
      amount,
      currency: "usd",
      status: "requires_payment_method"
    }
  });

  res.json({ server, clientSecret: paymentIntent.client_secret });
});

app.post("/stripe/webhook", express.raw({ type: "application/json" }), (req, res) => {
  const sig = req.headers["stripe-signature"]!;
  const event = stripe.webhooks.constructEvent(req.body, sig, process.env.STRIPE_WEBHOOK_SECRET!);

  if (event.type === "payment_intent.succeeded") {
    const { serverId } = event.data.object.metadata;
    prisma.server.update({ where: { id: Number(serverId) }, data: { status: "active" } });
  }

  res.json({ received: true });
});

server.listen(3000, () => console.log("API running on :3000"));
________________________________________
✅ 2. Frontend – Next.js Production Ready
📁 frontend/pages/_app.tsx

import "@/styles/globals.css";
import { AuthProvider } from "@/contexts/AuthContext";
import type { AppProps } from "next/app";

export default function App({ Component, pageProps }: AppProps) {
  return (
    <AuthProvider>
      <Component {...pageProps} />
    </AuthProvider>
  );
}
________________________________________
📁 frontend/contexts/AuthContext.tsx

import { createContext, useContext, useState, useEffect, ReactNode } from "react";
import axios from "axios";

type AuthContextType = {
  user: any;
  token: string | null;
  setToken: (t: string | null) => void;
  logout: () => void;
};

const AuthContext = createContext<AuthContextType>({
  user: null,
  token: null,
  setToken: () => {},
  logout: () => {},
});

export function AuthProvider({ children }: { children: ReactNode }) {
  const [token, setToken] = useState<string | null>(
    typeof window !== "undefined" ? localStorage.getItem("token") : null
  );
  const [user, setUser] = useState(null);

  useEffect(() => {
    if (!token) return;
    axios.get("/api/auth/user", {
      headers: { Authorization: `Bearer ${token}` },
    }).then(res => setUser(res.data.user))
      .catch(() => setToken(null));
  }, [token]);

  const logout = () => {
    setToken(null);
    setUser(null);
    localStorage.removeItem("token");
  };

  return (
    <AuthContext.Provider value={{ user, token, setToken, logout }}>
      {children}
    </AuthContext.Provider>
  );
}

export function useAuth() {
  return useContext(AuthContext);
}
________________________________________
📁 frontend/pages/servers.tsx

import { useEffect } from "react";
import { useAuth } from "@/contexts/AuthContext";
import AddServerForm from "@/components/AddServerForm";
import ServerList from "@/components/ServerList";

export default function Servers() {
  const { user } = useAuth();

  useEffect(() => {
    if (!user) window.location.href = "/login";
  }, [user]);

  return (
    <div className="max-w-4xl mx-auto p-6">
      <h1 className="text-2xl font-bold mb-4">Your Servers</h1>
      <AddServerForm />
      <ServerList />
    </div>
  );
}
________________________________________
✅ 3. GitHub Actions + Docker
📁 .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build & push backend

        run: |
          docker build -t ghcr.io/${{ github.repository }}/api:latest ./backend
          echo ${{ secrets.GITHUB_TOKEN }} | docker login ghcr.io -u ${{ github.actor }} --password-stdin
          docker push ghcr.io/${{ github.repository }}/api:latest

      - name: Deploy via SSH

        run: |
          ssh ${{ secrets.REMOTE_USER }}@${{ secrets.REMOTE_HOST }} << 'EOF'
            cd ~/game-panel
            docker compose pull
            docker compose up -d
          EOF

