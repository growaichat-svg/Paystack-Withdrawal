# Paystack-Withdrawal
{
  "name": "paystack-withdrawals",
  "version": "1.0.0",
  "main": "server.js",
  "scripts": {
    "start": "node server.js"
  },
  "engines": {
    "node": ">=18"
  },
  "dependencies": {
    "dotenv": "^16.4.5",
    "express": "^4.19.2"
  }
}
require("dotenv").config();
const express = require("express");
const crypto = require("crypto");

const app = express();
const KEY = process.env.PAYSTACK_SECRET_KEY;
const API_SECRET = process.env.API_SECRET;
const SUPABASE_URL = process.env.SUPABASE_URL;
const SUPABASE_KEY = process.env.SUPABASE_SERVICE_ROLE_KEY;
const PORT = process.env.PORT || 3000;

for (const [name, v] of Object.entries({
  PAYSTACK_SECRET_KEY: KEY,
  API_SECRET,
  SUPABASE_URL,
  SUPABASE_SERVICE_ROLE_KEY: SUPABASE_KEY,
})) {
  if (!v) {
    console.error(`Missing ${name}`);
    process.exit(1);
  }
}

const paystack = (path, method = "GET", body) =>
  fetch(`https://api.paystack.co${path}`, {
    method,
    headers: {
      Authorization: `Bearer ${KEY}`,
      "Content-Type": "application/json",
    },
    body: body ? JSON.stringify(body) : undefined,
  }).then((r) => r.json());

const sbHeaders = {
  apikey: SUPABASE_KEY,
  Authorization: `Bearer ${SUPABASE_KEY}`,
  "Content-Type": "application/json",
};

const rpc = async (fn, args) => {
  const r = await fetch(`${SUPABASE_URL}/rest/v1/rpc/${fn}`, {
    method: "POST",
    headers: sbHeaders,
    body: JSON.stringify(args),
  });
  if (!r.ok) throw new Error(`Supabase ${fn} failed: ${await r.text()}`);
  return r.json();
};

const sbSelect = async (path) => {
  const r = await fetch(`${SUPABASE_URL}/rest/v1/${path}`, {
    headers: sbHeaders,
  });
  if (!r.ok) throw new Error(`Supabase select failed: ${await r.text()}`);
  return r.json();
};

app.post(
  "/paystack-webhook",
  express.raw({ type: "application/json" }),
  async (req, res) => {
    const hash = crypto
      .createHmac("sha512", KEY)
      .update(req.body)
      .digest("hex");
    if (hash !== req.headers["x-paystack-signature"]) {
      return res.sendStatus(401);
    }

    try {
      const { event, data } = JSON.parse(req.body.toString());

      if (event === "transfer.success") {
        await rpc("complete_withdrawal", {
          p_ref: data.reference,
          p_transfer_code: data.transfer_code,
        });
      }

      if (event === "transfer.failed" || event === "transfer.reversed") {
        await rpc("refund_withdrawal", { p_ref: data.reference });
      }

      res.sendStatus(200);
    } catch (err) {
      console.error(err);
      res.sendStatus(500);
    }
  }
);

app.use(express.json());

const requireAuth = (req, res, next) => {
  if (req.headers["x-api-secret"] !== API_SECRET) {
    return res.status(401).json({ error: "Unauthorized" });
  }
  next();
};

app.get("/", (req, res) => res.send("Withdrawal server running"));

app.get("/banks", requireAuth, async (req, res) => {
  res.json(await paystack("/bank?country=nigeria&perPage=100"));
});

app.get("/resolve", requireAuth, async (req, res) => {
  const { account_number, bank_code } = req.query;
  res.json(
    await paystack(
      `/bank/resolve?account_number=${account_number}&bank_code=${bank_code}`
    )
  );
});

