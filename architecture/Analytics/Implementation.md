


gtag('consent', 'update', {

  analytics_storage: consent.analytics ? 'granted' : 'denied',

  ad_storage: consent.ads ? 'granted' : 'denied',

  ad_user_data: consent.ads ? 'granted' : 'denied',

  ad_personalization: consent.ads ? 'granted' : 'denied'

});

  const consent = { analytics: true, ads: true };

      gtag('consent', 'default', {
        analytics_storage: 'denied',
        ad_storage: 'denied',
        ad_user_data: 'denied',
        ad_personalization: 'denied'
      });

* hlhq_session
* hlhq_security
* hlhq_hmac
* hlhq_auth
* hlhq_preferences
