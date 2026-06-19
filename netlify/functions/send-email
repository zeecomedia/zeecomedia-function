const BREVO_API_URL = "https://api.brevo.com/v3/smtp/email";
const BREVO_CONTACTS_URL = "https://api.brevo.com/v3/contacts";

function corsHeaders() {
  const origin = process.env.ALLOWED_ORIGIN || "*";
  return {
    "Access-Control-Allow-Origin": origin,
    "Access-Control-Allow-Headers": "Content-Type",
    "Access-Control-Allow-Methods": "POST, OPTIONS",
  };
}

function json(statusCode, body) {
  return {
    statusCode,
    headers: { "Content-Type": "application/json", ...corsHeaders() },
    body: JSON.stringify(body),
  };
}

function isValidEmail(email) {
  return typeof email === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

// Minimal HTML-escaping so form input can't break the email markup
function escapeHtml(str = "") {
  return String(str)
    .replace(/&/g, "&amp;")
    .replace(/</g, "&lt;")
    .replace(/>/g, "&gt;")
    .replace(/"/g, "&quot;")
    .replace(/'/g, "&#39;");
}

exports.handler = async function (event) {
  // CORS preflight
  if (event.httpMethod === "OPTIONS") {
    return { statusCode: 204, headers: corsHeaders(), body: "" };
  }

  if (event.httpMethod !== "POST") {
    return json(405, { error: "Method not allowed" });
  }

  const apiKey = process.env.BREVO_API_KEY;
  if (!apiKey) {
    console.error("Missing BREVO_API_KEY environment variable");
    return json(500, { error: "Server is not configured correctly." });
  }

  let payload;
  try {
    payload = JSON.parse(event.body || "{}");
  } catch {
    return json(400, { error: "Invalid JSON body." });
  }

  const action = payload.action;

  if (action === "subscribe") {
    return handleSubscribe(payload, apiKey);
  }
  if (action === "contact") {
    return handleContact(payload, apiKey);
  }
  return json(400, { error: "Unknown or missing 'action'. Expected 'subscribe' or 'contact'." });
};

async function handleSubscribe(payload, apiKey) {
  const email = (payload.email || "").trim();

  if (!isValidEmail(email)) {
    return json(400, { error: "Please provide a valid email address." });
  }

  const fromEmail = process.env.FROM_EMAIL;
  const fromName = process.env.FROM_NAME || "Zeecomedia";

  if (!fromEmail) {
    console.error("Missing FROM_EMAIL environment variable");
    return json(500, { error: "Server is not configured correctly." });
  }

  // 1) Add/update the contact in Brevo (so they land in your list/automation)
  const listId = process.env.BREVO_LIST_ID ? Number(process.env.BREVO_LIST_ID) : undefined;
  try {
    const contactRes = await fetch(BREVO_CONTACTS_URL, {
      method: "POST",
      headers: {
        "api-key": apiKey,
        "Content-Type": "application/json",
        Accept: "application/json",
      },
      body: JSON.stringify({
        email,
        listIds: listId ? [listId] : undefined,
        updateEnabled: true, // don't fail if the contact already exists
      }),
    });

    // Brevo returns 204 for update, 201 for create — both fine.
    // A 400 with code "duplicate_parameter" can still happen on some plans; treat as success.
    if (!contactRes.ok && contactRes.status !== 400) {
      const errBody = await safeJson(contactRes);
      console.error("Brevo contact creation failed:", errBody);
      return json(502, { error: "Could not save your email. Please try again." });
    }
  } catch (err) {
    console.error("Brevo contact request error:", err);
    return json(502, { error: "Could not reach the email service. Please try again." });
  }

  // 2) Send the eBook email itself
  try {
    const emailRes = await fetch(BREVO_API_URL, {
      method: "POST",
      headers: {
        "api-key": apiKey,
        "Content-Type": "application/json",
        Accept: "application/json",
      },
      body: JSON.stringify({
        sender: { email: fromEmail, name: fromName },
        to: [{ email }],
        subject: "Your Free AI Prompts eBook",
        htmlContent: `
          <p>Hi there,</p>
          <p>Thanks for your interest in Zeecomedia! Here's your free <strong>AI Prompts For All Professionals</strong> eBook:</p>
          <p><a href="${process.env.EBOOK_DOWNLOAD_URL || "#"}">Download your eBook</a></p>
          <p>If the link doesn't work, reply to this email and we'll send it directly.</p>
          <p>— The Zeecomedia Team</p>
        `,
      }),
    });

    if (!emailRes.ok) {
      const errBody = await safeJson(emailRes);
      console.error("Brevo send email failed:", errBody);
      return json(502, { error: "We saved your email but the eBook could not be sent. We'll follow up manually." });
    }
  } catch (err) {
    console.error("Brevo send email request error:", err);
    return json(502, { error: "We saved your email but the eBook could not be sent right now." });
  }

  return json(200, { message: "Success! Check your inbox for the eBook." });
}

async function handleContact(payload, apiKey) {
  const fname = (payload.fname || "").trim();
  const lname = (payload.lname || "").trim();
  const email = (payload.email || "").trim();
  const subject = (payload.subject || "").trim();
  const message = (payload.message || "").trim();

  if (!fname || !lname) {
    return json(400, { error: "First and last name are required." });
  }
  if (!isValidEmail(email)) {
    return json(400, { error: "A valid email address is required." });
  }
  if (!subject) {
    return json(400, { error: "Please provide a subject." });
  }
  if (!message || message.length < 20) {
    return json(400, { error: "Please describe your project in at least 20 characters." });
  }

  const fromEmail = process.env.FROM_EMAIL;
  const fromName = process.env.FROM_NAME || "Zeecomedia Website";
  const toEmail = process.env.TO_EMAIL;

  if (!fromEmail || !toEmail) {
    console.error("Missing FROM_EMAIL or TO_EMAIL environment variable");
    return json(500, { error: "Server is not configured correctly." });
  }

  try {
    const emailRes = await fetch(BREVO_API_URL, {
      method: "POST",
      headers: {
        "api-key": apiKey,
        "Content-Type": "application/json",
        Accept: "application/json",
      },
      body: JSON.stringify({
        sender: { email: fromEmail, name: fromName },
        to: [{ email: toEmail }],
        replyTo: { email, name: `${fname} ${lname}` },
        subject: `New contact form submission: ${subject}`,
        htmlContent: `
          <h2>New website inquiry</h2>
          <p><strong>Name:</strong> ${escapeHtml(fname)} ${escapeHtml(lname)}</p>
          <p><strong>Email:</strong> ${escapeHtml(email)}</p>
          <p><strong>Subject:</strong> ${escapeHtml(subject)}</p>
          <p><strong>Message:</strong></p>
          <p>${escapeHtml(message).replace(/\n/g, "<br>")}</p>
        `,
      }),
    });

    if (!emailRes.ok) {
      const errBody = await safeJson(emailRes);
      console.error("Brevo send email failed:", errBody);
      return json(502, { error: "Could not send your message right now. Please try again or email us directly." });
    }
  } catch (err) {
    console.error("Brevo send email request error:", err);
    return json(502, { error: "Could not reach the email service. Please try again or email us directly." });
  }

  return json(200, { message: "Message sent! We'll get back to you within 24 hours." });
}

async function safeJson(res) {
  try {
    return await res.json();
  } catch {
    return { raw: await res.text().catch(() => "") };
  }
}