app.get("/balance/:user_id", requireAuth, async (req, res) => {
  try {
    const rows = await sbSelect(
      `users?id=eq.${encodeURIComponent(req.params.user_id)}&select=balance`
    );
    const kobo = rows[0] ? Number(rows[0].balance) : 0;
    res.json({ user_id: req.params.user_id, balance: kobo / 100 });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

app.post("/credit", requireAuth, async (req, res) => {
  try {
    const { user_id, amount } = req.body;
    if (!user_id || !(amount > 0)) {
      return res.status(400).json({ error: "Invalid request" });
    }
    const kobo = await rpc("credit_user", {
      p_user: user_id,
      p_amount: Math.round(amount * 100),
    });
    res.json({ user_id, balance: Number(kobo) / 100 });
  } catch (err) {
    console.error(err);
    res.status(500).json({ error: "Server error" });
  }
});

app.post("/withdraw", requireAuth, async (req, res) => {
  const { user_id, name, account_number, bank_code, amount } = req.body;

  if (!user_id || !name || !account_number || !bank_code || !(amount >= 100)) {
    return res.status(400).json({ error: "Invalid request" });
  }

  const reference = crypto.randomUUID();
  const kobo = Math.round(amount * 100);
  let deducted = false;

  try {
    deducted = await rpc("request_withdrawal", {
      p_user: user_id,
      p_amount: kobo,
      p_ref: reference,
    });
    if (!deducted) {
      return res.status(400).json({ error: "Insufficient balance" });
    }

    const recipient = await paystack("/transferrecipient", "POST", {
      type: "nuban",
      name,
      account_number,
      bank_code,
      currency: "NGN",
    });
    if (!recipient.status) {
      await rpc("refund_withdrawal", { p_ref: reference });
      return res.status(400).json(recipient);
    }

    const transfer = await paystack("/transfer", "POST", {
      source: "balance",
      amount: kobo,
      recipient: recipient.data.recipient_code,
      reference,
      reason: "Withdrawal",
    });
    if (!transfer.status) {
      await rpc("refund_withdrawal", { p_ref: reference });
      return res.status(400).json(transfer);
    }

    res.json({
      message: transfer.message,
      status: transfer.data.status,
      transfer_code: transfer.data.transfer_code,
      reference,
    });
  } catch (err) {
    console.error(err);
    res.status(500).json({
      error: "Server error. Check status later.",
      reference: deducted ? reference : undefined,
    });
  }
});

app.listen(PORT, () => console.log(`Server running on port ${PORT}`));
create table if not exists users (
  id text primary key,
  balance bigint not null default 0 check (balance >= 0)
);

create table if not exists withdrawals (
  reference text primary key,
  user_id text not null references users(id),
  amount bigint not null check (amount > 0),
  status text not null default 'pending',
  transfer_code text,
  created_at timestamptz not null default now()
);

alter table users enable row level security;
alter table withdrawals enable row level security;

create or replace function credit_user(p_user text, p_amount bigint)
returns bigint language plpgsql as $$
declare new_balance bigint;
begin
  insert into users (id, balance) values (p_user, p_amount)
  on conflict (id) do update set balance = users.balance + p_amount
  returning balance into new_balance;
  return new_balance;
end $$;

create or replace function request_withdrawal(p_user text, p_amount bigint, p_ref text)
returns boolean language plpgsql as $$
begin
  update users set balance = balance - p_amount
  where id = p_user and balance >= p_amount;
  if not found then return false; end if;
  insert into withdrawals (reference, user_id, amount) values (p_ref, p_user, p_amount);
  return true;
end $$;

create or replace function refund_withdrawal(p_ref text)
returns boolean language plpgsql as $$
declare w withdrawals%rowtype;
begin
  update withdrawals set status = 'failed'
  where reference = p_ref and status = 'pending'
  returning * into w;
  if not found then return false; end if;
  update users set balance = balance + w.amount where id = w.user_id;
  return true;
end $$;

create or replace function complete_withdrawal(p_ref text, p_transfer_code text)
returns boolean language plpgsql as $$
begin
  update withdrawals set status = 'success', transfer_code = p_transfer_code
  where reference = p_ref and status = 'pending';
  return found;
end $$;
