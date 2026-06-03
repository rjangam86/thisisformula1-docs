function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // =========================
    // DEV ACCESS PROTECTION
    // =========================
    
    // =========================
    // BASIC BOT FILTER
    // =========================

    if (!request.headers['user-agent']) {
        return {
            statusCode: 403
        };
    }

    var blocked = [
        '/wp-admin',
        '/xmlrpc.php',
        '/phpmyadmin',
        '/.env'
    ];

    var lowerUri = uri.toLowerCase();

    for (var i = 0; i < blocked.length; i++) {

        if (lowerUri.includes(blocked[i])) {

            return {
                statusCode: 403,
                statusDescription: 'Forbidden'
            };
        }
    }
    
    // ✅ NEVER touch API routes
    if (uri.startsWith('/api/')) {
        return request;
    }
    
    // ✅ Skip static files (VERY IMPORTANT)
    if (uri.includes('.')) {
        return request;
    }    

    // ✅ Rewrite only frontend article pages
    if (uri.startsWith('/articles/') && !uri.includes('.')) {

        var slug = uri.replace('/articles/', '');

        request.uri = '/single-post.html';

        request.querystring = {
            slug_id: { value: slug }
        };
        return request;
    }
    
// =========================
    // DRIVERS
    // =========================
    if (uri.startsWith('/drivers/') && !uri.includes('.')) {
        var driver_id = uri.replace('/drivers/', '');

        request.uri = '/single-driver.html';
        request.querystring = {
            driverId: { value: driver_id }
        };
        return request;
    }

    // =========================
    // TEAMS
    // =========================
    if (uri.startsWith('/teams/') && !uri.includes('.')) {
        var team_id = uri.replace('/teams/', '');

        request.uri = '/single-team.html';
        request.querystring = {
            teamId: { value: team_id }
        };
        return request;
    }
    
    // =========================
    // POLICIES
    // =========================

    var policyPages = [
        'privacy-policy',
        'cookie-policy',
        'terms-of-use',
        'editorial-policy',
        'corrections-policy',
        'copyright-policy',
        'disclaimer'
    ];

    var cleanPath = uri.replace(/^\/|\/$/g, '');

    if (policyPages.includes(cleanPath)) {
        request.uri = '/policies/' + cleanPath + '.html';
        return request;
    }
    
    // =========================
    // OTHER STATIC PAGES
    // =========================

    var staticPages = {
        'about-us': '/about-us.html',
        'contact-us': '/contact-us.html'
    };

    var path = uri.replace(/^\/|\/$/g, '');

    if (staticPages[path]) {
        request.uri = staticPages[path];
        return request;
    }    
    
    return request;
}



LAMBDA BOOTSTRAP:

import crypto from "crypto";
import { corsHeaders } from "./cors.mjs";

const   SECRET = process.env.HQ_HMAC_SECRET;

function base64UrlEncode(input) {
    return Buffer.from(input)
        .toString("base64")
        .replace(/\+/g, "-")
        .replace(/\//g, "_")
        .replace(/=+$/, "");
}

export const handler = async (event) => {

    const now = Math.floor(Date.now() / 1000);

    const origin =
        event.headers?.origin ||
        event.headers?.Origin;

    const domain =
        origin?.includes("dev.")
            ? "dev.hotlaphq.com"
            : "hotlaphq.com";

    const payload = {
        iat: now,
        exp: now + 300,
        dom: domain,
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

    const sameSite =
        origin?.includes("localhost")? "None; Partitioned": "Lax";    

    return {
        statusCode: 200,
        headers: {
            ...corsHeaders(event),

            "Set-Cookie":
                `hlhq_dev_session=${token}; Path=/; Secure; HttpOnly; SameSite=${sameSite}; Max-Age=300`,

            "Cache-Control":
                "no-store"
        },
        body: JSON.stringify({
            success: true
        })
    };
};