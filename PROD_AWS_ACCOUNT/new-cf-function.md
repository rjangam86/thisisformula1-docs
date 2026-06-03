function handler(event) {
    var request = event.request;
    var uri = request.uri;

    // =========================
    // BASIC BOT FILTER
    // =========================

    if (!request.headers['user-agent']) {
        return {
            statusCode: 403,
            statusDescription: 'Forbidden'
        };
    }

    var userAgent = request.headers['user-agent'].value || '';
    var lowerUserAgent = userAgent.toLowerCase();

    // Identify search engine bots that shouldn't get session redirects
    var isSearchBot = 
        lowerUserAgent.includes('googlebot') || 
        lowerUserAgent.includes('google-inspectiontool') || 
        lowerUserAgent.includes('bingbot');

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
    
    // =========================
    // SESSION BOOTSTRAP CHECK
    // =========================

    var hasSession = false;
    if (
        request.cookies &&
        request.cookies.hlhq_session &&
        request.cookies.hlhq_session.value
    ) {
        hasSession = true;
    }
            
    var isBootstrap =
        uri.startsWith(
            '/api/v1/session'
        );
        
    // Skip static assets
    var isStatic =
        uri.includes('.') &&
        !uri.endsWith('.html');

    // ✅ NEW: Skip session checks for robots.txt and sitemaps
    var isRobotsOrSitemap = 
        uri === '/robots.txt' || 
        uri === '/sitemap.xml' || 
        lowerUri.includes('sitemap');
        
    if (
        !hasSession &&
        !isBootstrap &&
        !isStatic &&
        !isRobotsOrSitemap &&   // Don't redirect search files
        !isSearchBot            // Don't redirect Googlebot/Bingbot
    ) {
        return {
            statusCode: 302,
            statusDescription: 'Found',
            headers: {
                location: {
                    value:
                        '/api/v1/session' +
                        '?redirect=' +
                        encodeURIComponent(uri)
                }
            }
        };
    }      
    
    // ✅ NEVER touch API routes
    if (uri.startsWith('/api/')) {
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
