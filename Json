require("dotenv").config();
const express = require("express");
const axios = require("axios");
const cors = require("cors");
const { v4: uuidv4 } = require("uuid");

const app = express();
app.use(express.json());
app.use(cors({ origin: process.env.FRONTEND_URL || "*" }));

const IS_SANDBOX = process.env.NODE_ENV !== "production";

const MTN = {
  subscriptionKey: process.env.MTN_SUBSCRIPTION_KEY,
  userId: process.env.MTN_USER_ID,
  apiKey: process.env.MTN_API_KEY,
  currency: process.env.MTN_CURRENCY || "ZMW",
  targetEnv: IS_SANDBOX ? "sandbox" : "mtnzambia",
  baseUrl: IS_SANDBOX
    ? "https://sandbox.momodeveloper.mtn.com"
    : "https://proxy.momoapi.mtn.com",
  callbackHost: process.env.CALLBACK_HOST || "https://localhost",
};

let tokenCache = { token: null, expiresAt: 0 };
const transactions = {};

async function getMtnToken() {
  const now = Date.now();
  if (tokenCache.token && tokenCache.expiresAt > now + 60000) {
    return tokenCache.token;
  }
  const credentials = Buffer.from(`${MTN.userId}:${MTN.apiKey}`).toString("base64");
  const response = await axios.post(
    `${MTN.baseUrl}/collection/token/`,
    {},
    {
      headers: {
        "Authorization": `Basic ${credentials}`,
        "Ocp-Apim-Subscription-Key": MTN.subscriptionKey,
        "X-Target-Environment": MTN.targetEnv,
      },
    }
  );
  const { access_token, expires_in } = response.data;
  tokenCache = { token: access_token, expiresAt: now + expires_in * 1000 };
  console.log("✅ MTN token obtained");
  return access_token;
}

function formatPhone(phone) {
  return phone.replace(/\s+/g, "").replace(/^\+260/, "").replace(/^0/, "");
}

app.post("/api/payment/initiate", async (req, res) => {
  const { phone, amount, planId, planName, userId } = req.body;
  if (!phone || !amount || !planId) {
    return res.status(400).json({ success: false, error: "Missing required fields" });
  }
  const referenceId = uuidv4();
  try {
    const token = await getMtnToken();
    const currency = IS_SANDBOX ? "EUR" : MTN.currency;
    await axios.post(
      `${MTN.baseUrl}/collection/v1_0/requesttopay`,
      {
        amount: String(amount),
        currency: currency,
        externalId: uuidv4(),
        payer: { partyIdType: "MSISDN", partyId: formatPhone(phone) },
        payerMessage: `ECZ Prep ${planName} subscription`,
        payeeNote: "Thank you for subscribing to ECZ Prep Zambia!",
      },
      {
        headers: {
          "Authorization": `Bearer ${token}`,
          "X-Reference-Id": referenceId,
          "X-Target-Environment": MTN.targetEnv,
          "X-Callback-Url": `${MTN.callbackHost}/api/payment/webhook`,
          "Ocp-Apim-Subscription-Key": MTN.subscriptionKey,
          "Content-Type": "application/json",
        },
      }
    );
    transactions[referenceId] = {
      id: referenceId, phone, amount, planId, planName, userId,
      status: "PENDING", createdAt: new Date().toISOString(),
    };
    console.log(`📱 MoMo push sent: ${referenceId} → ${phone}`);
    return res.json({ success: true, transactionId: referenceId });
  } catch (err) {
    console.error("MTN error:", err?.response?.data || err.message);
    return res.status(500).json({ success: false, error: "Failed to initiate payment" });
  }
});

app.get("/api/payment/status/:transactionId", async (req, res) => {
  const { transactionId } = req.params;
  const local = transactions[transactionId];
  if (!local) return res.status(404).json({ success: false, error: "Transaction not found" });
  if (local.status === "SUCCESSFUL") return res.json({ success: true, status: "SUCCESSFUL", transaction: local });
  if (local.status === "FAILED") return res.json({ success: false, status: "FAILED" });
  try {
    const token = await getMtnToken();
    const response = await axios.get(
      `${MTN.baseUrl}/collection/v1_0/requesttopay/${transactionId}`,
      {
        headers: {
          "Authorization": `Bearer ${token}`,
          "X-Target-Environment": MTN.targetEnv,
          "Ocp-Apim-Subscription-Key": MTN.subscriptionKey,
        },
      }
    );
    const { status } = response.data;
    console.log(`🔍 Status ${transactionId}: ${status}`);
    if (status === "SUCCESSFUL") {
      transactions[transactionId].status = "SUCCESSFUL";
      transactions[transactionId].completedAt = new Date().toISOString();
      return res.json({ success: true, status: "SUCCESSFUL", transaction: transactions[transactionId] });
    } else if (status === "FAILED") {
      transactions[transactionId].status = "FAILED";
      return res.json({ success: false, status: "FAILED" });
    } else {
      return res.json({ success: true, status: "PENDING" });
    }
  } catch (err) {
    console.error("Status error:", err?.response?.data || err.message);
    return res.status(500).json({ success: false, error: "Could not check status" });
  }
});

app.post("/api/payment/webhook", (req, res) => {
  const body = req.body;
  console.log("📩 Webhook:", JSON.stringify(body));
  const refId = body?.externalId || body?.referenceId;
  if (refId && transactions[refId]) {
    transactions[refId].status = body.status;
  }
  res.status(200).json({ received: true });
});

app.get("/api/health", (_, res) => {
  res.json({ status: "ok", environment: IS_SANDBOX ? "sandbox" : "production", timestamp: new Date().toISOString() });
});

app.get("/", (_, res) => res.send("ECZ Prep Payment Server is running 🚀"));

const PORT = process.env.PORT || 3001;
app.listen(PORT, () => {
  console.log(`🚀 ECZ Prep MTN Server on port ${PORT}`);
  console.log(`🌍 ${IS_SANDBOX ? "SANDBOX" : "PRODUCTION"} mode`);
});
