https://dev.hotlaphq.com/unlock/7c8f9d2a4b6e1f93c7d84e2b51fa93d7c84b1f6e2d9a73f4


    // =========================
    // DEV ACCESS PROTECTION
    // =========================

    var SECRET =
        "7c8f9d2a4b6e1f93c7d84e2b51fa93d7c84b1f6e2d9a73f4";
    
    var hasAccess = false;
    
    // Check auth cookie
    if (
        request.cookies &&
        request.cookies.dev_access &&
        request.cookies.dev_access.value === SECRET
    ) {
        hasAccess = true;
    }
    
    // Unlock URL
    if (
        uri === "/unlock/" + SECRET
    ) {
    
        return {
            statusCode: 302,
            statusDescription: "Found",
    
            headers: {
                location: {
                    value: "/"
                }
            },
    
            cookies: {
                dev_access: {
                    value: SECRET,
                    attributes:
                        "Path=/; Secure; HttpOnly; SameSite=Lax; Max-Age=2592000"
                }
            }
        };
    }
    
    // Block all unauthenticated requests
    if (!hasAccess) {
    
        return {
            statusCode: 403,
            statusDescription: "Forbidden",
    
            headers: {
                "content-type": {
                    value: "text/plain"
                },
    
                "cache-control": {
                    value: "no-store"
                },
    
                "x-robots-tag": {
                    value: "noindex, nofollow"
                }
            },
    
            body: "403 Forbidden"
        };
    }


BACKUP:

window.SessionManager = (function () {

  const SESSION_READY_KEY =
    "hlhq_session_ready";

  const SESSION_TS_KEY =
    "hlhq_session_ts";

  const SESSION_TTL_MS =
    300 * 1000;

  const SESSION_REFRESH_BUFFER_MS =
    20 * 1000;

  function markSessionReady() {

    localStorage.setItem(
      SESSION_READY_KEY,
      "1"
    );

    localStorage.setItem(
      SESSION_TS_KEY,
      String(Date.now())
    );
  }

  function clearSession() {

    localStorage.removeItem(
      SESSION_READY_KEY
    );

    localStorage.removeItem(
      SESSION_TS_KEY
    );
  }

  function isSessionValid() {

    const ready =
      localStorage.getItem(
        SESSION_READY_KEY
      );

    const ts =
      Number(
        localStorage.getItem(
          SESSION_TS_KEY
        )
      );

    if (!ready || !ts) {
      return false;
    }

    const age =
      Date.now() - ts;

    return age < (
      SESSION_TTL_MS -
      SESSION_REFRESH_BUFFER_MS
    );
  }

  async function createSession() {

    if (isSessionValid()) {
      return true;
    }

    const API_BASE =
      window.APP_CONFIG.API_BASE;

    try {
        const res = await fetch(
        API_BASE + "/session",
        {
          credentials: "include"
        }
      );

      if (!res.ok) {
        clearSession();
        throw new Error(
          "Session bootstrap failed"
        );
      }
      markSessionReady();
      return true;
    } catch (err) {
      clearSession();
      throw err;
    }
  }

  return {
    createSession,
    clearSession,
    markSessionReady,
    isSessionValid
  };

})();

window.SiteStore = (function () {

  async function apiFetch(url, options = {}) {

    const headers =
      new Headers(options.headers || {});

    headers.set(
      "X-HQ-TS",
      String(Date.now())
    );

    return fetch(url, {
      ...options,
      headers,
      credentials: "include"
    });
  }

  let data = null;
  let promise = null;
  let listeners = []; // ðŸ”¥ ADD THIS

  function load() {
    if (data) return Promise.resolve(data);
    if (promise) return promise;

    const API_BASE = window.APP_CONFIG.API_BASE;

    promise = (async () => {

      // SESSION BOOTSTRAP
      await SessionManager.createSession();

      // MAIN API CALL
      let res  = await apiFetch(
        `${API_BASE}/layout-data`
      );
      if (res.status === 401 || res.status === 403) {
        SessionManager.clearSession();
        await SessionManager.createSession();
        res = await apiFetch(
          `${API_BASE}/layout-data`
        );
      }
      const json = await res.json();

      data = json;
      listeners.forEach(fn => fn(data));
      listeners = [];
      return data;
      
    })()
      .catch(err => {
        console.error(
          "SiteStore load failed",
          err
        );
        handleApiError({
          status: 0,
          message: "Network error"
        });
        return null;
      });
    return promise;
  }

  function get() {
    return data;
  }

  // ðŸ”¥ ADD THIS FUNCTION
  function onReady(cb) {
    if (data) {
      cb(data); // already loaded â†’ run immediately
    } else {
      listeners.push(cb); // wait until load completes
    }
  }

  return {
    load,
    get,
    onReady // ðŸ”¥ EXPORT THIS
  };

})();