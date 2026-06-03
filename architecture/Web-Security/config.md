

X-HQ-SECRET

Q7mL2xV9pR4kT8wN1cY6hF3zD5sB0jU7eA2qM9vX4nK8tP1rC6yH5gL0wZ3fS8d


HQ_HMAC_SECRET - DEV
V8qLm2Xf9Kp4Rw7Nc1Zh6Ty3Ua0Bj5Ds8Qv2He9Mx4Pt7Yk1Cf6Jn3Lw0Sb5Gx8R


HQ_HMAC_SECRET - PROD
M4xQv8Ty1Nc7Rw3Zh6Lp0Ks5Df9Bj2He8Ua4Pt1Yk7Cf3Jn6Lm5Sb0Gx9Vw2Rq8T

SESSION BOOTSTRAP LAMBDA CODE:

import crypto from "crypto";

const SECRET = process.env.HQ_HMAC_SECRET;

function base64UrlEncode(input) {
    return Buffer.from(input)
        .toString("base64")
        .replace(/\+/g, "-")
        .replace(/\//g, "_")
        .replace(/=+$/, "");
}

export const handler = async () => {

    const now = Math.floor(Date.now() / 1000);

    const payload = {
        iat: now,
        exp: now + 300,
        dom: "dev.hotlaphq.com",
        aud: "hotlaphq-api"
    };

    const encoded = base64UrlEncode(
        JSON.stringify(payload)
    );

    const signature = crypto
        .createHmac("sha256", SECRET)
        .update(encoded)
        .digest("hex");

    const token = `${encoded}.${signature}`;

    return {
        statusCode: 200,
        headers: {
            "Set-Cookie":
                `hq_session=${token}; Path=/; Secure; SameSite=Lax; Max-Age=300`
        },
        body: JSON.stringify({
            success: true
        })
    };
};

HT_SECRET DEV

Q7mL2xV9pR4kT8wN1cY6hF3zD5sB0jU7eA2qM9vX4nK8tP1rC6yH5gL0wZ3fS8d

HT_SECRET PROD

7f9c2d8e4a1b6c3f5d0e9a7b2c4f8e1d6a3c9b5f0e7d2a8c4b1f6e9d3c7a2f5