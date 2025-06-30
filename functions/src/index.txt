import { onRequest } from "firebase-functions/v2/https";
import * as logger from "firebase-functions/logger";
import express, { Request, Response } from "express";
import fetch from "cross-fetch";
import { defineSecret } from "firebase-functions/params";

const AUTH_TOKEN = defineSecret("AUTH_TOKEN");
const OCR_API_KEY = defineSecret("OCR_API_KEY");

const app = express();
app.use(express.json());

// Proxy route
app.all("/proxy", async (req: Request, res: Response) => {
  try {
    const url = new URL("https://script.google.com/macros/s/AKfycbyTTvZAHUfYBQ1rwFdl389oiLa6E8K0LKkeECwZsz1ljF76mz60W99PpqjsRiRSjTVygw/exec");
    url.searchParams.append("token", AUTH_TOKEN.value());
    
    Object.entries(req.query).forEach(([key, value]) =>
      url.searchParams.append(key, String(value))
    );

    const scriptRes = await fetch(url.toString(), {
      method: req.method,
      headers: { "Content-Type": "application/json" },
      body: req.method !== "GET" ? JSON.stringify(req.body) : undefined,
    });

    const data = await scriptRes.json();
    res.status(scriptRes.status).json(data);
  } catch (err) {
    logger.error("Proxy error", err);
    res.status(500).json({ error: "Internal proxy error" });
  }
});

// OCR route
app.post("/ocr-proxy", async (req: Request, res: Response) => {
  try {
    const OCR_API_URL = "https://api.ocr.space/parse/image";
    const response = await fetch(OCR_API_URL, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded" },
      body: new URLSearchParams({
        apikey: OCR_API_KEY.value(),
        base64Image: `data:image/jpeg;base64,${req.body.imageBase64}`,
        isOverlayRequired: "false",
      }),
    });

    const data = await response.json();
    res.status(response.status).json(data);
  } catch (error) {
    logger.error("OCR Proxy error", error);
    res.status(500).json({ error: "OCR proxy failed" });
  }
});

// Export with runtime config
export const api = onRequest(
  {
    secrets: [AUTH_TOKEN, OCR_API_KEY],
    region: "us-central1",
    memory: "1GiB",
    cpu: 1,
  },
  app
);

