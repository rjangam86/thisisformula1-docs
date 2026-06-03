    // =========================
    // OG PAGES
    // =========================
    
    if (uri.startsWith('/og/')) {
    
        var ua =
            (
                request.headers['user-agent'] &&
                request.headers['user-agent'].value
                    ? request.headers['user-agent'].value
                    : ''
            ).toLowerCase();
    
        console.log('UA:',ua);
        
        var isBot =
            ua.includes('facebookexternalhit') ||
            ua.includes('facebot') ||
            ua.includes('twitterbot') ||
            ua.includes('linkedinbot');
    
        var og_slug_id =
            uri.replace('/og/', '');    
            
        var parts =
            uri.split('/');
        
        var fileName =
            parts[parts.length - 1];
        
        var slug_id =
            fileName.replace('.html', '');
    
        /*
          Bots:
          Serve static OG HTML
          from hotlaphq-prod-articles
        */
    
        //if (isBot) {
        if (uri.startsWith('/og/')) {        
            request.uri =
                '/og-api/render';
            request.querystring = {
                    slug_id: {
                        value: og_slug_id
                    }
                };
                
            request.headers['x-debug-bot'] = {
                value: isBot ? 'true' : 'false'
            };
            request.headers['x-debug-ua'] = {
                value: ua
            };                
            return request;
        }
        
        /*
          Humans:
          Redirect to real article URL
        */
    
        return {
            statusCode: 302,
            statusDescription: 'Found',
            headers: {
                location: {
                    value: '/articles/' + slug_id
                }
            }
        };
    }
